# الموجه + النموذج اللغوي الكبير (LLM)

التكوين الأكثر شيوعًا وقيمة هو الجمع بين:

```
PromptTemplate / ChatPromptTemplate -> LLM / ChatModel -> OutputParser
```

تقريبًا جميع السلاسل الأخرى التي ستبنيها ستستخدم هذه اللبنة الأساسية.

## PromptTemplate + LLM

أبسط تكوين هو مجرد دمج موجه ونموذج لإنشاء سلسلة تأخذ مدخلات المستخدم، وتضيفها إلى موجه، وتمررها إلى نموذج، وتعيد المدخلات الخام للنموذج.

ملاحظة: يمكنك مزج ومطابقة `PromptTemplate`/`ChatPromptTemplate` و `LLM`/`ChatModel` كما يحلو لك هنا.

```dart
final openaiApiKey = Platform.environment["OPENAI_API_KEY"];
final model = ChatOpenAI(apiKey: openaiApiKey);

final promptTemplate = ChatPromptTemplate.fromTemplate(
  'أخبرني نكتة عن {foo}',
);

final chain = promptTemplate | model;

final res = await chain.invoke({'foo': 'الدببة'});
print(res);
// ChatResult{
//   id: chatcmpl-9LBNiPXHzWIwc02rR6sS1HTcL9pOk,
//   output: AIChatMessage{
//     content: لماذا لا ترتدي الدببة الأحذية؟\nلأن أقدامها عارية!,
//   },
//   finishReason: FinishReason.stop,
//   metadata: {
//     model: gpt-4o-mini,
//     created: 1714835666,
//     system_fingerprint: fp_3b956da36b
//   },
//   usage: LanguageModelUsage{
//     promptTokens: 13,
//     responseTokens: 13,
//     totalTokens: 26,
//   },
//   streaming: false
// }
```

غالبًا ما نرغب في إرفاق خيارات سيتم تمريرها إلى كل استدعاء للنموذج. يمكنك القيام بذلك بطريقتين:

1.  تكوين الخيارات الافتراضية عند إنشاء النموذج. سيتم تطبيق هذا على جميع استدعاءات النموذج.
2.  تكوين الخيارات عند استخدام النموذج في سلسلة باستخدام طريقة `.bind`. سيتم تطبيق هذا فقط على الاستدعاءات في تلك السلسلة.

دعنا نلقي نظرة على بعض الأمثلة:

### إرفاق تسلسلات التوقف (Stop Sequences)

```dart
final openaiApiKey = Platform.environment["OPENAI_API_KEY"];
final model = ChatOpenAI(apiKey: openaiApiKey);

final promptTemplate = ChatPromptTemplate.fromTemplate(
  'أخبرني نكتة عن {foo}',
);

final chain = promptTemplate | model.bind(ChatOpenAIOptions(stop: ['\n']));

final res = await chain.invoke({'foo': 'الدببة'});
print(res);
// ChatResult{
//   id: chatcmpl-9LBOohTtdg12zD8zzz2GX1ib24UXO,
//   output: AIChatMessage{
//     content: لماذا لا ترتدي الدببة الأحذية؟ ,
//   },
//   finishReason: FinishReason.stop,
//   metadata: {
//     model: gpt-4o-mini,
//     created: 1714835734,
//     system_fingerprint: fp_a450710239
//   },
//   usage: LanguageModelUsage{
//     promptTokens: 13,
//     responseTokens: 8,
//     totalTokens: 21
//   },
//   streaming: false
// }
```

### إرفاق معلومات استدعاء الأداة (Tool Call information)

```dart
final openaiApiKey = Platform.environment["OPENAI_API_KEY"];
final model = ChatOpenAI(apiKey: openaiApiKey);

final promptTemplate = ChatPromptTemplate.fromTemplate(
  'أخبرني نكتة عن {foo}',
);

const tool = ToolSpec(
  name: 'joke',
  description: 'نكتة',
  inputJsonSchema: {
    'type': 'object',
    'properties': {
      'setup': {
        'type': 'string',
        'description': 'مقدمة النكتة',
      },
      'punchline': {
        'type': 'string',
        'description': 'خاتمة النكتة',
      },
    },
    'required': ['setup', 'punchline'],
  },
);

final chain = promptTemplate |
    model.bind(
      ChatOpenAIOptions(
        tools: const [tool],
        toolChoice: ChatToolChoice.forced(name: tool.name),
      ),
    );

final res = await chain.invoke({'foo': 'الدببة'});
print(res);
// ChatResult{
//   id: chatcmpl-9LBPyaZcFMgjmOvkD0JJKAyA4Cihb,
//   output: AIChatMessage{
//     content: ,
//     toolCalls: [
//       AIChatMessageToolCall{
//         id: call_JIhyfu6jdIXaDHfYzbBwCKdb,
//         name: joke,
//         argumentsRaw: {"setup":"لماذا لا تحب الدببة الوجبات السريعة؟","punchline":"لأنها لا تستطيع الإمساك بها!"},
//         arguments: {
//           setup: لماذا لا تحب الدببة الوجبات السريعة؟,
//           punchline: لأنها لا تستطيع الإمساك بها!
//         },
//       }
//     ],
//   },
//   finishReason: FinishReason.stop,
//   metadata: {
//     model: gpt-4o-mini,
//     created: 1714835806,
//     system_fingerprint: fp_3b956da36b
//   },
//   usage: LanguageModelUsage{
//     promptTokens: 77,
//     responseTokens: 24,
//     totalTokens: 101
//   },
//   streaming: false
// }
```

