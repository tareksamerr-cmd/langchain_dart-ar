---
title : ربط المكونات القابلة للتنفيذ
---

إحدى المزايا الرئيسية لواجهة `Runnable` هي إمكانية “ربط” أي مكونين قابلين للتنفيذ معًا في تسلسل. يتم تمرير مخرجات استدعاء `.invoke()` للمكون السابق كمدخل للمكون التالي. يمكن فعل ذلك باستخدام الدالة `.pipe()` (أو العامل `|`، وهو اختصار مناسب لـ `.pipe()`). الناتج `RunnableSequence` هو بحد ذاته مكون قابل للتنفيذ، مما يعني أنه يمكن استدعاؤه أو تدفقه أو ربطه مثل أي مكون قابل للتنفيذ آخر.

> ملاحظة: عند استخدام العامل `|`، يتم دائمًا تحديد نوع مخرجات المكون الأخير على أنه `Object` بسبب [قيود لغة Dart](https://github.com/dart-lang/language/issues/1044). إذا كنت بحاجة إلى الحفاظ على نوع المخرجات، استخدم الدالة `.pipe()` بدلاً من ذلك.

## عامل الربط

لتوضيح كيفية عمل ذلك، دعنا نمر بمثال. سنستعرض نمطًا شائعًا في LangChain: استخدام قالب تعليمات (prompt template) لتنسيق المدخلات إلى نموذج محادثة، ثم تحويل مخرجات رسالة الدردشة إلى نص باستخدام محلل المخرجات.

```dart
final promptTemplate = ChatPromptTemplate.fromTemplate(
  'Tell me a joke about {topic}',
);
final model = ChatOpenAI(apiKey: openaiApiKey);
const outputParser = StringOutputParser<ChatResult>();

final chain = promptTemplate.pipe(model).pipe(outputParser);
```

كل من القوالب والنماذج قابلة للتنفيذ، ونوع المخرجات من استدعاء القالب هو نفسه نوع المدخلات لنموذج الدردشة، لذا يمكننا ربطهما معًا. يمكننا بعد ذلك استدعاء التسلسل الناتج مثل أي مكون قابل للتنفيذ:

```dart
final res = await chain.invoke({'topic': 'bears'});
print(res);
// Why don't bears wear socks?
// Because they have bear feet!
```

تنسيق المدخلات والمخرجات

يمكننا دمج هذا التسلسل مع مكونات أخرى قابلة للتنفيذ لإنشاء تسلسل آخر. قد يتضمن ذلك بعض تنسيق المدخلات والمخرجات باستخدام أنواع أخرى من المكونات القابلة للتنفيذ، حسب المدخلات والمخرجات المطلوبة لمكونات التسلسل.

على سبيل المثال، لنفترض أننا نريد دمج تسلسل إنشاء النكات مع تسلسل آخر يقيّم ما إذا كانت النكتة الناتجة مضحكة.

سنحتاج إلى توخي الحذر في كيفية تنسيق المدخلات للتسلسل التالي. في المثال أدناه، نستخدم RunnableMap الذي يقوم بتنفيذ جميع قيمه بشكل متزامن ويعيد خريطة تحتوي على النتائج، والتي يمكن بعد ذلك تمريرها إلى قالب التعليمات.

```dart
final analysisPrompt = ChatPromptTemplate.fromTemplate(
  'is this a funny joke? {joke}',
);
final composedChain = Runnable.fromMap({
  'joke': chain,
}).pipe(analysisPrompt).pipe(model).pipe(outputParser);

final res1 = await composedChain.invoke({'topic': 'bears'});
print(res1);
// Some people may find this joke funny, especially if they enjoy puns or wordplay...
```

بدلاً من استخدام Runnable.fromMap، يمكننا استخدام الطريقة المساعدة Runnable.getMapFromInput التي تنشئ تلقائيًا RunnableMap وتضع قيمة الإدخال في الخريطة بالمفتاح المحدد.

```dart
final composedChain2 = chain
    .pipe(Runnable.getMapFromInput('joke'))
    .pipe(analysisPrompt)
    .pipe(model)
    .pipe(outputParser);
```

خيار آخر هو استخدام Runnable.mapInput الذي يتيح تحويل قيمة الإدخال باستخدام الدالة المقدمة.

```dart
final composedChain3 = chain
    .pipe(Runnable.mapInput((joke) => <String, dynamic>{'joke': joke}))
    .pipe(analysisPrompt)
    .pipe(model)
    .pipe(outputParser);
```

Runnable.fromList

يمكنك أيضًا إنشاء RunnableSequence من قائمة من المكونات القابلة للتنفيذ باستخدام Runnable.fromList.

```dart
final chain = Runnable.fromList([promptTemplate, chatModel]);
```

```