# البدء (Get started)

تسهل LCEL بناء سلاسل معقدة من المكونات الأساسية، وتدعم وظائف جاهزة مثل البث (streaming)، والتوازي (parallelism)، والتسجيل (logging).

# مثال أساسي: مطالبة (prompt) + نموذج (model) + محلل مخرجات (output parser)

الحالة الأكثر أساسية وشيوعًا هي ربط قالب مطالبة (prompt template) ونموذج (model) معًا. لمعرفة كيفية عمل ذلك، دعنا ننشئ سلسلة تأخذ موضوعًا وتولد نكتة:

```dart
final openaiApiKey = Platform.environment["OPENAI_API_KEY"];

final promptTemplate = ChatPromptTemplate.fromTemplate(
  'Tell me a joke about {topic}',
);
final model = ChatOpenAI(apiKey: openaiApiKey);
const outputParser = StringOutputParser<ChatResult>();

final chain = promptTemplate.pipe(model).pipe(outputParser);

final res = await chain.invoke({'topic': 'ice cream'});
print(res);
// Why did the ice cream truck break down?
// Because it had too many "scoops"!
```

لاحظ هذا السطر من هذا الكود، حيث نجمع المكونات المختلفة في سلسلة واحدة باستخدام LCEL:

```dart
final chain = promptTemplate.pipe(model).pipe(outputParser);
```

تتشابه طريقة `.pipe()` (أو عامل التشغيل `|`) مع عامل تشغيل الأنابيب (unix pipe operator)، الذي يربط المكونات المختلفة معًا ويغذي المخرجات من مكون واحد كمدخل للمكون التالي.

في هذه السلسلة، يتم تمرير إدخال المستخدم إلى قالب المطالبة (prompt template)، ثم يتم تمرير مخرجات قالب المطالبة إلى النموذج (model)، ثم يتم تمرير مخرجات النموذج إلى محلل المخرجات (output parser). دعنا نلقي نظرة على كل مكون على حدة لفهم ما يحدث حقًا.

## 1. المطالبة (Prompt)

`promptTemplate` هو `BasePromptTemplate`، مما يعني أنه يأخذ خريطة من متغيرات القالب (template variables) وينتج `PromptValue`. `PromptValue` هو غلاف حول مطالبة مكتملة يمكن تمريرها إما إلى `LLM` (الذي يأخذ سلسلة كمدخل) أو `ChatModel` (الذي يأخذ تسلسلًا من الرسائل كمدخل). يمكن أن يعمل مع أي نوع من نماذج اللغة لأنه يحدد منطقًا لإنتاج `ChatMessage` ولإنتاج سلسلة.

```dart
final promptValue = await promptTemplate.invoke({'topic': 'ice cream'});

final messages = promptValue.toChatMessages();
print(messages);
// [HumanChatMessage{
//   content: ChatMessageContentText{
//     text: Tell me a joke about ice cream,
//   },
// }]

final string = promptValue.toString();
print(string);
// Human: Tell me a joke about ice cream
```

## 2. النموذج (Model)

يتم بعد ذلك تمرير `PromptValue` إلى `model`. في هذه الحالة، `model` الخاص بنا هو `ChatModel`، مما يعني أنه سيخرج `ChatMessage`.

```dart
final chatOutput = await model.invoke(promptValue);
print(chatOutput.output);
// AIChatMessage{
//   content: Why did the ice cream truck break down? 
//   Because it couldn't make it over the rocky road!,
// }
```

إذا كان نموذجنا `LLM`، فسيخرج `String`.

```dart
final llm = OpenAI(apiKey: openaiApiKey);
final llmOutput = await llm.invoke(promptValue);
print(llmOutput.output);
// Why did the ice cream go to therapy?
// Because it had a rocky road!
```

## 3. محلل المخرجات (Output parser)

وأخيرًا، نمرر مخرجات `model` الخاصة بنا إلى `outputParser`، وهو `BaseOutputParser` مما يعني أنه يأخذ إما `String` أو `ChatMessage` كمدخل. يقوم `StringOutputParser` على وجه التحديد بتحويل أي إدخال إلى `String`.

```dart
final parsed = await outputParser.invoke(chatOutput);
print(parsed);
// Why did the ice cream go to therapy?
// Because it had a rocky road!
```

## 4. المسار الكامل (Entire Pipeline)

لمتابعة الخطوات:

1. نمرر إدخال المستخدم حول الموضوع المطلوب كـ `{'topic': 'ice cream'}`.
2. يأخذ مكون `promptTemplate` إدخال المستخدم، والذي يستخدم بعد ذلك لإنشاء `PromptValue` بعد استخدام `topic` لإنشاء المطالبة.
3. يأخذ مكون `model` المطالبة التي تم إنشاؤها، ويمررها إلى نموذج دردشة OpenAI للتقييم. المخرجات التي تم إنشاؤها من النموذج هي كائن `ChatMessage` (تحديدًا `AIChatMessage`).
4. أخيرًا، يأخذ مكون `outputParser` `ChatMessage`، ويحولها إلى `String`، والتي يتم إرجاعها من طريقة `invoke`.

![Pipeline](img/pipeline.png)

لاحظ أنه إذا كنت مهتمًا بمخرجات أي مكونات، يمكنك دائمًا اختبار نسخة أصغر من السلسلة مثل `promptTemplate` أو `promptTemplate.pipe(model)` لرؤية النتائج الوسيطة.

