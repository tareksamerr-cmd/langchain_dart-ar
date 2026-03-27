# البث (Streaming) باستخدام LangChain

يعد البث أمرًا بالغ الأهمية لجعل التطبيقات المستندة إلى نماذج اللغة الكبيرة (LLMs) تبدو سريعة الاستجابة للمستخدمين النهائيين.

تطبق بدائيات LangChain المهمة مثل نماذج اللغة الكبيرة (LLMs)، والمحللات (parsers)، والمطالبات (prompts)، والمسترجعات (retrievers)، والوكلاء (agents) واجهة LangChain [Runnable Interface](/expression_language/interface.md).

سيوضح لك هذا الدليل كيفية استخدام `.stream()` لبث المخرجات النهائية للسلسلة.

## استخدام البث (Stream)

تنفذ جميع كائنات `Runnable` طريقة تسمى `stream`.

تم تصميم هذه الطرق لبث المخرجات النهائية في أجزاء (chunks)، وتقديم كل جزء بمجرد توفره.

البث ممكن فقط إذا كانت جميع الخطوات في البرنامج تعرف كيفية معالجة **تدفق الإدخال (input stream)**؛ أي معالجة جزء إدخال واحدًا تلو الآخر، وتقديم جزء إخراج مطابق.

يمكن أن يختلف تعقيد هذه المعالجة، من المهام المباشرة مثل إصدار الرموز (tokens) التي تنتجها نماذج اللغة الكبيرة (LLM)، إلى المهام الأكثر تحديًا مثل بث أجزاء من نتائج JSON قبل اكتمال JSON بأكمله.

أفضل مكان للبدء في استكشاف البث هو مع أهم المكونات في تطبيقات نماذج اللغة الكبيرة (LLM apps) - النماذج نفسها!

## نماذج اللغة الكبيرة (LLMs) ونماذج الدردشة (Chat Models)

تعد نماذج اللغة الكبيرة ومتغيرات الدردشة الخاصة بها هي عنق الزجاجة الأساسي في التطبيقات المستندة إلى نماذج اللغة الكبيرة (LLM).

يمكن أن تستغرق نماذج اللغة الكبيرة **عدة ثوانٍ** لإنشاء استجابة كاملة لاستعلام. هذا أبطأ بكثير من عتبة **~200-300 مللي ثانية** التي يشعر عندها التطبيق بالاستجابة للمستخدم النهائي.

الاستراتيجية الرئيسية لجعل التطبيق يبدو أكثر استجابة هي إظهار التقدم الوسيط؛ على سبيل المثال، لبث المخرجات من النموذج رمزًا تلو الآخر.

```dart
final model = ChatOpenAI(apiKey: openAiApiKey);

final stream = model.stream(PromptValue.string("Hello! Tell me about yourself."));
final chunks = <ChatResult>[];
await for (final chunk in stream) {
  chunks.add(chunk);
  stdout.write("${chunk.output.content}|");
}
// Hello|!| I| am| a| language| model| AI| created| by| Open|AI|,|...
```

دعنا نلقي نظرة على أحد الأجزاء الخام (raw chunks):

```dart
print(chunks.first);
// ChatResult{
//   id: chatcmpl-9IHQvyTl9fyVmF7P6zamGaX1XAN6d,
//   output: AIChatMessage{
//     content: Hello,
//   },
//   finishReason: FinishReason.unspecified,
//   metadata: {
//     model: gpt-4o-mini,
//     created: 1714143945,
//     system_fingerprint: fp_3b956da36b
//   },
//   streaming: true
// }
```

لقد حصلنا على مثيل `ChatResult` كالمعتاد، ولكنه يحتوي فقط على جزء من الاستجابة الكاملة (`Hello`).

يمكننا تحديد النتائج التي يتم بثها عن طريق التحقق من حقل `streaming`. كائنات النتائج إضافية بطبيعتها - يمكن للمرء ببساطة إضافتها باستخدام طريقة `.concat()` للحصول على حالة الاستجابة حتى الآن!

```dart
final result = chunks.sublist(0, 6).reduce((prev, next) => prev.concat(next));
print(result);
// ChatResult{
//   id: chatcmpl-9IHQvyTl9fyVmF7P6zamGaX1XAN6d,
//   output: AIChatMessage{
//     content: Hello! I am a language model
//   },
//   finishReason: FinishReason.unspecified,
//   metadata: {
//     model: gpt-4o-mini,
//     created: 1714143945,
//     system_fingerprint: fp_3b956da36b
//   },
//   streaming: true
// }
```

