# سلاسل متعددة

يمكن استخدام `Runnables` بسهولة لدمج سلاسل متعددة:

```dart
final openaiApiKey = Platform.environment["OPENAI_API_KEY"];
final model = ChatOpenAI(apiKey: openaiApiKey);
const stringOutputParser = StringOutputParser<ChatResult>();

final promptTemplate1 = ChatPromptTemplate.fromTemplate(
  'ما هي المدينة التي ينتمي إليها {person}؟ أجب باسم المدينة فقط.',
);

final promptTemplate2 = ChatPromptTemplate.fromTemplate(
  'في أي بلد تقع مدينة {city}؟ أجب بـ {language}.',
);

final cityChain = promptTemplate1 | model | stringOutputParser;
final combinedChain = Runnable.fromMap({
      'city': cityChain,
      'language': Runnable.getItemFromMap('language'),
    }) |
    promptTemplate2 |
    model |
    stringOutputParser;

final res = await combinedChain.invoke({
  'person': 'أوباما',
  'language': 'الإسبانية',
});
print(res);
// La ciudad de Chicago se encuentra en los Estados Unidos.
```

نستخدم `RunnableMap` لتشغيل سلسلتين بالتوازي، إحداهما تحصل على اسم المدينة والأخرى تنشر مدخل `language` فقط. أخيرًا، يتم تمرير مخرجات `RunnableMap` إلى الموجه الثاني وتغذيتها للنموذج.

## Runnable.getItemFromMap و Runnable.passthrough

في المثال أعلاه، نستدعي `combinedChain` بخريطة (Map) ثم نستخدم `Runnable.getItemFromMap` لنشر مدخل `language` إلى الموجه الثاني.

حالة استخدام نموذجية أخرى هي استدعاء السلسلة بمدخل نصي واحد (String) ثم استخدام مزيج من `Runnable.fromMap` و `Runnable.passthrough` لبناء المدخل للموجه الثاني.

دعنا نرى مثالاً آخر مع المزيد من السلاسل ومدخل نصي واحد:

```dart
final openaiApiKey = Platform.environment["OPENAI_API_KEY"];
final model = ChatOpenAI(apiKey: openaiApiKey);

final promptTemplate1 = ChatPromptTemplate.fromTemplate(
  'توليد لون {attribute}. '
  'أعد اسم اللون فقط ولا شيء آخر:',
);
final promptTemplate2 = ChatPromptTemplate.fromTemplate(
  'ما هي فاكهة بلون: {color}. '
  'أعد اسم الفاكهة فقط ولا شيء آخر:',
);
final promptTemplate3 = ChatPromptTemplate.fromTemplate(
  'ما هو البلد الذي يحتوي علمه على اللون: {color}. '
  'أعد اسم البلد فقط ولا شيء آخر:',
);
final promptTemplate4 = ChatPromptTemplate.fromTemplate(
  'ما هو لون {fruit} وعلم {country}؟',
);

final modelParser = model | StringOutputParser();

final colorGenerator = Runnable.getMapFromInput('attribute') |
    promptTemplate1 |
    Runnable.fromMap({
      'color': modelParser,
    });
final colorToFruit = promptTemplate2 | modelParser;
final colorToCountry = promptTemplate3 | modelParser;
final questionGenerator = colorGenerator | Runnable.fromMap({
  'fruit': colorToFruit,
  'country': colorToCountry,
}) | promptTemplate4 | modelParser;

final res = await questionGenerator.invoke('دافئ');
print(res);
// عادة ما يتم تصوير لون Apple باللون الفضي أو الرمادي لشعارها ومنتجاتها. 
// يتكون علم أرمينيا من ثلاثة خطوط أفقية باللون الأحمر والأزرق والبرتقالي من الأعلى إلى الأسفل.
```

## التفرع والدمج (Branching and Merging)

قد ترغب في أن تتم معالجة مخرجات أحد المكونات بواسطة مكونين آخرين أو أكثر. تتيح لك `RunnableMaps` تقسيم أو تفرع السلسلة بحيث يمكن لمكونات متعددة معالجة المدخلات بالتوازي. لاحقًا، يمكن للمكونات الأخرى الانضمام أو دمج النتائج لتوليف استجابة نهائية. ينشئ هذا النوع من السلاسل رسمًا بيانيًا للحسابات يبدو كالتالي:

```
     المدخلات
      / \
     /   \
   الفرع 1 الفرع 2
     \   /
      \ /
      الدمج
```

دعنا نرى مثالاً:

```dart
final openaiApiKey = Platform.environment["OPENAI_API_KEY"];
final model = ChatOpenAI(apiKey: openaiApiKey);
const stringOutputParser = StringOutputParser<ChatResult>();

final planner = Runnable.getMapFromInput() |
    ChatPromptTemplate.fromTemplate('توليد حجة حول: {input}') |
    model |
    stringOutputParser |
    Runnable.getMapFromInput('base_response');

final argumentsFor = ChatPromptTemplate.fromTemplate(
      'اذكر الإيجابيات أو الجوانب الإيجابية لـ {base_response}',
    ) |
    model |
    stringOutputParser;

final argumentsAgainst = ChatPromptTemplate.fromTemplate(
      'اذكر السلبيات أو الجوانب السلبية لـ {base_response}',
    ) |
    model |
    stringOutputParser;

final finalResponder = ChatPromptTemplate.fromPromptMessages([
      AIChatMessagePromptTemplate.fromTemplate(
        '{original_response}'
      ),
      HumanChatMessagePromptTemplate.fromTemplate(
        'الإيجابيات:\n{results_1}\n\nالسلبيات:\n{results_2}',
      ),
      SystemChatMessagePromptTemplate.fromTemplate(
        'توليد استجابة نهائية بناءً على النقد',
      ),
    ]) |
    model |
    stringOutputParser;

final chain = planner |
    Runnable.fromMap({
      'results_1': argumentsFor,
      'results_2': argumentsAgainst,
      'original_response': Runnable.getItemFromMap('base_response'),
    }) |
    finalResponder;

final res = await chain.invoke('سكروم');
print(res);
// بينما يتمتع سكروم بالعديد من الفوائد، من الضروري الاعتراف بالسلبيات أو الجوانب السلبية المحتملة التي تأتي مع تنفيذه ومعالجتها.
// من خلال فهم هذه التحديات، يمكن للفرق اتخاذ الخطوات اللازمة للتخفيف منها وزيادة فعالية سكروم.
//
// لمعالجة نقص القدرة على التنبؤ، يمكن للفرق التركيز على تحسين تقنيات التقدير الخاصة بهم، وإجراء تتبع منتظم للتقدم، واعتماد
// تقنيات مثل تقدير نقاط القصة أو تتبع السرعة. يمكن أن يوفر هذا لأصحاب المصلحة فهمًا أفضل لجداول المشروع الزمنية
// والمخرجات.
//
// ...
//
// في الختام، بينما يواجه سكروم تحدياته، فإن معالجة هذه السلبيات المحتملة من خلال تدابير استباقية يمكن أن تساعد في زيادة الفوائد
// والفعالية القصوى للإطار. من خلال التحسين المستمر وتكييف ممارسات سكروم، يمكن للفرق التغلب على هذه التحديات وتحقيق
// نتائج مشروع ناجحة.
```
