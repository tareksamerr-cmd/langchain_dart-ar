# أدوات (Tools)

الأدوات قابلة للتنفيذ أيضاً، وبالتالي يمكن استخدامها ضمن سلسلة من العمليات:

```dart
final openaiApiKey = Platform.environment['OPENAI_API_KEY'];
final model = ChatOpenAI(apiKey: openaiApiKey);
const stringOutputParser = StringOutputParser<ChatResult>();

final promptTemplate = ChatPromptTemplate.fromTemplate('''
Turn the following user input into a math expression for a calculator. 
Output only the math expression. Let's think step by step.

INPUT:
{input}

MATH EXPRESSION:''');

final chain = Runnable.getMapFromInput() |
    promptTemplate |
    model |
    stringOutputParser |
    Runnable.getMapFromInput() |
    CalculatorTool();

final res = await chain.invoke(
  'If I had 3 apples and you had 5 apples but we ate 3. '
  'If we cut the remaining apples in half, how many pieces would we have?',
  options: const ChatOpenAIOptions(temperature: 0),
);
print(res);
// 10.0
```