## السلاسل (Chains)

تتضمن جميع تطبيقات نماذج اللغة الكبيرة (LLM) تقريبًا خطوات أكثر من مجرد استدعاء لنموذج لغة.

دعنا نبني سلسلة بسيطة باستخدام لغة تعبير LangChain (LCEL) تجمع بين مطالبة (prompt)، ونموذج (model)، ومحلل (parser) ونتحقق من أن البث يعمل.

سنستخدم `StringOutputParser` لتحليل المخرجات من النموذج. هذا محلل بسيط يستخرج المخرجات النصية من النتيجة التي يعيدها النموذج.

> LCEL هي طريقة تصريحية لتحديد 
برنامج (program) عن طريق ربط بدائيات LangChain المختلفة معًا. تستفيد السلاسل التي تم إنشاؤها باستخدام LCEL من تطبيق تلقائي للبث (stream)، مما يسمح ببث المخرجات النهائية. في الواقع، السلاسل التي تم إنشاؤها باستخدام LCEL تنفذ واجهة Runnable القياسية بأكملها.

```dart
final model = ChatOpenAI(apiKey: openAiApiKey);
final prompt = ChatPromptTemplate.fromTemplate("Tell me a joke about {topic}");
const parser = StringOutputParser<ChatResult>();

final chain = prompt.pipe(model).pipe(parser);

final stream = chain.stream({"topic": "parrot"});
await stream.forEach((final chunk) => stdout.write("$chunk|"));
// |Why| don|'t| you| ever| play| hide| and| seek| with| a| par|rot|?|
// |Because| they| always| squ|awk| when| they| find| you|!||
```

قد تلاحظ أعلاه أن المحلل (parser) لا يمنع مخرجات البث من النموذج، وبدلاً من ذلك يعالج كل جزء على حدة. تدعم العديد من بدائيات LCEL أيضًا هذا النوع من البث التحويلي (transform-style passthrough streaming)، والذي يمكن أن يكون مناسبًا جدًا عند بناء التطبيقات.

> ليس عليك استخدام لغة تعبير LangChain لاستخدام LangChain ويمكنك بدلاً من ذلك الاعتماد على نهج برمجة إلزامي قياسي عن طريق استدعاء `invoke` أو `batch` أو `stream` على كل مكون على حدة، وتعيين النتائج للمتغيرات ثم استخدامها في المراحل اللاحقة كما تراه مناسبًا.
> 
> إذا كان ذلك يلبي احتياجاتك، فلا بأس بذلك 👌!

## العمل مع تدفقات الإدخال (Input Streams)

ماذا لو أردت بث JSON من المخرجات أثناء إنشائها؟

إذا كنت ستعتمد على `json.decode` لتحليل JSON الجزئي، فسيفشل التحليل لأن JSON الجزئي لن يكون JSON صالحًا.

من المحتمل أن تكون في حيرة تامة بشأن ما يجب فعله وتدعي أنه من المستحيل بث JSON.

حسنًا، اتضح أن هناك طريقة للقيام بذلك - يحتاج المحلل (parser) إلى العمل على تدفق الإدخال (input stream)، ومحاولة "إكمال تلقائي" لـ JSON الجزئي إلى حالة صالحة.

دعنا نرى مثل هذا المحلل في العمل لفهم ما يعنيه هذا.

```dart
final model = ChatOpenAI(
  apiKey: openAiApiKey,
  defaultOptions: const ChatOpenAIOptions(
    responseFormat: ChatOpenAIResponseFormat.jsonObject,
  ),
);
final parser = JsonOutputParser<ChatResult>();

final chain = model.pipe(parser);

final stream = chain.stream(
  PromptValue.string(
    "Output a list of the countries france, spain and japan and their "
    "populations in JSON format. Use a dict with an outer key of "
    "\"countries\" which contains a list of countries. "
    "Each country should have the key \"name\" and \"population\"",
  ),
);
await stream.forEach((final chunk) => print("$chunk|"));
// {}|
// {countries: []}|
// {countries: [{}]}|
// {countries: [{name: }]}|
// {countries: [{name: France}]}|
// {countries: [{name: France, population: 670}]}|
// {countries: [{name: France, population: 670760}]}|
// {countries: [{name: France, population: 67076000}]}|
// {countries: [{name: France, population: 67076000}, {}]}|
// {countries: [{name: France, population: 67076000}, {name: Spain}]}|
// {countries: [{name: France, population: 67076000}, {name: Spain, population: 467}]}|
// {countries: [{name: France, population: 67076000}, {name: Spain, population: 467237}]}|
// {countries: [{name: France, population: 67076000}, {name: Spain, population: 46723749}]}|
// {countries: [{name: France, population: 67076000}, {name: Spain, population: 46723749}, {}]}|
// {countries: [{name: France, population: 67076000}, {name: Spain, population: 46723749}, {name: Japan}]}|
// {countries: [{name: France, population: 67076000}, {name: Spain, population: 46723749}, {name: Japan, population: 126}]}|
// {countries: [{name: France, population: 67076000}, {name: Spain, population: 46723749}, {name: Japan, population: 126476}]}|
// {countries: [{name: France, population: 67076000}, {name: Spain, population: 46723749}, {name: Japan, population: 126476461}]}|
```

