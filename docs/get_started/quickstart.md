# دليل البدء السريع (Quickstart)

في هذا الدليل السريع، سنوضح لك كيفية:

- البدء باستخدام LangChain.dart.
- استخدام المكونات الأساسية والأكثر شيوعًا في LangChain: قوالب الأوامر (prompt templates)، النماذج (models)، ومحللات المخرجات (output parsers).
- استخدام لغة تعبير LangChain (LangChain Expression Language)، وهو البروتوكول الذي بنيت عليه LangChain ويسهل ربط المكونات ببعضها.
- بناء تطبيق بسيط باستخدام LangChain.

هذا قدر لا بأس به لتغطيته! دعنا نتعمق.

## الإعداد (Setup)

للبدء، اتبع [تعليمات التثبيت](/get_started/installation.md) لتثبيت LangChain.dart.

سيتطلب استخدام LangChain.dart عادةً دمجًا مع مزود واحد أو أكثر من مزودي النماذج (model providers)، ومخازن البيانات (data stores)، وواجهات برمجة التطبيقات (APIs)، وما إلى ذلك. لهذا المثال، سنستخدم واجهات برمجة تطبيقات نماذج OpenAI.

أولاً، سنحتاج إلى إضافة حزمة LangChain.dart OpenAI:

```yaml
dependencies:
  langchain: { version }
  langchain_openai: { version }
```

