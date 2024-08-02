---
layout: post
title:  "LLM Evaluation with Prompt Flow SDK"
date:   2024-08-02 07:00:00 +0300
categories: blog
---

In my previous [blogpost](https://www.datasavanna.co.ke/blog/2024/07/27/llm-evaluation-with-ragas.html) I discussed how you could do LLM evaluation with [RAGAS](https://www.ragas.io). In this one we will keep the structure and try to do exactly the same but with Azure PromptFlow SDK

![Bot Evaluation]({{site.url}}/images/bot-evaluation.jpeg)

## Tools: Prompt Flow SDK

Documentation: [Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-studio/how-to/develop/flow-evaluate-sdk)

The Prompt Flow SDK is a toolset provided by Azure AI Studio that allows developers to evaluate the performance of their generative AI applications. It includes built-in evaluators for various scenarios like question and answer, chat, and content safety, as well as the ability to create custom evaluators. These evaluators help in assessing the quality and safety of the AI's outputs using quantitative metrics. For a thorough evaluation, the SDK can process a single data row or a larger test dataset, providing insights into the application's capabilities and limitations.

## How to evaluate your LLM application
You will notice if you have read the post about LLM evaluation with RAGAS that some steps are identically the same

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

Therefore we need to initiate the models that we want to make use. For this we can make use of the langchain chat models.

```python
from promptflow.core import AzureOpenAIModelConfiguration

model_config = AzureOpenAIModelConfiguration(
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_deployment="gpt-4o",
    api_version="2024-06-01",
)
```

### Step 4: Initiate the dataset
Where RAGAS could work with a python object (Dataset), Prompt Flow sadly works with jsonl files. This is kind of a pitty, and I think a negative point towards the SDK. But it is what it is. Following code will convert your dataframe into a jsonl file.

```python
json_rows = ragas_data_df.to_json(orient="records", lines=True).splitlines()

with open("data.jsonl", "w") as f:
    for row in json_rows:
        f.write(row + "\n")
```

### Step 5: Initiate the metrics
Evaluation libraries like Prompt Flow offer a list of standard metrics.

![Prompt Flow Metrics]({{site.url}}/images/promptflow-metrics.png)

Prompt Flow SDK has the following, I am not going to describe what they do because all that is explained in the documentation. ([link](https://learn.microsoft.com/en-us/azure/ai-studio/how-to/develop/flow-evaluate-sdk#built-in-evaluators))

You notice that Prompt Flow has **Composite** evaluators, this means its a mix of different metrics combined. Based on what kind of application your are evaluating you can select a specific Composite evaluator.

Each evaluator needs to be configured with the llm that you defined earlier.
You can combine multiple evaluators. As I did in below example

```python
from promptflow.evals.evaluators import GroundednessEvaluator, QAEvaluator

groundedness_eval = GroundednessEvaluator(model_config=model_config)
qa_eval = QAEvaluator(model_config=model_config)

evaluators = {
    "qa_eval": qa_eval,
    "groundedness_eval": groundedness_eval
}
```

You can evaluate a single datarow by calling the evaluator immidatly
```python
qa_eval(question= "What is the capital of France?",
        context= "France is in Europe",
        answer= "Paris is the capital of France.",
        ground_truth="Paris")
```

Output:
```json
{'gpt_groundedness': 5.0}
```

### Step 6: Execute the evaluation
Now we can bring everything togeter and run our evaluation

```python
from promptflow.evals.evaluate import evaluate

result = evaluate(
    data="data.jsonl",
    evaluators= evaluators,
)
```

Notice that when running this script it generates a lot of information, am not going into detail about this. But 1 thing it prints out is an url

```
http://127.0.0.1:23333/v1.0/ui/traces/?#run=promptflow_evals_evaluators_groundedness_groundedness_groundednessevaluator_0q_1osdf_20240802_104218_775676
```

This links to an dashboard where you can find back all your evaluations

![Prompt Flow Dashboard]({{site.url}}/images/promptflow-dashboard.png)

This is pretty cool, but it means all that info is saved somewhere. All results are saved into your AppData/Local/Temp (atleast on Windows)

![Prompt Flow Folders]({{site.url}}/images/promptflow-folders.png)

So be carefull where you run this code, so you can implement an auto cleanup.

**AN IMPORTANT NOTE (BUG)** before we go into the results:

Some evaluators change your current working directory, which means that out of nothing you might get an error that says that your jsonl file is not valid. But the issue is actually that it is looking in the wrong folder. 

### Results

The general results can be viewed by calling the *metrics* key that is returned by the evaluation function.

```python
result["metrics"]
```

**Output:**

Output will be different based on what metrics you have chosen and the name you given to the evaluators
```json
{'qa_eval.f1_score': 0.33333333330000003,
 'qa_eval.gpt_fluency': 5.0,
 'qa_eval.gpt_relevance': 5.0,
 'qa_eval.gpt_coherence': 5.0,
 'qa_eval.gpt_similarity': 5.0,
 'qa_eval.gpt_groundedness': 5.0,
 'groundedness_eval.gpt_groundedness': 5.0}
```

The detailed results can be viewed by calling the *rows* key. I do convert it into a dataframe to make it easier readable and to be able to export it to a CSV or Excel.

```python
pd.DataFrame(result["rows"])
```

**Output:**
![Prompt Flow Output]({{site.url}}/images/promptflow-output.png)


## My feedback
I did not like working with this library, compared with RAGAS its complicated.
It still has bugs, and that I need to convert my data into a file before I can use it was a serious downpoint to me.

I did give my feedback to Microsoft and they said they are working on fixing this bugs. So I am looking forward to test this SDK again when a new release comes out!



