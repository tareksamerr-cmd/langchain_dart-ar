# RunnableRetry: إعادة محاولة Runnables

`RunnableRetry` يغلف `Runnable` ويعيد محاولته إذا فشل. يمكن إنشاؤه باستخدام `runnable.withRetry()`.

بشكل افتراضي، ستتم إعادة محاولة `runnable` 3 مرات باستراتيجية التراجع الأسي (exponential backoff).

## الاستخدام

## إنشاء RunnableRetry

```dart
final model = ChatOpenAI();
final input = PromptValue.string('Explain why sky is blue in 2 lines');

final modelWithRetry = model.withRetry();
final res = await modelWithRetry.invoke(input);
print(res);
```

## إعادة محاولة سلسلة (Retrying a chain)

يمكن استخدام `RunnableRetry` لإعادة محاولة أي `Runnable`، بما في ذلك سلسلة من `Runnable`s.

مثال

```dart
final promptTemplate = ChatPromptTemplate.fromTemplate('tell me a joke about {topic}');
final model = ChatOpenAI(
  defaultOptions: ChatOpenAIOptions(model: 'gpt-4o'),
);
final chain = promptTemplate.pipe(model).withRetry();

final res = await chain.batch(
  [
    {'topic': 'bears'},
    {'topic': 'cats'},
  ],
);
print(res);
```

> بشكل عام، من الأفضل إبقاء نطاق إعادة المحاولة (retry) صغيرًا قدر الإمكان.

## تكوين إعادة المحاولة (Configuring the retry)

```dart
// passing a fake model to cause Exception
final input = PromptValue.string('Explain why sky is blue in 2 lines');
final model = ChatOpenAI(
  defaultOptions: ChatOpenAIOptions(model: 'fake-model'),
);
final modelWithRetry = model.withRetry(
    maxRetries: 3,
    addJitter: true,
);
final res = await modelWithRetry.invoke(input);
print(res);
// retried 3 times and returned Exception:
// OpenAIClientException({
//   "uri": "https://api.openai.com/v1/chat/completions",
//   "method": "POST",
//   "code": 404,
//   "message": "Unsuccessful response",
//   "body": {
//     "error": {
//       "message": "The model `fake-model` does not exist or you do not have access to it.",
//       "type": "invalid_request_error",
//       "param": null,
//       "code": "model_not_found"
//     }
//   }
// }) 
```

## تمرير فترات التأخير (Passing delay durations)

إذا كنت ترغب في استخدام فترات تأخير مخصصة لكل محاولة إعادة، يمكنك تمرير قائمة من كائنات `Duration` إلى معلمة `delayDurations`.

```dart
final input = PromptValue.string('Explain why sky is blue in 2 lines');
final model = ChatOpenAI(
  defaultOptions: ChatOpenAIOptions(model: 'fake-model'),
);
final modelWithRetry = model.withRetry(
    maxRetries: 3,
    delayDurations: [
      Duration(seconds: 1),
      Duration(seconds: 2),
      Duration(seconds: 3),
    ],
);
final res = await modelWithRetry.invoke(input);
print(res);
```
