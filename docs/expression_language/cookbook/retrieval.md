# توليد معزز بالاسترجاع (RAG)

دعنا نلقي نظرة على إضافة خطوة استرجاع إلى موجه ونموذج لغوي كبير (LLM)، والتي تشكل معًا "سلسلة توليد معزز بالاسترجاع".

لهذا المثال، سنستخدم مخزن المتجهات Chroma. أولاً، سنضيف بعض المستندات إلى مخزن المتجهات:

```dart
final openaiApiKey = Platform.environment["OPENAI_API_KEY"]!;
final embeddings = OpenAIEmbeddings(apiKey: openaiApiKey);

final vectorStore = Chroma(embeddings: embeddings);
await vectorStore.addDocuments(
  documents: const [
    Document(pageContent: 'طرق الدفع: iDEAL، باي بال، وبطاقة الائتمان'),
    Document(pageContent: 'شحن مجاني: للطلبات التي تزيد عن 30 يورو'),
  ],
);
```

الآن يمكننا استخدام مخزن المتجهات كأداة استرجاع في سلسلة:

```dart
final retriever = vectorStore.asRetriever();
final model = ChatOpenAI(apiKey: openaiApiKey);

final promptTemplate = ChatPromptTemplate.fromTemplate('''
أجب عن السؤال بناءً على السياق التالي فقط:
{context}

السؤال: {question}''');

final chain = Runnable.fromMap<String>({
  'context': retriever | Runnable.mapInput((docs) => docs.join('\n')),
  'question': Runnable.passthrough(),
}) | promptTemplate | model | StringOutputParser();

final res1 = await chain.invoke('ما هي طرق الدفع التي تقبلونها؟');
print(res1);
// طرق الدفع المقبولة هي iDEAL، باي بال، وبطاقة الائتمان.

await chain.stream('كيف يمكنني الحصول على شحن مجاني؟').forEach(stdout.write);
// للحصول على شحن مجاني، يجب أن يكون طلبك أكثر من 30 يورو.
```

تخيل أننا نريد الآن الإجابة على السؤال بلغة مختلفة. سنحتاج إلى تمرير معلمتين عند استدعاء السلسلة. يمكننا استخدام:

```dart
final promptTemplate = ChatPromptTemplate.fromTemplate('''
أجب عن السؤال بناءً على السياق التالي فقط:
{context}

السؤال: {question}

أجب باللغة التالية: {language}''');

final chain = Runnable.fromMap({
      'context': Runnable.getItemFromMap<String>('question') |
          (retriever | Runnable.mapInput((docs) => docs.join('\n'))),
      'question': Runnable.getItemFromMap('question'),
      'language': Runnable.getItemFromMap('language'),
    }) |
    promptTemplate |
    model |
    StringOutputParser();

final res1 = await chain.invoke({
  'question': 'ما هي طرق الدفع التي تقبلونها؟',
  'language': 'es_ES',
});
print(res1);
// Aceptamos los siguientes métodos de pago: iDEAL, PayPal y tarjeta de crédito.

await chain.stream({
  'question': 'كيف يمكنني الحصول على شحن مجاني؟',
  'language': 'nl_NL',
}).forEach(stdout.write);
// Om gratis verzending te krijgen, moet je bestellingen plaatsen van meer dan 30€.
```

*ملاحظة: ربما لاحظت أننا أضفنا أقواسًا حول أداة الاسترجاع. هذا للتحايل على قيود استنتاج النوع في Dart عند استخدام عامل التشغيل `|`. لن تحتاج إليها إذا استخدمت `.pipe` بدلاً من ذلك.*

## سلسلة الاسترجاع الحوارية

نظرًا لأنه يمكننا إنشاء `Runnable`s من الدوال، يمكننا إضافة سجل المحادثة عبر دالة تنسيق. يتيح لنا هذا إعادة إنشاء `ConversationalRetrievalQAChain` الشهيرة لـ "الدردشة مع البيانات":

```dart
final retriever = vectorStore.asRetriever();
final model = ChatOpenAI(apiKey: openaiApiKey);

final condenseQuestionPrompt = ChatPromptTemplate.fromTemplate('''
بناءً على المحادثة التالية وسؤال متابعة، أعد صياغة سؤال المتابعة ليكون سؤالًا مستقلاً، بلغته الأصلية.

سجل المحادثة:
{chat_history}
مدخل المتابعة: {question}
سؤال مستقل:''');

final answerPrompt = ChatPromptTemplate.fromTemplate('''
أجب عن السؤال بناءً على السياق التالي فقط:
{context}

السؤال: {question}''');

String combineDocuments(
  final List<Document> documents, {
  final String separator = '\n\n',
}) {
  return documents.map((final d) => d.pageContent).join(separator);
}

String formatChatHistory(final List<(String, String)> chatHistory) {
  final formattedDialogueTurns = chatHistory.map((final dialogueTurn) {
    final (human, ai) = dialogueTurn;
    return 'Human: $human\nAssistant: $ai';
  });
  return formattedDialogueTurns.join('\n');
}

final inputs = Runnable.fromMap({
  'standalone_question': Runnable.fromMap({
        'question': Runnable.getItemFromMap('question'),
        'chat_history': 
            Runnable.getItemFromMap<List<(String, String)>>('chat_history') |
                Runnable.mapInput(formatChatHistory),
      }) |
      condenseQuestionPrompt |
      model |
      StringOutputParser(reduceOutputStream: true),
});

final context = Runnable.fromMap({
  'context': Runnable.getItemFromMap<String>('standalone_question') |
      retriever |
      Runnable.mapInput<List<Document>, String>(combineDocuments),
  'question': Runnable.getItemFromMap('standalone_question'),
});

final conversationalQaChain =
    inputs | context | answerPrompt | model | StringOutputParser();

final res1 = await conversationalQaChain.invoke({
  'question': 'ما هي طرق الدفع التي تقبلونها؟',
  'chat_history': <(String, String)>[],
});
print(res1);
// طرق الدفع المقبولة حاليًا هي iDEAL، باي بال، وبطاقة الائتمان.

await conversationalQaChain.stream({
  'question': 'هل أحصل على شحن مجاني؟',
  'chat_history': [('كم أنفقت؟', 'أنفقت 100 يورو')],
}).forEach(stdout.write);
// نعم، الشحن مجاني للطلبات التي تزيد عن 30 يورو.
```