يتطلب الوصول إلى OpenAI API مفتاح API (API key)، والذي يمكنك الحصول عليه عن طريق إنشاء حساب والتوجه [هنا](https://platform.openai.com/account/api-keys).

لا تفرض المكتبة عليك استخدام أي استراتيجية محددة لإدارة المفاتيح. ما عليك سوى تمرير المفتاح إلى مُنشئ `ChatOpenAI`:

```dart
import 'package:langchain/langchain.dart';
import 'package:langchain_openai/langchain_openai.dart';

final llm = ChatOpenAI(apiKey: openaiApiKey);
```

## البناء باستخدام LangChain.dart (Building with LangChain.dart)

توفر LangChain العديد من الوحدات (modules) التي يمكن استخدامها لبناء تطبيقات نماذج اللغة (language model applications). يمكن استخدام الوحدات بشكل مستقل في التطبيقات البسيطة، ويمكن تجميعها لحالات استخدام أكثر تعقيدًا. يتم تشغيل التجميع بواسطة LangChain Expression Language (LCEL)، والذي يحدد واجهة `Runnable` موحدة تنفذها العديد من الوحدات، مما يجعل من الممكن ربط المكونات بسلاسة.

السلسلة (chain) الأبسط والأكثر شيوعًا تحتوي على ثلاثة أشياء:

- LLM/Chat Model: نموذج اللغة (language model) هو محرك التفكير الأساسي هنا. للعمل مع LangChain، تحتاج إلى فهم الأنواع المختلفة لنماذج اللغة وكيفية العمل معها.
- Prompt Template: يوفر هذا تعليمات لنموذج اللغة. يتحكم هذا في ما يخرجه نموذج اللغة، لذا فإن فهم كيفية بناء الـ prompts واستراتيجيات الـ prompting المختلفة أمر بالغ الأهمية.
- Output Parser: تقوم هذه بتحويل الاستجابة الخام من نموذج اللغة إلى تنسيق أكثر قابلية للاستخدام، مما يسهل استخدام المخرجات في المراحل اللاحقة.

في هذا الدليل، سنغطي هذه المكونات الثلاثة بشكل فردي، ثم ننتقل إلى كيفية دمجها. سيساعدك فهم هذه المفاهيم على استخدام وتخصيص تطبيقات LangChain بشكل جيد. تسمح لك معظم تطبيقات LangChain بتكوين النموذج و/أو الـ prompt، لذا فإن معرفة كيفية الاستفادة من ذلك سيكون عامل تمكين كبير.

## نموذج LLM / Chat Model

هناك نوعان من نماذج اللغة:

- `LLM`: النموذج الأساسي يأخذ سلسلة نصية (string) كمدخل ويعيد سلسلة نصية.
- `ChatModel`: النموذج الأساسي يأخذ قائمة من الرسائل (list of messages) كمدخل ويعيد رسالة (message).

السلاسل النصية بسيطة، ولكن ما هي الرسائل بالضبط؟ يتم تعريف واجهة الرسالة الأساسية بواسطة `ChatMessage`، والتي تحتوي على خاصيتين مطلوبتين:

- `content`: محتوى الرسالة. عادة ما يكون سلسلة نصية.
- `role`: الكيان الذي تأتي منه `ChatMessage`.

توفر LangChain عدة كائنات للتمييز بسهولة بين الأدوار المختلفة:

- `HumanChatMessage`: `ChatMessage` قادمة من إنسان/مستخدم.
- `AIChatMessage`: `ChatMessage` قادمة من ذكاء اصطناعي/مساعد.
- `SystemChatMessage`: `ChatMessage` قادمة من النظام.
- `FunctionChatMessage` / `ToolChatMessage`: `ChatMessage` تحتوي على مخرجات استدعاء دالة (function call) أو أداة (tool call).

إذا لم يكن أي من هذه الأدوار مناسبًا، فهناك أيضًا فئة `CustomChatMessage` حيث يمكنك تحديد الدور يدويًا.

توفر LangChain واجهة مشتركة تشاركها كل من `LLMs` و `ChatModels`. ومع ذلك، من المفيد فهم الفرق من أجل بناء الـ prompts الأكثر فعالية لنموذج لغة معين.

أبسط طريقة لاستدعاء `LLM` أو `ChatModel` هي استخدام `.invoke()`، وهي طريقة الاستدعاء العالمية لجميع كائنات LangChain Expression Language (LCEL):

- `LLM.invoke`: يأخذ سلسلة نصية، ويعيد سلسلة نصية.
- `ChatModel.invoke`: يأخذ قائمة من `ChatMessage`، ويعيد `ChatMessage`.

أنواع المدخلات لهذه الطرق هي في الواقع أكثر عمومية من هذا، ولكن للتبسيط هنا يمكننا افتراض أن `LLMs` تأخذ سلاسل نصية فقط وأن `ChatModels` تأخذ قوائم من الرسائل فقط. تحقق من قسم "Go deeper" أدناه لمعرفة المزيد حول استدعاء النموذج.

دعنا نرى كيفية العمل مع هذه الأنواع المختلفة من النماذج وهذه الأنواع المختلفة من المدخلات. أولاً، دعنا نستورد `LLM` و `ChatModel`.

```dart
final llm = OpenAI(apiKey: openaiApiKey);
final chatModel = ChatOpenAI(apiKey: openaiApiKey);
```

تعتبر كائنات `LLM` و `ChatModel` كائنات تكوين (configuration objects) فعالة. يمكنك تهيئتها بمعاملات مثل `temperature` وغيرها، وتمريرها.

```dart
const text = 'What would be a good company name for a company that makes colorful socks?';
final messages = [ChatMessage.humanText(text)];

final res1 = await llm.invoke(PromptValue.string(text));
print(res1.output);
// 'Feetful of Fun'

final res2 = await chatModel.invoke(PromptValue.chat(messages));
print(res2.output);
// AIChatMessage(content='RainbowSock Co.')
```

?> `LLM.invoke` و `ChatModel.invoke` يأخذان كمدخل `PromptValue`. هذا كائن يحدد منطقه المخصص لإرجاع مدخلاته إما كسلسلة نصية أو كرسائل. تحتوي `LLMs` على منطق لتحويل أي من هذه إلى سلسلة نصية، وتحتوي `ChatModels` على منطق لتحويل أي من هذه إلى رسائل. حقيقة أن `LLM` و `ChatModel` يقبلان نفس المدخلات تعني أنه يمكنك تبديلهما مباشرة ببعضهما البعض في معظم السلاسل دون كسر أي شيء، على الرغم من أنه من المهم بالطبع التفكير في كيفية تحويل المدخلات وكيف يمكن أن يؤثر ذلك على أداء النموذج. للتعمق أكثر في النماذج، توجه إلى قسم [نماذج اللغة](/modules/model_io/models/models.md).

## قوالب الأوامر (Prompt templates)

معظم تطبيقات LLM لا تمرر مدخلات المستخدم مباشرة إلى `LLM`. عادةً ما تضيف مدخلات المستخدم إلى جزء أكبر من النص، يسمى قالب الأمر (prompt template)، والذي يوفر سياقًا إضافيًا للمهمة المحددة. 

في المثال السابق، النص الذي مررناه إلى النموذج يحتوي على تعليمات لإنشاء اسم شركة. لتطبيقنا، سيكون رائعًا إذا كان على المستخدم فقط تقديم وصف لشركة/منتج، دون الحاجة إلى القلق بشأن إعطاء النموذج تعليمات.

تساعد `PromptTemplates` في هذا بالضبط! إنها تجمع كل المنطق للانتقال من مدخلات المستخدم إلى prompt منسق بالكامل. يمكن أن يبدأ هذا بسيطًا جدًا - على سبيل المثال، prompt لإنتاج السلسلة المذكورة أعلاه سيكون:

```dart
final prompt = PromptTemplate.fromTemplate(
  'What is a good name for a company that makes {product}?',
);
final res = prompt.format({'product': 'colorful socks'});
print(res);
// 'What is a good name for a company that makes colorful socks?'
```

ومع ذلك، فإن مزايا استخدام هذه على تنسيق السلسلة النصية الخام (raw string formatting) عديدة. يمكنك "تجزئة" المتغيرات (partial out variables) - على سبيل المثال، يمكنك تنسيق بعض المتغيرات فقط في كل مرة. يمكنك تجميعها معًا، ودمج القوالب المختلفة بسهولة في prompt واحد. لشرح هذه الوظائف، راجع [prompts](/modules/model_io/prompts/prompts.md) لمزيد من التفاصيل.

يمكن أيضًا استخدام `PromptTemplates` لإنتاج قائمة من الرسائل. في هذه الحالة، لا يحتوي الـ prompt على معلومات حول المحتوى فحسب، بل يحتوي أيضًا على كل رسالة (دورها، موقعها في القائمة، إلخ). هنا، ما يحدث غالبًا هو أن `ChatPromptTemplate` عبارة عن قائمة من `ChatMessagePromptTemplates`. يحتوي كل `ChatMessagePromptTemplate` على تعليمات حول كيفية تنسيق `ChatMessage` - دورها، ثم محتواها أيضًا. دعنا نلقي نظرة على هذا أدناه:

```dart
const template = 'You are a helpful assistant that translates {input_language} to {output_language}.';
const humanTemplate = '{text}';

final chatPrompt = ChatPromptTemplate.fromTemplates([
  (ChatMessageType.system, template),
  (ChatMessageType.human, humanTemplate),
]);

final res = chatPrompt.formatMessages({
  'input_language': 'English',
  'output_language': 'French',
  'text': 'I love programming.',
});
print(res);
// [
//   SystemChatMessage(content='You are a helpful assistant that translates English to French.'),
//   HumanChatMessage(content='I love programming.')
// ]
```

يمكن أيضًا بناء `ChatPromptTemplates` بطرق أخرى - راجع قسم [prompts](/modules/model_io/prompts/prompts.md) لمزيد من التفاصيل.

## محللات المخرجات (Output parsers)

تقوم `OutputParsers` بتحويل المخرجات الخام لـ LLM إلى تنسيق يمكن استخدامه في المراحل اللاحقة. هناك أنواع قليلة رئيسية من `OutputParsers`، بما في ذلك:

- تحويل النص من LLM -> معلومات منظمة (على سبيل المثال JSON).
- تحويل `ChatMessage` إلى سلسلة نصية فقط.
- تحويل المعلومات الإضافية التي تم إرجاعها من استدعاء بخلاف الرسالة (مثل استدعاء دالة OpenAI) إلى سلسلة نصية.

للحصول على معلومات كاملة حول هذا، راجع قسم [output parsers](/modules/model_io/output_parsers/output_parsers.md).

في دليل البدء هذا، سنكتب محلل مخرجات خاص بنا - يقوم بتحويل قائمة مفصولة بفواصل إلى قائمة.

```dart
class CommaSeparatedListOutputParser 
    extends BaseOutputParser<ChatResult, OutputParserOptions, List<String>> {
  
  const CommaSeparatedListOutputParser()
      : super(defaultOptions: const OutputParserOptions());

  @override
  Future<List<String>> invoke(
      final ChatResult input, {
      final OutputParserOptions? options,
  }) async {
    final message = input.output;
    return message.content.trim().split(',');
  }
}
```

```dart
final res = await const CommaSeparatedListOutputParser().invoke(
  const ChatResult(
    id: 'id',
    output: AIChatMessage(content: 'hi, bye'),
    finishReason: FinishReason.stop,
    metadata: {},
    usage: LanguageModelUsage(),
  ),
);
print(res);
// ['hi',  'bye']
```

## التجميع باستخدام LCEL (Composing with LCEL)

يمكننا الآن دمج كل هذه المكونات في سلسلة واحدة. ستأخذ هذه السلسلة متغيرات الإدخال (input variables)، وتمررها إلى قالب أمر (prompt template) لإنشاء prompt، وتمرر الـ prompt إلى نموذج لغة (language model)، ثم تمرر المخرجات عبر محلل مخرجات اختياري (output parser). هذه طريقة ملائمة لتجميع جزء معياري من المنطق. دعنا نراها عمليًا!

```dart
const systemTemplate = '''
You are a helpful assistant who generates comma separated lists.
A user will pass in a category, and you should generate 5 objects in that category in a comma separated list.
ONLY return a comma separated list, and nothing more.
''';
const humanTemplate = '{text}';

final chatPrompt = ChatPromptTemplate.fromTemplates([
  (ChatMessageType.system, systemTemplate),
  (ChatMessageType.human, humanTemplate),
]);

final chatModel = ChatOpenAI(apiKey: openAiApiKey);

final chain = chatPrompt.pipe(chatModel).pipe(CommaSeparatedListOutputParser());
// Alternative syntax:
// final chain = chatPrompt | chatModel | CommaSeparatedListOutputParser();

final res = await chain.invoke({'text': 'colors'});
print(res); // ['red', 'blue', 'green', 'yellow', 'orange']
```

لاحظ أننا نستخدم صيغة `.pipe` (أو بدلاً من ذلك صيغة `|`) لربط هذه المكونات معًا. يتم تشغيل هذه الصيغة بواسطة LangChain Expression Language (LCEL) وتعتمد على واجهة `Runnable` العالمية التي تنفذها جميع هذه الكائنات. لمعرفة المزيد حول هذه الصيغة، اقرأ الوثائق [هنا](/expression_language/expression_language.md).

## الخطوات التالية (Next steps)

هذا كل شيء! لقد قمنا الآن بتغطية كيفية إنشاء اللبنة الأساسية لتطبيقات LangChain. هناك العديد من الميزات الأخرى في كل من هذه المكونات الثلاثة أكثر مما يمكننا تغطيته هنا. لمواصلة رحلتك:

- اقرأ عن [LangChain Expression Language](/expression_language/expression_language.md) لمعرفة كيفية ربط هذه المكونات معًا.
- [تعمق أكثر](/modules/model_io/model_io.md) في LLMs، prompts، ومحللات المخرجات (output parsers) وتعلم [المكونات الرئيسية](/modules/modules.md) الأخرى.
- استكشف [حالات الاستخدام الشائعة الشاملة](https://python.langchain.com/docs/use_cases).
