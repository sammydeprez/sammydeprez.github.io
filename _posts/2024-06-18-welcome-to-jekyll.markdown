---
layout: post
title:  "Sending images to OpenAI via the C# SDK"
date:   2024-06-18 23:28:43 +0300
categories: c#
---
While preparing a training for a customer, one of the examples I want to show was how you can send images to the OpenAI ChatCompletion endpoint.
I could not find any examples online how to do it in C# so I delft into the sourcecode, try to understand how it should be done.

After some try and error, I finally got some code working. See below the snippet in case you want to make use of an url.

{% highlight ruby %}
var response = chatClient.CompleteChat(
    messages: [
        new SystemChatMessage("You are an assistant for a car insurance company, please describe the image given by the user and provide a list of parts that need to be replaced or fixed."),
        new UserChatMessage(
            content: new List<ChatMessageContentPart>(){
                ChatMessageContentPart.CreateTextMessageContentPart("How can I fix this car?"),
                ChatMessageContentPart.CreateImageMessageContentPart(
                    imageUri: new Uri("https://www.after-car-accidents.com/wp-content/uploads/2019/01/minor-car-accident.png")
                )
            }
        )
    ]
);
{% endhighlight %}

And below code if you want to send binary data

{% highlight ruby %}
var response = chatClient.CompleteChat(
    messages: [
        new SystemChatMessage("You are a helpfull assistant."),
        new UserChatMessage(
            content: new List<ChatMessageContentPart>(){
                ChatMessageContentPart.CreateTextMessageContentPart("Whats the title of this document?"),
                ChatMessageContentPart.CreateImageMessageContentPart(
                    imageBytes: new BinaryData(File.ReadAllBytes("..\\sample-docs\\contract.png")),
                    imageBytesMediaType: "image/png"
                )
            }
        )
    ]
);
{% endhighlight %}