### مع الذاكرة وإرجاع المستندات المصدرية

في هذا المثال، سنضيف ذاكرة إلى السلسلة ونعيد المستندات المصدرية من أداة الاسترجاع.

```dart
final retriever = vectorStore.asRetriever(
  defaultOptions: const VectorStoreRetrieverOptions(
    searchType: VectorStoreSimilaritySearch(k: 1),
  ),
);
final model = ChatOpenAI(apiKey: openaiApiKey);
const stringOutputParser = StringOutputParser<ChatResult>();
final memory = ConversationBufferMemory(
  inputKey: 'question',
  outputKey: 'answer',
  memoryKey: 'history',
  returnMessages: true,
);

final condenseQuestionPrompt = ChatPromptTemplate.fromTemplate('''
بناءً على المحادثة التالية وسؤال متابعة، أعد صياغة سؤال المتابعة ليكون سؤالًا مستقلاً يتضمن جميع التفاصيل من المحادثة بلغته الأصلية

سجل المحادثة:
{chat_history}
مدخل المتابعة: {question}
سؤال مستقل:''');

final answerPrompt = ChatPromptTemplate.fromTemplate('''
أجب عن السؤال بناءً على السياق التالي فقط:
{context}

السؤال: {question}''');

String combineDocuments(
  final List<Document> documents, {
  final String separator = '\n\n',
}) =>
    documents.map((final d) => d.pageContent).join(separator);

String formatChatHistory(final List<ChatMessage> chatHistory) {
  final formattedDialogueTurns = chatHistory
      .map(
        (final msg) => switch (msg) {
          HumanChatMessage _ => 'Human: ${msg.content}',
          AIChatMessage _ => 'AI: ${msg.content}',
          _ => '',
        },
      )
      .toList();
  return formattedDialogueTurns.join('\n');
}

// أولاً، نقوم بتحميل الذاكرة
final loadedMemory = Runnable.fromMap({
  'question': Runnable.getItemFromMap('question'),
  'memory': Runnable.mapInput((_) => memory.loadMemoryVariables()),
});

// بعد ذلك، نحصل على سجل المحادثة من الذاكرة
final expandedMemory = Runnable.fromMap({
  'question': Runnable.getItemFromMap('question'),
  'chat_history': Runnable.getItemFromMap('memory') |
      Runnable.mapInput<MemoryVariables, List<ChatMessage>>(
        (final input) => input['history'],
      ),
});

// الآن، نولد سؤالاً مستقلاً يتضمن التفاصيل الضرورية من سجل المحادثة
final standaloneQuestion = Runnable.fromMap({
  'standalone_question': Runnable.fromMap({
        'question': Runnable.getItemFromMap('question'),
        'chat_history': Runnable.getItemFromMap<List<ChatMessage>>('chat_history') |
            Runnable.mapInput(formatChatHistory),
      }) |
      condenseQuestionPrompt |
      model |
      stringOutputParser,
});

// الآن نسترجع المستندات
final retrievedDocs = Runnable.fromMap({
  'docs': Runnable.getItemFromMap('standalone_question') | retriever,
  'question': Runnable.getItemFromMap('standalone_question'),
});

// بناء المدخلات لموجه الإجابة
final finalInputs = Runnable.fromMap({
  'context': Runnable.getItemFromMap('docs') |
      Runnable.mapInput<List<Document>, String>(combineDocuments),
  'question': Runnable.getItemFromMap('question'),
});

// نوجه النموذج للحصول على إجابة
final answer = Runnable.fromMap({
  'answer': finalInputs | answerPrompt | model | stringOutputParser,
  'docs': Runnable.getItemFromMap('docs'),
});

// وأخيرًا، نجمع كل ذلك معًا
final conversationalQaChain = loadedMemory |
    expandedMemory |
    standaloneQuestion |
    retrievedDocs |
    answer;

// إذا أضفنا بعض الرسائل إلى الذاكرة،
// فسيتم استخدامها في الاستدعاء التالي
await memory.saveContext(
  inputValues: {
    'question': ChatMessage.humanText('كم تبلغ تكلفة طلبي؟')
  },
  outputValues: {'answer': ChatMessage.ai('عليك دفع 100 يورو')},
);

final res = await conversationalQaChain.invoke({
  'question': 'هل أحصل على شحن مجاني بقيمة طلبي؟',
});
print(res);
// {
//   answer: نعم، بناءً على السياق المعطى، ستحصل على شحن مجاني لطلبك بقيمة 100 يورو لأنه يتجاوز الحد الأدنى المطلوب وهو 30 يورو للشحن المجاني.,
//   docs: [
//     Document{
//       id: 69974fe1-8436-40c7-87d1-c59c5ff1c6a6,
//       pageContent: شحن مجاني: للطلبات التي تزيد عن 30 يورو,
//       metadata: {},
//     }
//   ]
// }
```