### تحويل التدفقات (Transforming Streams)

الآن، بدلاً من إرجاع كائن JSON الكامل، نريد استخراج أسماء الدول من JSON أثناء إنشائها. يمكننا استخدام `Runnable.mapInputStream` لتحويل التدفق.

```dart
final mapper = Runnable.mapInputStream((Stream<Map<String, dynamic>> inputStream) {
  return inputStream.map((input) {
    final countries = (input["countries"] as List?)?.cast<Map<String, dynamic>>() ?? [];
    final countryNames = countries
        .map((country) => country["name"] as String?)
        .where((c) => c != null && c.isNotEmpty);
    return countryNames.join(", ");
  }).distinct();
});

final chain = model.pipe(parser).pipe(mapper);

final stream = chain.stream(
  PromptValue.string(
    "Output a list of the countries france, spain and japan and their "
    "populations in JSON format. Use a dict with an outer key of "
    "\"countries\" which contains a list of countries. "
    "Each country should have the key \"name\" and \"population\"",
  ),
);
await stream.forEach(print);
// France
// France, Spain
// France, Spain, Japan
```

## المكونات غير المتدفقة (Non-streaming components)

لا يمكن للمكونات القابلة للتشغيل (runnables) التالية معالجة أجزاء الإدخال الفردية وبدلاً من ذلك تقوم بتجميع إدخال البث من الخطوة السابقة في قيمة واحدة قبل معالجتها:
- `PromptTemplate`
- `ChatPromptTemplate`
- `LLM`
- `ChatModel`
- `Retriever`
- `Tool`
- `RunnableFunction`
- `RunnableRouter`

دعنا نرى ما يحدث عندما نحاول بثها. 🤨

```dart
final openaiApiKey = Platform.environment["OPENAI_API_KEY"];

final vectorStore = MemoryVectorStore(
  embeddings: OpenAIEmbeddings(apiKey: openaiApiKey),
);
await vectorStore.addDocuments(
  documents: const [
    Document(pageContent: "LangChain was created by Harrison"),
    Document(
      pageContent: "David ported LangChain to Dart in LangChain.dart",
    ),
  ],
);
final retriever = vectorStore.asRetriever();

await retriever.stream("Who created LangChain.dart?").forEach(print);
// [Document{pageContent: David ported LangChain to Dart in LangChain.dart}, 
// Document{pageContent: LangChain was created by Harrison}]
```

لقد أظهر البث (Stream) للتو النتيجة النهائية من هذا المكون.

هذا جيد 🥹! ليس من الضروري أن تنفذ جميع المكونات البث - ففي بعض الحالات يكون البث غير ضروري أو صعب أو لا معنى له ببساطة.

ستظل سلسلة LCEL التي تم إنشاؤها باستخدام مكونات غير متدفقة قادرة على البث في كثير من الحالات، مع بدء بث المخرجات الجزئية بعد آخر خطوة غير متدفقة في السلسلة.

```dart
final promptTemplate = ChatPromptTemplate.fromTemplates(const [
  (
    ChatMessageType.system,
    "Answer the question based on only the following context:\n{context}",
  ),
  (ChatMessageType.human, "{question}"),
]);
final model = ChatOpenAI(apiKey: openaiApiKey);
const outputParser = StringOutputParser<ChatResult>();

final retrievalChain = Runnable.fromMap<String>({
  "context": retriever,
  "question": Runnable.passthrough(),
}).pipe(promptTemplate).pipe(model).pipe(outputParser);

await retrievalChain
    .stream("Who created LangChain.dart?")
    .forEach((chunk) => stdout.write("$chunk|"));
// |David| created| Lang|Chain|.dart|.||
```
