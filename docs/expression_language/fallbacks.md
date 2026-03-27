# البدائل (Fallbacks)

عند العمل مع نماذج اللغة (language models)، قد تواجه غالبًا مشكلات من واجهات برمجة التطبيقات (APIs) الأساسية، مثل حدود المعدل (rate limits) أو فترات التوقف (downtime). لذلك، كلما نقلت تطبيقات نماذج اللغة الكبيرة (LLM applications) الخاصة بك إلى مرحلة الإنتاج، أصبح من المهم أكثر فأكثر أن تكون لديك خطط طوارئ للأخطاء. ولهذا السبب قدمنا مفهوم البدائل (fallbacks).

الأهم من ذلك، يمكن تطبيق البدائل ليس فقط على مستوى نموذج اللغة الكبير (LLM) ولكن على مستوى التشغيل بأكمله (runnable level). هذا مهم لأنه في كثير من الأحيان تتطلب النماذج المختلفة مطالبات (prompts) مختلفة. لذلك إذا فشلت مكالمتك إلى OpenAI، فأنت لا تريد فقط إرسال نفس المطالبة إلى Anthropic - فمن المحتمل أنك تريد استخدام قالب مطالبة (prompt template) مختلف على سبيل المثال.

## معالجة أخطاء واجهة برمجة تطبيقات نموذج اللغة الكبير (LLM API) باستخدام البدائل

ربما يكون هذا هو الاستخدام الأكثر شيوعًا للبدائل. يمكن أن يفشل طلب إلى واجهة برمجة تطبيقات نموذج اللغة الكبير (LLM API) لمجموعة متنوعة من الأسباب - قد تكون واجهة برمجة التطبيقات (API) معطلة، أو قد تكون قد وصلت إلى حد المعدل (rate limit)، أو أي عدد من الأشياء. يمكن التعامل مع هذا الموقف باستخدام البدائل (Fallbacks).

يمكن إنشاء البدائل باستخدام دالة `withFallbacks()` على التشغيل (runnable) الذي تعمل عليه، على سبيل المثال `final runnablWithFallbacks = mainRunnable.withFallbacks([fallback1, fallback2])` سيؤدي هذا إلى إنشاء `RunnableWithFallback` جنبًا إلى جنب مع قائمة من البدائل. عند استدعائه، سيتم استدعاء `mainRunnable` أولاً، إذا فشل فسيتم استدعاء البدائل بالتسلسل حتى يعيد أحد البدائل في القائمة مخرجًا (output). إذا نجح `mainRunnable` وأعاد مخرجًا، فلن يتم استدعاء البدائل.

## البدائل لنماذج الدردشة (chat models)

```dart
// fake model will throw error during invoke and fallback model will be called
final fakeOpenAIModel = ChatOpenAI(
    defaultOptions: const ChatOpenAIOptions(model: 'tomato'),
);

final latestModel = ChatOpenAI(
  defaultOptions: const ChatOpenAIOptions(model: 'gpt-4o'),
);

final modelWithFallbacks = fakeOpenAIModel.withFallbacks([latestModel]);

final prompt = PromptValue.string('Explain why sky is blue in 2 lines');

final res = await modelWithFallbacks.invoke(prompt);
print(res);
/*
{
  "ChatResult": {
    "id": "chatcmpl-9nKBcFNkzo5qUrdNB92b36J0d1meA",
    "output": {
      "AIChatMessage": {
        "content": "The sky appears blue because molecules in the Earth's atmosphere scatter shorter wavelength blue light from the sun more effectively than longer wavelengths like red. This scattering process is known as Rayleigh scattering.",
        "toolCalls": []
      }
    },
    "finishReason": "FinishReason.stop",
    "metadata": {
      "model": "gpt-4o-2024-05-13",
      "created": 1721542696,
      "system_fingerprint": "fp_400f27fa1f"
    },
    "usage": {
      "LanguageModelUsage": {
        "promptTokens": 16,
        "promptBillableCharacters": null,
        "responseTokens": 36,
        "responseBillableCharacters": null,
        "totalTokens": 52
      }
    },
    "streaming": false
  }
}
*/
```

ملاحظة: إذا كانت الخيارات المقدمة عند استدعاء التشغيل (runnable) مع البدائل (fallbacks) غير متوافقة مع بعض البدائل، فسيتم تجاهلها. إذا كنت ترغب في استخدام خيارات مختلفة للبدائل المختلفة، فقدمها كـ `defaultOptions` عند إنشاء البدائل أو استخدم `bind()`.

## البدائل لتسلسلات التشغيل (RunnableSequences) مع الدُفعات (batch)

```dart
final fakeOpenAIModel = ChatOpenAI(
    defaultOptions: const ChatOpenAIOptions(model: 'tomato'),
);

final latestModel = ChatOpenAI(
  defaultOptions: const ChatOpenAIOptions(model: 'gpt-4o'),
);

final promptTemplate = ChatPromptTemplate.fromTemplate('tell me a joke about {topic}');

final badChain = promptTemplate.pipe(fakeOpenAIModel);
final goodChain = promptTemplate.pipe(latestModel);

final chainWithFallbacks = badChain.withFallbacks([goodChain]);

final res = await chainWithFallbacks.batch(
  [
    {'topic': 'bears'},
    {'topic': 'cats'},
  ],
);
print(res);
/*
[
  {
    "id": "chatcmpl-9nKncT4IpAxbUxrEqEKGB0XUeyGRI",
    "output": {
      "content": "Sure! How about this one?\n\nWhy did the bear bring a suitcase to the forest?\n\nBecause it wanted to pack a lunch! 🐻🌲",
      "toolCalls": []
    },
    "finishReason": "FinishReason.stop",
    "metadata": {
      "model": "gpt-4o-2024-05-13",
      "created": 1721545052,
      "system_fingerprint": "fp_400f27fa1f"
    },
    "usage": {
      "promptTokens": 13,
      "promptBillableCharacters": null,
      "responseTokens": 31,
      "responseBillableCharacters": null,
      "totalTokens": 44
    },
    "streaming": false
  },
  {
    "id": "chatcmpl-9nKnc58FpXFTPkzZfm2hHxJ5VSQQh",
    "output": {
      "content": "Sure, here's a cat joke for you:\n\nWhy was the cat sitting on the computer?\n\nBecause it wanted to keep an eye on the mouse!",
      "toolCalls": []
    },
    "finishReason": "FinishReason.stop",
    "metadata": {
      "model": "gpt-4o-2024-05-13",
      "created": 1721545052,
      "system_fingerprint": "fp_c4e5b6fa31"
    },
    "usage": {
      "promptTokens": 13,
      "promptBillableCharacters": null,
      "responseTokens": 29,
      "responseBillableCharacters": null,
      "totalTokens": 42
    },
    "streaming": false
  }
]
*/
```
