---
layout: post
title:  "Query Fields in Document Intelligence: A Hidden Gem"
date:   2024-07-12 07:00:00 +0300
categories: blog
---

Query fields in Document Intelligence are a powerful feature that can significantly enhance your document processing capabilities. This feature allows you to extract specific fields from documents without the need to train a custom model. Instead, you can define the fields you want to extract, and the model will only extract the corresponding values. This is particularly useful when the values you need cannot be described as key-value pairs in the document, such as the agreement date of a contract.

![Query Fields in the Document Intelligence Studio]({{site.url}}/images/query-fields.png)

### Why Query Fields are Powerful

1. **Flexibility**: Query fields provide the flexibility to extract specific information from documents, even if the structure varies. This means you can handle a wide range of document types and formats.
2. **Efficiency**: By specifying only the fields you need, you can streamline the extraction process, making it faster and more efficient.
3. **Scalability**: This feature is ideal for large-scale document processing tasks where training a custom model for each document type would be impractical.
4. **Cost-Effective**: Since you don't need to train custom models, you save on both time and resources, making it a cost-effective solution for many businesses.

### Use Cases

- **Legal Documents**: Extract specific clauses, dates, and parties involved in contracts.
- **Invoices**: Retrieve payment terms, due dates, and amounts without needing to process the entire document.
- **Forms**: Pull out specific fields from various forms, such as names, addresses, and dates.

### How to Use Query Fields in C#

Here's a simple example of how to call the query fields feature in C#:

{% highlight ruby %}
var client = new DocumentIntelligenceClient(
    new Uri("<your-endpoint>"), 
    new AzureKeyCredential("<your-api-key>"));

AnalyzeDocumentContent content = new AnalyzeDocumentContent()
{
    UrlSource = new Uri(<your-document-url>)
};

// Enabling the query fields feature
List<DocumentAnalysisFeature> features = new List<DocumentAnalysisFeature> { DocumentAnalysisFeature.QueryFields};

// A list of descriptive fields to query for in the document (Ex. DocumentTitle, Party1_CompanyName)
List<string> queryFields = new List<string> { "<query-field-1>", "<query-field-2>", "<query-field-n>" };

Operation<AnalyzeResult> operation = await client.AnalyzeDocumentAsync(
    waitUntil: WaitUntil.Completed, 
    modelId: "prebuild-layout", 
    analyzeRequest: content,
    queryFields: queryFields,
    features: features);

return operation.Value;
{% endhighlight %}

In this example, replace `<your-endpoint>`, `<your-api-key>`, and `<your-document-url>` with your actual endpoint, API key, and document URL. This code sends a request to the Document Intelligence API to extract the specified fields from the document.

Query fields are indeed a hidden gem within the Document Intelligence service, offering a powerful, flexible, and cost-effective way to extract specific information from a wide variety of documents.

: [Query field extraction - Document Intelligence](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept-query-fields?view=doc-intel-4.0.0)
: [Add-on capabilities - Document Intelligence](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/concept-add-on-capabilities?view=doc-intel-4.0.0)