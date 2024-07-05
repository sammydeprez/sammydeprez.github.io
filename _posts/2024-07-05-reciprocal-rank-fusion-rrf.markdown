---
layout: post
title:  "Reciprocal Rank Fusion (RRF), Euhm what?"
date:   2024-07-05 07:00:00 +0300
categories: blog
---
Yes you heard me well, I said *Reciprocal Rank Fusion*. It is a mouth full, but such an important method to improve your search results when using RAG. In below article I will explain what it is and how you can make use of it when using Azure AI Search.

**PS:** In Azure terms this is also called Hybrid Search, so when you combine Vector with Semantic this is also RRF

## So lets start with what is it

RRF is an algoritm that combines multiple search queries, and then reranks the results by their combined scoring.

Example: You search for the query *"What is Reciprocal Rank Fusion"*

The results you get might be good, but if you think about it this query does not have all the details that you might want to know.

So you rewrite your query to: *"What are the key principles behind reciprocal rank fusion"*

You get new results, some might be the same and others might be new and more relevant.

Imagine you can bring those results together and give you result page with the best results of both queries!

### Simplistic Process Flow
![RRF Process Flow](../images/rrf.png)

Well that is what RRF is.

## Why would I use this

When writing Question Answer based chatbots with LLM technology, you might notice that what the user is asking not always fits to the content that you have, they might use words that are not available in your documents. And although Vector query already catches a lot of those issues. Its not perfect.

So to collect the top context to prompt your LLM with a question. RRF is at the moment one of the best ways to do.

## How can we do this
I'll be sharing here a couple of code statements in C# that can assist you collecting RRF search results

### Step 1 - Generate multiple search queries
First off we need to generate multiple versions of the initial question. Guess what we will make use of an LLM to do this. By making use of this example prompt (finetuning might be needed, based on your needs)

{% highlight ruby %}
var searchTerm = "what is reciprocal rank fusion";

var response = chatClient.CompleteChat(
    messages: [
        new UserChatMessage($@"You are an AI language model assistant. Your task is to generate five 
different versions of the given user question to retrieve relevant documents from a vector 
database. By generating multiple perspectives on the user question, your goal is to help
the user overcome some of the limitations of the distance-based similarity search. 
Provide these alternative questions separated by pipeline, do not add numbers. Original question: {searchTerm}")
    ]
);

var new_queries = response.Value.Content[0].Text.Split("|").ToList();
{% endhighlight %}

**Result**

{% highlight ruby %}
[ 
    What does reciprocal rank fusion mean? ,  
    Can you explain the concept of reciprocal rank fusion? ,  
    How does reciprocal rank fusion work in information retrieval? ,  
    What are the key principles behind reciprocal rank fusion? ,  
    Could you provide an overview of reciprocal rank fusion? 
]
{% endhighlight %}

### Step 2 - Vectorize each search term
Now we need to embed all the search queries to be able to use them in a vector search query

{% highlight ruby %}
var new_queries_embeddings = new List<System.ReadOnlyMemory<float>>();
foreach(var q in new_queries){
    new_queries_embeddings.Add(embeddingClient.GenerateEmbedding(q).Value.Vector);
}
new_queries_embeddings
{% endhighlight %}

**Result**
{% highlight ruby %}
[
    [ -0.001081995, 0.020376638, 0.028344905, 0.006761067, .... ],
    [ -0.0150158815, 0.026469436, 0.0258682, 0.0042462326, .... ],
    [ -0.0042913854, 0.017329674, 0.057391725, -0.0057172882, .... ],
    [ -0.024362233, 0.012113107, 0.041681785, 0.020342162, .... ],
    [ -0.02241878, 0.0040840413, 0.013294858, -0.017676823, .... ]
]
{% endhighlight %}

### Step 3 (final) - Get your results
In our final step we need to make use of all the above calculated embeddings and send them off to a vector database, in this case we will make use of Azure AI Search.

We add the different embeddings as all different VectorizedQueries to the search request.

{% highlight ruby %}
var vectorSearch = new VectorSearchOptions();
var top = 5; // number of top results to return in the vector search
var vector_field = "vector"; // this is the name of the field in the index that contains the vector

foreach(var q in new_queries_embeddings){
    vectorSearch.Queries.Add(new VectorizedQuery(q) { KNearestNeighborsCount = top, Fields = { "vector" }});
}

var results = await searchClient.SearchAsync<SearchDocument>(
    searchText: searchTerm,
    new SearchOptions {
        QueryType = SearchQueryType.Semantic,
        IncludeTotalCount = true,
        VectorSearch = vectorSearch
    }
);
{% endhighlight %}

And thats it, just to say it one more time *'Reciprocal Rank Fusion'*, it sounds very complicated, but at the end its very easy to implement. 

## Extra
Now the cool thing is if you make use of the Azure AI Search (Bring Your own Data), then there is no need to do all these single statements but it does it all automatically for you.

An example snippet:

{% highlight ruby %}
ChatCompletionOptions options = new ChatCompletionOptions();
options.AddDataSource(new AzureSearchChatDataSource()
{
    Endpoint = new Uri("YOUR_SEARCH_ENDPOINT"),
    IndexName = "YOUR_INDEX_NAME",
    InScope = false,
    VectorizationSource = DataSourceVectorizer.FromDeploymentName(""YOUR_EMBEDDING_DEPLOYMENT_NAME"),
    QueryType = DataSourceQueryType.Vector,
    Authentication = DataSourceAuthentication.FromApiKey("YOUR_SEARCH_API_KEY"),
    FieldMappings = new DataSourceFieldMappings()
    { //This will need to updated based on your index
        TitleFieldName = "title",
        UrlFieldName = "url",
        FilepathFieldName = "filePath",
        ContentFieldNames = { "content" },
        ContentFieldSeparator = { },
        VectorFieldNames = { "vector" },
        ImageVectorFieldNames = { }
    }
});

ChatCompletion completion = chatClient.CompleteChat(
    options: options,
    messages: [
        new UserChatMessage(""what is reciprocal rank fusion"")
    ]);
{% endhighlight %}

Although this makes live easier, you have no full control of what is actually happens and based on your use case you still might want to update or manipulate things before you use them as context.


*Thats it folks, hope you can make use of this in your current or next project*

## Documentation
 - [Microsoft documentation about RRF/Hybrid Search](https://learn.microsoft.com/en-us/azure/search/hybrid-search-ranking)
 - [RRF example with LangChain](https://github.com/langchain-ai/langchain/blob/master/cookbook/rag_fusion.ipynb?ref=blog.langchain.dev)