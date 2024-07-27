---
layout: post
title:  "LLM Evaluation with RAGAS"
date:   2024-07-27 22:00:00 +0300
categories: blog
---

Embarking on the journey of evaluating your RAG LLM application can be both exciting and daunting. It's a process that not only tests the capabilities of the model but also reflects your own understanding of its intricate workings. In this blog post, we'll dive into the key aspects of evaluation, from setting up the right metrics to interpreting the results for continuous improvement. Whether you're a seasoned data scientist or a curious enthusiast, this guide aims to shed light on the nuances of machine learning evaluation and its significance in the ever-evolving field of AI.

![Bot Evaluation]({{site.url}}/images/bot-evaluation.jpeg)

## What is LLM Evaluation?
LLM evaluation is a process that assesses the performance, reliability, and effectiveness of Large Language Models (LLMs). It's crucial for ensuring that these models produce results that are ethical, safe, and aligned with human values. Evaluations can range from checking the accuracy of specific tasks to measuring overall behavior and biases. Essentially, it's about making sure that LLMs are doing their jobs correctly and responsibly.

## Tools: RAGAS
In this blog post we will make use of [RAGAS](https://ragas.io/)

RAGAS is a cutting-edge Python library designed to evaluate and improve the performance of Retrieval Augmented Generation (RAG) pipelines. It's a dedicated solution for monitoring and enhancing LLM applications that utilize external data to augment their context. With RAGAS, developers can access tools based on the latest research to assess the quality of LLM-generated text, providing valuable insights into their RAG pipeline's performance. It can also be integrated into CI/CD workflows for continuous performance checks, ensuring that your LLM applications run optimally in production environments.

## How to evaluate your LLM application

### Step 1: Gather evaluation data
It all start with generating a list of questions or prompts + the ground truth.
For this example I just created a CSV with a list of questions that I want to check every time I run the evaluation.

| question                                                                                     | ground_truth             |
| -------------------------------------------------------------------------------------------- | ------------------------ |
| Which animal is known as the "king of the savanna"?                                          | Lion                     |
| What is the tallest animal in the savanna?                                                   | Giraffe                  |
| Which large mammal in the savanna is known for its distinctive horn-like tusks?              | African elephant         |
| This spotted carnivore is known for its speed and agility in the savanna. What animal is it? | Cheetah                  |
| Which bird is known for its large, distinctive bill and can often be seen in the savanna?    | Southern ground hornbill |

### Step 2: Gather newly generated answers
This step is hard to give you some example code since it will all be based on your own process.
But the idea is that you push your question or prompt through your LLM application to generate an answer. In my case I have a specific function that will send my question to an API and return the answer and the used context. (Notice I am also returning the intent and the conversation id, but thats not needed for the evaluation)

```python
ragas_data = list()

for i in range(len(data)):
    try:
        question = data.loc[i, "question"]
        ground_truth = data.loc[i, "ground_truth"]
        answer, context, intent, conversationId = get_answer_api(question)
        ragas_data.append({
            "question" :question,
            "ground_truth" : ground_truth,
            "answer" : answer,
            "contexts" : context,
            "intent" : intent
        })
        print(f"{i+1}/{len(data)} - {question}")
    except Exception as e:
        print(f"Error at index {i} - {e}")

ragas_data_df = pd.DataFrame(ragas_data)
```

Notice that all of this is using standard Python and not any specific library.
The idea is that in step 2 we have a dataframe with our question, our ground truth, or generated answer and our context.

*The context is needed if you want to evaluate if the used context is actually relevant and if the answer is available in the context.*

### Step 3: Initiate the evaluation models
You need to understand that LLM evaluation is done by LLMs.
Weird right. But the idea is that you or what has been build in into the library compare the ground truth and the generated answer with a PROMPT.
This is mostly done with an LLM that is better than the one you have already been using in your application.

For example in your LLM application you make use of gpt-35-turbo or maybe a mini model.
Then you will do your evaluation with a bigger model like gpt-4o

Therefore we need to initiate the models that we want to make use. For this we can make use of the langchain chat models + the embedding models since we also need those.

```python
azure_model = AzureChatOpenAI(
    openai_api_version = os.environ.get("AZURE_OPENAI_VERSION"),
    azure_endpoint = os.environ.get("AZURE_OPENAI_ENDPOINT"),
    azure_deployment = os.environ.get("AZURE_OPENAI_CHAT_DEPLOYMENT"),
    model = os.environ.get("AZURE_OPENAI_CHAT_MODEL"),
    api_key = os.environ.get("AZURE_OPENAI_KEY")
)

azure_embeddings = AzureOpenAIEmbeddings(
    openai_api_version = os.environ.get("AZURE_OPENAI_VERSION"),
    azure_endpoint = os.environ.get("AZURE_OPENAI_ENDPOINT"),
    azure_deployment = os.environ.get("AZURE_OPENAI_EMBEDDING_DEPLOYMENT"),
    model = os.environ.get("AZURE_OPENAI_EMBEDDING_MODEL"),
    api_key = os.environ.get("AZURE_OPENAI_KEY")
)
```

### Step 4: Initiate the dataset
Depending on the Evaluation library this might look different. This is specific for the RAGAS library. The Ragas library needs a dataset existing out of 4 elements each containing a list of all the items.

```python
ragas_dataset = {
    "question" : ragas_data_df["question"].to_list(),
    "ground_truth" : ragas_data_df["ground_truth"].to_list(),
    "answer" : ragas_data_df["answer"].to_list(),
    "contexts" : ragas_data_df["contexts"].to_list(),
}
ds = datasets.Dataset.from_dict(ragas_dataset)
```

### Step 5: Initiate the metrics + execute the evaluation
Evaluation libraries like RAGAS offer a list of standard metrics.

![RAGAS Metrics]({{site.url}}/images/ragas-score.png)

RAGAS has the following, I am not going to describe what they do because all that is nicely explained on their own website. (See links)

- [Faithfulness](https://docs.ragas.io/en/stable/concepts/metrics/faithfulness.html)
- [Answer Relevance](https://docs.ragas.io/en/stable/concepts/metrics/answer_relevance.html)
- [Context Precision](https://docs.ragas.io/en/stable/concepts/metrics/context_precision.html)
- [Context Recall](https://docs.ragas.io/en/stable/concepts/metrics/context_recall.html)
- [Context entities recall](https://docs.ragas.io/en/stable/concepts/metrics/context_entities_recall.html)
- [Answer semantic similarity](https://docs.ragas.io/en/stable/concepts/metrics/semantic_similarity.html#)
- [Answer Correctness](https://docs.ragas.io/en/stable/concepts/metrics/answer_correctness.html)
- [Aspect Critique](https://docs.ragas.io/en/stable/concepts/metrics/critique.html)
- [Summarization Score](https://docs.ragas.io/en/stable/concepts/metrics/summarization_score.html)

The next step is to make a selection of what metrics you want to evaluate and bring our dataset, our selection of metrics and our llm configuration all together

```python
result = ragas.evaluate(
    dataset= ds,
    metrics=[metrics.faithfulness, 
             metrics.answer_correctness, 
             metrics.context_recall, 
             metrics.context_precision],
    llm=azure_model,
    embeddings=azure_embeddings,
    raise_exceptions=False
)
```

When you run this in a Jupyter Notebook you will notice it will show the progress like this
```
Evaluating:  10%|█         | 2/20 [01:05<11:29, 38.32s/it]
```
Once this is done you can have a look at the general results by printing the output
```python
print(result)
```
**Results:**
```json
{'faithfulness': 0.9750, 'answer_correctness': 0.6349, 'context_recall': 0.7000, 'context_precision': 0.8011}
```
Although this is great for reporting. But in case you want to go more in details and want to figure out where exactly your problems are you can get the detailed score per item by converting the result.scores into a dataframe

```python
pd.DataFrame(result.scores)
```
**Output:**

| faithfulness | answer_correctness | context_recall | context_precision |
| ------------ | ------------------ | -------------- | ----------------- |
| 1.000        | 1.000000           | 1.0            | 1.000000          |
| 0.875        | 0.235431           | 0.5            | 1.000000          |
| 1.000        | 0.731198           | 0.0            | 0.416667          |
| 1.000        | 0.589449           | 1.0            | 0.833333          |
| 1.000        | 0.618378           | 1.0            | 0.755556          |

### Final Step
Yes you might have noticed that scores without any other info is kinda worthless. Isn't it?
So I personally like to link it all together in a dataframe. It is up to you if you want to export it to a database our a file (csv, excel,...) to keep track of them for analytics.

```python
merged_df = pd.merge(
    ragas_data_df, 
    pd.DataFrame(result.scores), 
    left_index=True, 
    right_index=True)
```

## My honest feedback on LLM Evaluation
- Its a great way to evaluate if your LLM application is stable. 
- Its a great way to evaluate other models to what you are using right now. Since you update your code and generate your answer again, and then run it through the evaluation.
- Its a great way to evaluate where your bot is not doing well, (context vs llm capabilities)

**But I do not like using it to generate some kind of accuracy score to tell your customer or manager that you have a score of x%. Because you might get a score on a particular question of 71%. But if you check on it manually, you might actually say its perfectly correct. And the only reason the evaluation tool said 71% is because the wordings were not the same.**

So it has its positives, but try it out and see where you want to use it.
But as you noticed the code is quite easy!