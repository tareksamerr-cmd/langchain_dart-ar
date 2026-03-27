# Mapper: ربط قيم الإدخال

من الشائع أن نحتاج إلى ربط (mapping) قيمة الإخراج لـ "Runnable" سابق بقيمة جديدة تتوافق مع متطلبات الإدخال لـ "Runnable" التالي. هنا يأتي دور `Runnable.mapInput`.

## Runnable.mapInput

يسمح لك `Runnable.mapInput` بتعريف دالة تربط (maps) قيمة الإدخال بقيمة جديدة.

في المثال التالي، نسترجع قائمة من كائنات `Document` من مخزن المتجهات (vector store) الخاص بنا، ونريد دمجها في سلسلة نصية واحدة لتغذيتها في الموجه (prompt) الخاص بنا. للقيام بذلك، نستخدم `Runnable.mapInput` لتطبيق منطق الدمج.

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
final setupAndRetrieval = Runnable.fromMap<String>({
  'context': retriever.pipe(
    Runnable.mapInput((docs) => docs.map((d) => d.pageContent).join('\n')),
  ),
  'question': Runnable.passthrough(),
});

final promptTemplate = ChatPromptTemplate.fromTemplates(const [
  (ChatMessageType.system, 'Answer the question based on only the following context:\n{context}'),
  (ChatMessageType.human, '{question}'),
]);

final model = ChatOpenAI(apiKey: openaiApiKey);
const outputParser = StringOutputParser<ChatResult>();
final chain = setupAndRetrieval
    .pipe(promptTemplate)
    .pipe(model)
    .pipe(outputParser);

final res = await chain.invoke('Who created LangChain.dart?');
print(res);
// David created LangChain.dart
```

## Runnable.mapInputStream

افتراضيًا، عند تشغيل سلسلة (chain) باستخدام `stream` بدلاً من `invoke`، سيتم استدعاء `Runnable.mapInput` لكل عنصر في تدفق الإدخال (input stream). إذا كنت بحاجة إلى مزيد من التحكم في تدفق الإدخال، يمكنك استخدام `Runnable.mapInputStream` بدلاً من ذلك، والذي يأخذ تدفق الإدخال كمعامل ويعيد تدفقًا جديدًا.

في المثال التالي، يقوم النموذج ببث الإخراج في أجزاء (chunks) ويقوم محلل الإخراج (output parser) بمعالجة كل منها على حدة. ومع ذلك، نريد أن تقوم سلسلتنا بإخراج الجزء الأخير فقط. يمكننا استخدام `Runnable.mapInputStream` للحصول على الجزء الأخير من تدفق الإدخال.

```dart
final model = ChatOpenAI(
  apiKey: openAiApiKey,
  defaultOptions: ChatOpenAIOptions(
    responseFormat: ChatOpenAIResponseFormat.jsonObject,
  ),
);
final parser = JsonOutputParser<ChatResult>();
final mapper = Runnable.mapInputStream((Stream<Map<String, dynamic>> inputStream) async* {
  yield await inputStream.last;
});

final chain = model.pipe(parser).pipe(mapper);

final stream = chain.stream(
  PromptValue.string(
    'Output a list of the countries france, spain and japan and their '
        'populations in JSON format. Use a dict with an outer key of '
        '"countries" which contains a list of countries. '
        'Each country should have the key "name" and "population"',
  ),
);
await stream.forEach((final chunk) => print('$chunk|'));
// {countries: [{name: France, population: 65273511}, {name: Spain, population: 46754778}, {name: Japan, population: 126476461}]}|
```

> ملاحظة: لحالات الاستخدام الأكثر تعقيدًا حيث تريد تعريف منطق منفصل عند تشغيل السلسلة باستخدام `invoke` أو `stream`، يمكنك استخدام `Runnable.function`.

## Runnable.getItemFromMap

أحيانًا يُرجع "Runnable" السابق خريطة (map)، وتريد الحصول على قيمة منها لتغذيتها إلى "Runnable" التالي. يمكنك استخدام `Runnable.getItemFromMap` للحصول على قيمة من خريطة الإدخال.

في المثال التالي، نريد تغذية "retriever" بالسؤال، ولكن الإدخال عبارة عن خريطة تحتوي على عدة قيم أخرى. يمكننا استخدام `Runnable.getItemFromMap` للحصول على السؤال من خريطة الإدخال، وكذلك لنشر القيم الأخرى إلى "Runnable" التالي.

```dart
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

> ملاحظة: هذا يعادل `Runnable.mapInput<Map<String, dynamic>, RunOutput>((input) => input[key])`

## Runnable.getMapFromInput

أحيانًا يُرجع "Runnable" السابق عنصرًا واحدًا، ولكن "Runnable" التالي يتوقع خريطة. يمكنك استخدام `Runnable.getMapFromInput` لتنسيق الإدخال لـ "Runnable" التالي.

في المثال التالي، نريد أن يكون نوع إدخال سلسلتنا (chain) عبارة عن سلسلة نصية (String)، ولكن قالب الموجه (prompt template) يتوقع خريطة. يمكننا استخدام `Runnable.getMapFromInput` لتنسيق الإدخال لقالب الموجه.

```dart
final model = ChatOpenAI(apiKey: openaiApiKey);
const outputParser = StringOutputParser<ChatResult>();

final promptTemplate = ChatPromptTemplate.fromTemplates([
  (
  ChatMessageType.system,
  'Write out the following equation using algebraic symbols then solve it. '
      'Use the format\n\nEQUATION:...\nSOLUTION:...\n\n',
  ),
  (ChatMessageType.human, '{equation_statement}'),
]);

final chain = Runnable.getMapFromInput<String>('equation_statement')
    .pipe(promptTemplate)
    .pipe(model)
    .pipe(outputParser);

final res = await chain.invoke('x raised to the third plus seven equals 12');
print(res);
// EQUATION: \(x^3 + 7 = 12\)
//
// SOLUTION:
// Subtract 7 from both sides:
// \(x^3 = 5\)
//
// Take the cube root of both sides:
// \(x = \sqrt[3]{5}\)
```

> ملاحظة: هذا يعادل `Runnable.mapInput<RunInput, Map<String, dynamic>>((input) => {key: input})`
