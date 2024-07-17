---
layout: post
title:  "Azure Function with FastAPI + Non HttpTriggers"
date:   2024-07-19 07:00:00 +0300
categories: blog
---

Integrating FastAPI with Azure Functions can enhance the functionality of serverless applications, especially when you're looking to create robust APIs with Python. While FastAPI doesn't natively support Azure-specific triggers like Queue, Timer, or Blob, it's possible to set up a workaround. This involves using the Azure Functions' bindings and connecting them with FastAPI endpoints. By doing so, you can enjoy the benefits of FastAPI's features, such as automatic Swagger documentation and data validation, while also leveraging Azure Functions' powerful event-driven triggers.

## Lets have a look how

### Step 1 - Create a new Azure Function in VSCode

1. Open Visual Studio Code
2. Open a new folder
3. Open the *Command Pallete* (Ctrl + Shift + P)
4. Choose: Azure Function: Create Function
5. Select: the folder where you want to add the code
6. Select: Python as your coding language
7. Select: Azure Function Model V2
8. Select: Your Python Version (so make sure you have installed Python on your local PC/Mac)
9. Select: HttpTrigger as template
10. Choose a name for the function, call it something generic because all your FastAPI endpoints will be part of this (I leave it with http_trigger)
11. And finally: Anonymous as authorization (you will need to implement your own security within your FastAPI endpoints)

![VSCode Workspace]({{site.url}}/images/fastapi-function-1.png)

### Step 2 - Add a trigger function

This could be any other trigger then HTTP, but for the sake of he example we will use a blob

1. Open the *Command Pallete* (Ctrl + Shift + P)
2. Choose: Azure Function: Create Function
3. Choose: Add BlobTrigger function to the main file
4. Provide a name, container (of your storage account), 
5. Provide a container name (this is the container of your storage account)
6. Provide a connection string app setting variable (connection string to your storage account)

This code was added to your function_app.py file

{% highlight ruby %}
@app.blob_trigger(arg_name="myblob", path="mycontainer",
                               connection="AzureWebJobsStorage") 
def BlobTrigger(myblob: func.InputStream):
    logging.info(f"Python blob trigger function processed blob"
                f"Name: {myblob.name}"
                f"Blob Size: {myblob.length} bytes")
{% endhighlight %}

### Step 3: Ready to convert

**Step 3.1 - Requirements.txt**

First we need to make sure that fastapi library is available for the code. So we need to add the library to our requirements.txt.

Add fastapi==0.111.1 (or when you read this blog it might be a newer version already) to the requirements.txt file

{% highlight ruby %}
azure-functions==1.20.0
fastapi==0.111.1
{% endhighlight %}
By adding the version numbers to the packages you are sure that when a new version is released, that you have no breaking changes when you redeploy.

**Step 3.2 - host.json**

In the host.json we need to update the routePrefix, that way api endpoints do not have a prefix of api/ (Ex. http://func-example.azurewebsites.net/**api**/myfunction), but you can call them immediatly (Ex. http://func-example.azurewebsites.net/myfunction).

Your host file should look like this.
{% highlight ruby %}
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "excludedTypes": "Request"
      }
    }
  },
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[4.*, 5.0.0)"
  },
  "extensions": {
    "http": {
      "routePrefix": ""
    }
  }
}
{% endhighlight %}

**Step 3.4 - function_app.py**

First of all start with adding your fastapi package

{% highlight ruby %}
import fastapi
{% endhighlight %}

You will notice we already have a FunctionApp

{% highlight ruby %}
app = func.FunctionApp(http_auth_level=func.AuthLevel.ANONYMOUS)
{% endhighlight %}

That you can leave like it is, but add underneith the following code, this initializes the FastAPI:

{% highlight ruby %}
fastapi_app = fastapi.FastAPI(root_path="")
{% endhighlight %}

At this point you will reach the httpTrigger endpoint. To make it easy, you can remove all that code. (You can remove all the code starting from *@app.route* till but not including *@app.blob_trigger*)

Now at the end of your code file or in between the fastapi initialization and the blob trigger.
We will add following code:

{% highlight ruby %}
@app.function_name("http_trigger")
@app.route(route="{*route}", auth_level=func.AuthLevel.ANONYMOUS)
async def register_apis(req: func.HttpRequest) -> func.HttpResponse:
    return await func.AsgiMiddleware(fastapi_app).handle_async(req)
{% endhighlight %}

On line 1, you can define the name that you want to give all the http triggers (this will only be visible in the Azure Portal as one of the functions)
On line 2 we define the route, notice this is a variable because it needs to be able to handle all routes.
The register_apis function will register the fastapi_app in the Azure Function *app* Middleware

At this point you can run/debug the code ()
And you will notice it returns your BlobTrigger and the http_trigger(FastAPI)

![2 Running Functions]({{site.url}}/images/fastapi-function-2.png)

**Step 3.5 - Add a fastapi route**

1. Create a folder **routes** in the root
2. In that folder create a file **hello_world.py**
3. Add following code to that file
{% highlight ruby %}
import fastapi

router = fastapi.APIRouter()

@router.post("/hello_world")
async def chat() -> str:
    return "Hello World via POST"


@router.get("/hello_world")
async def chat() -> str:
    return "Hello World via GET"
{% endhighlight %}

On line 3 we initialize the fastapi Router.
The APIRouter class in FastAPI is used to group path operations, allowing you to structure your app across multiple files. You can include it directly in your FastAPI app or nest it within another APIRouter. It’s a powerful tool for organizing routes and keeping your codebase clean and maintainable. 

Because of our combination with Azure Functions we need to make use of this router class.
Notice that 1 file can have multiple endpoints.
In this example we have a POST and a GET statement.

If you run this, it will not work yet. Why? Because we still need to register the route in the fastapi app.

Go back to the function_app.py and import your route 

{% highlight ruby %}
from routes import hello_world
{% endhighlight %}

And finally register your route with the fastapi app

{% highlight ruby %}
fastapi_app = fastapi.FastAPI(
    root_path="", 
    version="v1")

fastapi_app.include_router(hello_world.router)
{% endhighlight %}

Time to run/debug. Notice that no functions are added to the list. All fastapi functions are behind the http_trigger.


![A working endpoint]({{site.url}}/images/fastapi-function-3.png)

But!!!

I have OpenAPI documentation, an awesome feature of FastAPI, out of the box.
Just surf to your endpoint /docs (http://localhost:7071/docs)

![A working endpoint]({{site.url}}/images/fastapi-function-4.png)

## Template?
Offcourse the full template can be found on my github.
Just clone it and add your own endpoints
```
https://github.com/sammydeprez/fastapi-azure-function-template
```

Just share or tag me if you like this blog!