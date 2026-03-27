# واجهة Runnable (Runnable interface)

لتسهيل إنشاء سلاسل مخصصة قدر الإمكان، توفر LangChain واجهة `Runnable` التي تنفذها معظم المكونات، بما في ذلك نماذج الدردشة (chat models)، ونماذج اللغة الكبيرة (LLMs)، ومحللات المخرجات (output parsers)، والمسترجعات (retrievers)، وقوالب المطالبات (prompt templates)، والمزيد.

هذه واجهة قياسية، مما يسهل تحديد السلاسل المخصصة بالإضافة إلى استدعائها بطريقة قياسية. تتضمن الواجهة القياسية ما يلي:

- `invoke`: استدعاء السلسلة على إدخال وإرجاع المخرجات.
- `stream`: استدعاء السلسلة على إدخال وبث المخرجات.
- `batch`: استدعاء السلسلة على قائمة من المدخلات وإرجاع قائمة من المخرجات.

يختلف نوع الإدخال والإخراج حسب المكون:

| المكون                      | نوع الإدخال             | نوع الإخراج            |
|-----------------------------|------------------------|------------------------|
| `PromptTemplate`            | `Map<String, dynamic>` | `PromptValue`          |
| `ChatMessagePromptTemplate` | `Map<String, dynamic>` | `PromptValue`          |
| `LLM`                       | `PromptValue`          | `LLMResult`            |
| `ChatModel`                 | `PromptValue`          | `ChatResult`           |
| `OutputParser`              | أي كائن                | نوع مخرجات المحلل      |
| `Retriever`                 | `String`               | `List<Document>`       |
| `DocumentTransformer`       | `List<Document>`       | `List<Document>`       |
| `Tool`                      | `Map<String, dynamic>` | `String`               |
| `Chain`                     | `Map<String, dynamic>` | `Map<String, dynamic>` |

هناك أيضًا العديد من البدائيات المفيدة للعمل مع `runnables`، والتي يمكنك قراءتها في [هذا القسم](/expression_language/primitives.md).

## واجهة Runnable (Runnable interface)

دعنا نلقي نظرة على هذه الطرق! للقيام بذلك، سنقوم بإنشاء سلسلة `PromptTemplate` + `ChatModel` بسيطة للغاية.

```dart
final model = ChatOpenAI(apiKey: openaiApiKey);

final promptTemplate = ChatPromptTemplate.fromTemplate(
  'Tell me a joke about {topic}',
);

final chain = promptTemplate.pipe(model).pipe(StringOutputParser());
```

في هذا المثال، نستخدم طريقة `pipe` لدمج `runnables` في تسلسل. يمكنك قراءة المزيد عن هذا في قسم [RunnableSequence: Chaining runnables](/expression_language/primitives/sequence.md).

### Invoke

تأخذ طريقة `invoke` إدخالًا وتعيد مخرجات استدعاء السلسلة على هذا الإدخال.

```dart
final res = await chain.invoke({'topic': 'bears'});
print(res);
// Why don't bears wear shoes? Because they have bear feet!
```

### Stream

تأخذ طريقة `stream` إدخالًا وتبث أجزاء من المخرجات.

```dart
final stream = chain.stream({'topic': 'bears'});
int count = 0;
await for (final res in stream) {
  print('$count: $res');
  count++;
}
// 0:
// 1: Why
// 2:  don
// 3: 't
// 4:  bears
// 5:  like
// 6:  fast
// 7:  food
// 8: ?
// 9: Because
// 10:  they
// 11:  can
// 12: 't
// 13:  catch
// 14:  it
// 15: !
```

### Batch

تقوم بتجميع استدعاء `Runnable` على `inputs` المعطاة.

```dart
final res = await chain.batch([
  {'topic': 'bears'},
  {'topic': 'cats'},
]);
print(res);
//['Why did the bear break up with his girlfriend? Because she was too "grizzly" for him!',
// 'Why was the cat sitting on the computer? Because it wanted to keep an eye on the mouse!']
```

إذا كان المزود الأساسي يدعم التجميع (batching)، فستحاول هذه الطريقة تجميع الاستدعاءات إلى المزود. وإلا، فستقوم فقط باستدعاء `invoke` على كل إدخال بشكل متزامن. يمكنك تكوين حد التزامن عن طريق تعيين حقل `concurrencyLimit` في معلمة `options`.

يمكنك توفير خيارات الاستدعاء لطريقة `batch` باستخدام معلمة `options`. يمكن أن تكون:
- `null`: يتم استخدام الخيارات الافتراضية.
- قائمة بعنصر واحد: يتم استخدام نفس الخيارات لجميع المدخلات.
- قائمة بنفس طول المدخلات: يحصل كل إدخال على خياراته الخاصة.

```dart
final res = await chain.batch(
  [
    {'topic': 'bears'},
    {'topic': 'cats'},
  ],
  options: [
    const ChatOpenAIOptions(model: 'gpt-4o-mini', temperature: 0.5),
    const ChatOpenAIOptions(model: 'gpt-4', temperature: 0.7),
  ],
);
print(res);
//['Why did the bear break up with his girlfriend? Because he couldn't bear the relationship anymore!',
// 'Why don't cats play poker in the jungle? Because there's too many cheetahs!']
```
