# Passthrough: تمرير المدخلات كما هي

يسمح لك `RunnablePassthrough` بمفرده بتمرير المدخلات دون تغيير. يُستخدم هذا عادةً بالاقتران مع `RunnableMap` لتمرير البيانات إلى مفتاح جديد في الخريطة.

انظر المثال أدناه:

```dart
final runnable = Runnable.fromMap<Map<String, dynamic>>({
  'passed': Runnable.passthrough(),
  'modified': Runnable.mapInput((input) => (input['num'] as int) + 1),
});

final res = await runnable.invoke({'num': 1});
print(res);
// {passed: {num: 1}, modified: 2}
```

كما رأينا أعلاه، تم استدعاء المفتاح `passed` باستخدام `RunnablePassthrough` وبالتالي قام ببساطة بتمرير `{'num': 1}`.

لقد قمنا أيضًا بتعيين مفتاح ثانٍ في الخريطة باسم `modified`. يستخدم هذا إدخال خريطة لتعيين قيمة واحدة بإضافة 1 إلى الرقم، مما أدى إلى المفتاح `modified` بقيمة 2.

## مثال الاسترجاع (Retrieval Example)

في المثال أدناه، نرى حالة استخدام حيث نستخدم `RunnablePassthrough` جنبًا إلى جنب مع `RunnableMap`.

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

هنا، من المتوقع أن يكون إدخال المطالبة عبارة عن خريطة تحتوي على مفتاحين "context" و "question". إدخال المستخدم هو السؤال فقط. لذلك نحتاج إلى الحصول على السياق باستخدام المسترجع (retriever) الخاص بنا وتمرير إدخال المستخدم تحت المفتاح "question". في هذه الحالة، يسمح لنا `RunnablePassthrough` بتمرير سؤال المستخدم إلى المطالبة والنموذج.