```dart
final input = {'topic': 'ice cream'};

final res1 = await promptTemplate.invoke(input);
print(res1.toChatMessages());
// [HumanChatMessage{
//   content: ChatMessageContentText{
//     text: Tell me a joke about ice cream,
//   },
// }]

final res2 = await promptTemplate.pipe(model).invoke(input);
print(res2);
// ChatResult{
//   id: chatcmpl-9J37Tnjm1dGUXqXBF98k7jfexATZW,
//   output: AIChatMessage{
//     content: Why did the ice cream cone go to therapy? Because it had too many sprinkles of emotional issues!,
//   },
//   finishReason: FinishReason.stop,
//   metadata: {
//     model: gpt-4o-mini,
//     created: 1714327251,
//     system_fingerprint: fp_3b956da36b
//   },
//   usage: LanguageModelUsage{
//     promptTokens: 14,
//     promptBillableCharacters: null,
//     responseTokens: 21,
//     responseBillableCharacters: null,
//     totalTokens: 35
//     },
//   streaming: false
// }
```

## مثال بحث RAG (RAG Search Example)

بالنسبة لمثالنا التالي، نريد تشغيل سلسلة توليد معززة بالاسترجاع (retrieval-augmented generation chain) لإضافة بعض السياق عند الإجابة على الأسئلة.

```dart
// 1. Create a vector store and add documents to it
final vectorStore = MemoryVectorStore(
  embeddings: OpenAIEmbeddings(apiKey: openaiApiKey),
);
await vectorStore.addDocuments(
  documents: [
    Document(pageContent: 'LangChain was created by Harrison'),
    Document(pageContent: 'David ported LangChain to Dart in LangChain.dart'),
  ],
);

// 2. Define the retrieval chain
final retriever = vectorStore.asRetriever();
final setupAndRetrieval = Runnable.fromMap<String>({
  'context': retriever.pipe(
    Runnable.mapInput((docs) => docs.map((d) => d.pageContent).join('\n')),
  ),
  'question': Runnable.passthrough(),
});

// 3. Construct a RAG prompt template
final promptTemplate = ChatPromptTemplate.fromTemplates([
  (ChatMessageType.system, 'Answer the question based on only the following context:\n{context}'),
  (ChatMessageType.human, '{question}'),
]);

// 4. Define the final chain
final model = ChatOpenAI(apiKey: openaiApiKey);
const outputParser = StringOutputParser<ChatResult>();
final chain = setupAndRetrieval
    .pipe(promptTemplate)
    .pipe(model)
    .pipe(outputParser);

// 5. Run the pipeline
final res = await chain.invoke('Who created LangChain.dart?');
print(res);
// David created LangChain.dart
```

في هذه الحالة، السلسلة المركبة هي:

```dart
final chain = setupAndRetrieval
    .pipe(promptTemplate)
    .pipe(model)
    .pipe(outputParser);
```

لتوضيح ذلك، يمكننا أولاً أن نرى أن قالب المطالبة (prompt template) أعلاه يأخذ `context` و `question` كقيم ليتم استبدالها في المطالبة. قبل بناء قالب المطالبة، نريد استرداد المستندات ذات الصلة بالبحث وتضمينها كجزء من السياق.

كخطوة أولية، قمنا بإعداد المسترجع (retriever) باستخدام مخزن في الذاكرة (in memory store)، والذي يمكنه استرداد المستندات بناءً على استعلام. هذا مكون قابل للتشغيل (runnable component) أيضًا يمكن ربطه بمكونات أخرى، ولكن يمكنك أيضًا محاولة تشغيله بشكل منفصل:

```dart
final res1 = await retriever.invoke('Who created LangChain.dart?');
print(res1);
// [Document{pageContent: David ported LangChain to Dart in LangChain.dart}, 
// Document{pageContent: LangChain was created by Harrison, metadata: {}}]
```

ثم نستخدم `RunnableMap` لإعداد المدخلات المتوقعة في المطالبة باستخدام سلسلة تحتوي على المستندات المسترجعة المدمجة بالإضافة إلى سؤال المستخدم الأصلي، باستخدام `retriever` للبحث عن المستندات، و `RunnableMapInput` لدمج المستندات و `RunnablePassthrough` لتمرير سؤال المستخدم:

```dart
final setupAndRetrieval = Runnable.fromMap<String>({
  'context': retriever.pipe(
    Runnable.mapInput((docs) => docs.map((d) => d.pageContent).join('\n')),
  ),
  'question': Runnable.passthrough(),
});
```

للمراجعة، السلسلة الكاملة هي:

```dart
final chain = setupAndRetrieval
    .pipe(promptTemplate)
    .pipe(model)
    .pipe(outputParser);
```

مع سير العمل كالتالي:
1. الخطوات الأولى تنشئ كائن `RunnableMap` بإدخالين. الإدخال الأول، `context` سيتضمن نتائج المستندات المدمجة التي تم جلبها بواسطة المسترجع (retriever). الإدخال الثاني، `question` سيحتوي على سؤال المستخدم الأصلي. لتمرير `question`، نستخدم `RunnablePassthrough` لنسخ هذا الإدخال.
2. يتم تغذية الخريطة من الخطوة أعلاه إلى مكون `promptTemplate`. ثم يأخذ إدخال المستخدم وهو `question` بالإضافة إلى المستندات المسترجعة وهي `context` لإنشاء مطالبة وإخراج `PromptValue`.
3. يأخذ مكون `model` المطالبة التي تم إنشاؤها، ويمررها إلى نموذج OpenAI LLM للتقييم. المخرجات التي تم إنشاؤها من النموذج هي كائن `ChatResult`.
4. أخيرًا، يأخذ مكون `outputParser` `ChatResult`، ويحولها إلى سلسلة Dart، والتي يتم إرجاعها من طريقة `invoke`.

![RAG Pipeline](img/rag_pipeline.png)
