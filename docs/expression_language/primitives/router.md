# توجيه المنطق ديناميكيًا بناءً على الإدخال

توضح هذه المفكرة كيفية إجراء التوجيه (routing) في لغة تعبير LangChain (LangChain Expression Language).

يسمح لك التوجيه بإنشاء سلاسل غير حتمية حيث يحدد إخراج خطوة سابقة الخطوة التالية. يساعد التوجيه في توفير بنية واتساق حول التفاعلات مع نماذج اللغة الكبيرة (LLMs).

## استخدام RunnableRouter

سنوضح كيفية إجراء التوجيه باستخدام تسلسل من خطوتين حيث تصنف الخطوة الأولى سؤال الإدخال على أنه يتعلق بـ LangChain أو Anthropic أو غير ذلك، ثم توجه إلى سلسلة موجه (prompt chain) مقابلة.

أولاً، دعنا ننشئ سلسلة تحدد الأسئلة الواردة على أنها تتعلق بـ `LangChain` أو `Anthropic` أو `Other`:

```dart
final chatModel = ChatOllama(
  defaultOptions: ChatOllamaOptions(model: 'llama3.2'),
);

final classificationChain = PromptTemplate.fromTemplate('''
Given the user question below, classify it as either being about `LangChain`, `Anthropic`, or `Other`.

Do not respond with more than one word.

<question>
{question}
</question>

Classification:
''') | chatModel | StringOutputParser();

final res1 = await classificationChain.invoke({
  'question': 'how do I call Anthropic?',
});
print(res1);
// Anthropic
```

الآن، دعنا ننشئ ثلاث سلاسل فرعية:

```dart
final langchainChain = PromptTemplate.fromTemplate('''
You are an expert in langchain.
Always answer questions starting with "As Harrison Chase told me".
Respond to the following question:

Question: {question}
Answer:
''') | chatModel | StringOutputParser();

final anthropicChain = PromptTemplate.fromTemplate('''
You are an expert in anthropic.
Always answer questions starting with "As Dario Amodei told me".
Respond to the following question:

Question: {question}
Answer:
''') | chatModel | StringOutputParser();

  final generalChain = PromptTemplate.fromTemplate('''
Respond to the following question:

Question: {question}
Answer:
  ''') | chatModel | StringOutputParser();
```

`RunnableRouter` هو نوع من "runnable" يأخذ دالة توجه الإدخال إلى `Runnable` معين. يمكنك استخدام `Runnable.fromRouter` لإنشاء `RunnableRouter`.

في هذا المثال، سنقوم بإرجاع إحدى السلاسل الثلاث التي عرفناها سابقًا بناءً على الموضوع الذي تم إرجاعه بواسطة سلسلة التصنيف.

```dart
final router = Runnable.fromRouter((Map<String, dynamic> input, _) {
  final topic = (input['topic'] as String).toLowerCase();
  if (topic.contains('langchain')) {
    return langchainChain;
  } else if (topic.contains('anthropic')) {
    return anthropicChain;
  } else {
    return generalChain;
  }
});

final fullChain = Runnable.fromMap({
      'topic': classificationChain,
      'question': Runnable.getItemFromMap('question'),
    }) | router;

final res2 = await fullChain.invoke({
  'question': 'how do I use Anthropic?',
});
print(res2);
// As Dario Amodei told me, using Anthropic is a straightforward process that...

final res3 = await fullChain.invoke({
  'question': 'how do I use LangChain?',
});
print(res3);
// As Harrison Chase told me, using LangChain is a breeze!

final res4 = await fullChain.invoke({
  'question': 'whats 2 + 2',
});
print(res4);
// The answer is... 4!
```

## التوجيه بالتشابه الدلالي (semantic similarity)

إحدى التقنيات المفيدة بشكل خاص هي استخدام التضمينات (embeddings) لتوجيه الاستعلام إلى الموجه الأكثر صلة.

إليك مثال حيث لدينا موجهان متخصصان، أحدهما للفيزياء والآخر للتاريخ. سنستخدم التضمينات لتحديد أي موجه هو الأنسب للإجابة على سؤال معين.

```dart
const physicsTemplate = '''
You are a very smart physicist.
You are great at answering questions about physics (e.g. black holes, quantum mechanics, etc.)
in a concise and easy to understand manner.
When you don't know the answer to a question you admit that you don't know.

Here is a question:
{query}
''';

const historyTemplate = '''
You are a very good historian.
You are great at answering history questions (e.g. about the Roman Empire, World War II, etc.) 
in a detailed and engaging manner. 
You are able to provide a lot of context and background information.


Here is a question:
{query}
''';

final embeddings = OllamaEmbeddings(model: 'llama3.2');
final promptTemplates = [physicsTemplate, historyTemplate];
final promptEmbeddings = await embeddings.embedDocuments(
  promptTemplates.map((final pt) => Document(pageContent: pt)).toList(),
);

final chain = Runnable.fromMap<String>({'query': Runnable.passthrough()}) |
    Runnable.fromRouter((input, _) async {
      final query = input['query'] as String;
      final queryEmbedding = await embeddings.embedQuery(query);
      final mostSimilarIndex = getIndexesMostSimilarEmbeddings(queryEmbedding, promptEmbeddings).first;
      print('Using ${mostSimilarIndex == 0 ? 'Physicist' : 'Historian'}');
      return PromptTemplate.fromTemplate(promptTemplates[mostSimilarIndex]);
    }) |
    ChatOllama(
      defaultOptions: const ChatOllamaOptions(model: 'llama3.2'),
    ) |
    StringOutputParser();

final res1 = await chain.invoke("What's a black hole?");
print(res1);
// Using Physicist
// Black holes! One of my favorite topics!
// A black hole is a region in space where the gravitational pull is so strong...

final res2 = await chain.invoke('When did World War II end?');
print(res2);
// Using Historian
// A great question to start with! World War II ended on September 2, 1945...
```
