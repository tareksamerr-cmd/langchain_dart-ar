# RunnableMap: تنسيق المدخلات والتزامن

إن `RunnableMap` هو في الأساس خريطة (map) تكون قيمها عبارة عن "runnables". يقوم بتشغيل جميع قيمه بشكل متزامن، ويتم استدعاء كل قيمة مع الإدخال الكلي لـ `RunnableMap`. القيمة النهائية التي يتم إرجاعها هي خريطة تحتوي على نتائج كل قيمة تحت مفتاحها المناسب.

إنه مفيد لتشغيل العمليات بشكل متزامن، ولكنه يمكن أن يكون مفيدًا أيضًا لمعالجة إخراج "Runnable" واحد ليتناسب مع تنسيق إدخال "Runnable" التالي في التسلسل.

هنا، من المتوقع أن يكون إدخال الموجه (prompt) عبارة عن خريطة تحتوي على المفتاحين "context" و "question". إدخال المستخدم هو السؤال فقط. لذلك نحتاج إلى الحصول على السياق باستخدام المسترجع (retriever) الخاص بنا وتمرير إدخال المستخدم تحت مفتاح "question".

```dart
final vectorStore = MemoryVectorStore(
  embeddings: OpenAIEmbeddings(apiKey: openaiApiKey),
);
await vectorStore.addDocuments(
  documents: [
    Document(pageContent: 'LangChain was created by Harrison'),
    Document(pageContent: 'David ported LangChain to Dart in LangChain.dart'),
  ],
);
final retriever = vectorStore.asRetriever();
final promptTemplate = ChatPromptTemplate.fromTemplates([
  (ChatMessageType.system, 'Answer the question based on only the following context:\n{context}'),
  (ChatMessageType.human, '{question}'),
]);
final model = ChatOpenAI(apiKey: openaiApiKey);
const outputParser = StringOutputParser<ChatResult>();

final retrievalChain = Runnable.fromMap<String>({
  'context': retriever,
  'question': Runnable.passthrough(),
}).pipe(promptTemplate).pipe(model).pipe(outputParser);

final res = await retrievalChain.invoke('Who created LangChain.dart?');
print(res);
// David created LangChain.dart.
```

## استخدام Runnable.getItemFromMap كاختصار

أحيانًا تحتاج إلى استخراج قيمة واحدة من خريطة وتمريرها إلى "Runnable" التالي. يمكنك استخدام `Runnable.getItemFromMap` للقيام بذلك. يأخذ خريطة الإدخال ويعيد قيمة المفتاح المقدم.

```dart
final vectorStore = MemoryVectorStore(
  embeddings: OpenAIEmbeddings(apiKey: openaiApiKey),
);
await vectorStore.addDocuments(
  documents: [
    const Document(pageContent: 'LangChain was created by Harrison'),
    const Document(
      pageContent: 'David ported LangChain to Dart in LangChain.dart',
    ),
  ],
);
final retriever = vectorStore.asRetriever();
final promptTemplate = ChatPromptTemplate.fromTemplates(const [
  (
    ChatMessageType.system,
    'Answer the question based on only the following context:\n{context}\n'
        'Answer in the following language: {language}',
  ),
  (ChatMessageType.human, '{question}'),
]);
final model = ChatOpenAI(apiKey: openaiApiKey);
const outputParser = StringOutputParser<ChatResult>();

final retrievalChain = Runnable.fromMap<Map<String, dynamic>>({
  'context': Runnable.getItemFromMap('question').pipe(retriever),
  'question': Runnable.getItemFromMap('question'),
  'language': Runnable.getItemFromMap('language'),
}).pipe(promptTemplate).pipe(model).pipe(outputParser);

final res = await retrievalChain.invoke({
  'question': 'Who created LangChain.dart?',
  'language': 'Spanish',
});
print(res);
// David portó LangChain a Dart en LangChain.dart
```

## تشغيل الخطوات بشكل متزامن

يجعل `RunnableMap` من السهل تنفيذ عدة "Runnables" بشكل متزامن وإرجاع إخراج هذه "Runnables" كخريطة.

```dart
final openaiApiKey = Platform.environment['OPENAI_API_KEY'];
final model = ChatOpenAI(apiKey: openaiApiKey);
const outputParser = StringOutputParser<ChatResult>();

final jokeChain = PromptTemplate.fromTemplate('tell me a joke about {topic}')
    .pipe(model)
    .pipe(outputParser);
final poemChain =
    PromptTemplate.fromTemplate('write a 2-line poem about {topic}')
        .pipe(model)
        .pipe(outputParser);

final mapChain = Runnable.fromMap<Map<String, dynamic>>({
  'joke': jokeChain,
  'poem': poemChain,
});

final res = await mapChain.invoke({
  'topic': 'bear',
});
print(res);
// {joke: Why did the bear bring a flashlight to the party? Because he wanted to be the "light" of the party!, 
//  poem: In the forest's hush, the bear prowls wide, A silent guardian, a force of nature's pride.}
```

يتم تشغيل كل فرع من `RunnableMap` على نفس المعزل (isolate)، ولكن يتم تشغيلها بشكل متزامن. في المثال أعلاه، يتم إجراء الطلبين إلى OpenAI API بشكل متزامن، دون انتظار انتهاء الأول قبل بدء الثاني.