## PromptTemplate + LLM + OutputParser

يمكننا أيضًا إضافة محلل مخرجات (OutputParser) لتحويل مخرجات LLM/ChatModel الخام بسهولة إلى تنسيق متسق.

### محلل مخرجات السلسلة النصية (String Output Parser)

إذا أردنا فقط المخرجات النصية، يمكننا استخدام `StringOutputParser`:

```dart
final openaiApiKey = Platform.environment["OPENAI_API_KEY"];
final model = ChatOpenAI(apiKey: openaiApiKey);

final promptTemplate = ChatPromptTemplate.fromTemplate(
  'أخبرني نكتة عن {foo}',
);

final chain = promptTemplate | model | StringOutputParser();

final res = await chain.invoke({'foo': 'الدببة'});
print(res);
// لماذا لا ترتدي الدببة الأحذية؟ لأن أقدامها عارية!
```

لاحظ أن هذا يعيد الآن سلسلة نصية - وهو تنسيق أكثر قابلية للاستخدام للمهام اللاحقة.

### محلل مخرجات الأدوات (Tools Output Parser)

عند تحديد أداة يجب أن يستدعيها النموذج، قد ترغب فقط في تحليل استدعاء الأداة مباشرة.

```dart
final openaiApiKey = Platform.environment["OPENAI_API_KEY"];
final model = ChatOpenAI(apiKey: openaiApiKey);

final promptTemplate = ChatPromptTemplate.fromTemplate(
  'أخبرني نكتة عن {foo}',
);

const tool = ToolSpec(
  name: 'joke',
  description: 'نكتة',
  inputJsonSchema: {
    'type': 'object',
    'properties': {
      'setup': {
        'type': 'string',
        'description': 'مقدمة النكتة',
      },
      'punchline': {
        'type': 'string',
        'description': 'خاتمة النكتة',
      },
    },
    'required': ['setup', 'punchline'],
  },
);

final chain = promptTemplate |
    model.bind(
      ChatOpenAIOptions(
        tools: const [tool],
        toolChoice: ChatToolChoice.forced(name: tool.name),
      ),
    ) |
    ToolsOutputParser();

final res = await chain.invoke({'foo': 'الدببة'});
print(res);
// [ParsedToolCall{
//   id: call_tDYrlcVwk7bCi9oh5IuknwHu,
//   name: joke,
//   arguments: {
//     setup: ماذا تسمي الدب الذي لا يملك أسنان؟, 
//     punchline: دب حلوى!
//   },
// }]
```

## تبسيط المدخلات (Simplifying input)

لجعل الاستدعاء أبسط، يمكننا إضافة `RunnableMap` للعناية بإنشاء خريطة مدخلات الموجه باستخدام `RunnablePassthrough` للحصول على المدخلات:

```dart
final map = Runnable.fromMap({
  'foo': Runnable.passthrough(),
});
final chain = map | promptTemplate | model | StringOutputParser();
```

*`Runnable.passthrough()` هي طريقة مساعدة تنشئ كائن `RunnablePassthrough`. هذا هو `Runnable` الذي يأخذ المدخلات التي يتلقاها ويمررها كمخرجات.*

ومع ذلك، هذا مطول بعض الشيء. يمكننا تبسيطه باستخدام `Runnable.getMapFromInput` الذي يقوم بنفس الشيء ضمنيًا:

```dart
final chain = Runnable.getMapFromInput('foo') |
    promptTemplate |
    model |
    StringOutputParser();
```

الآن، يمكننا استدعاء السلسلة بمدخلاتنا التي نهتم بها فقط:

```dart
final res = await chain.invoke('الدببة');
print(res);
// لماذا لا ترتدي الدببة الأحذية؟ لأن أقدامها عارية!
```
