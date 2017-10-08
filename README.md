# Azure Serverless IoT Button

## Tweet with Azure Functions and Flic Button

This tutorial shows you how to integrate an Azure Function with your Flic button
by posting a tweet to your Twitter account when the Flic is clicked.  We are going to use Azure Functions for Serverless Compute, and Azure Logic Apps for serverless workflows/integration with Twitter.

Prerequisites:

-   Twitter account
-   Flic button
-   iPhone or Android smartphone with Flic app installed
-   Azure account

The solution will be:

> A Flic button sends an HTTPS request to an Azure Function which processes the data and sends a message to tweet to an Azure Logic App.  The Logic App fires and posts the tweet.

It only takes a few minutes to setup and get working end-to-end.

## Working with Functions in the Azure Portal

Functions can be created, developed, configured, and tested the Azure portal.

![images/1.png](images/1.png)

## Create a Function App

Functions require a function app to host function execution. This can be done in
the Azure portal.

1.  Log in to the Azure portal and click the New button in the upper left-hand
    corner.

2.  Click Compute > Function App. Then, configure your app settings:

    -   App Name: Create a globally unique name.

    -   Subscription: Add a new or existing subscription.

    -   Resource Group: Add a new or existing resource group.

    -   Hosting plan: the Consumption Plan is recommended.

    -   Location: Choose a location near you.

    -   Storage account: Create a globally unique name for the storage account
        that will be used by your function app, or use an existing account.

3.  Click Create.

## Create an HTTP Triggered Function

Now that the function app has been created, a function can be added to it. The
template for an HTTP triggered function will execute when sent an HTTP request.

1.  At the top of the portal, locate and click the magnifying glass button to
    search for your new function app. Enter the function app's name in the
    search bar to find and select it.

2.  Expand your new function app, then click the + button next to functions.

3.  Select the HttpTrigger function template for either C# or JavaScript.

4.  Change the Authorization level to Anonymous

5.  Click Create.

## Configure Function

1.	In the portal, expand the function and click Integrate in the expanded view.
2.	Add the following route to the Route template field: notify/{messageType:alpha}

This will give the function URL a path parameter `messageType` we can access within the function.

Implement Function – C#
========================

The C# implementation will use a NuGet package named linqtotwitter to interact
with the Twitter api.

1.  Select the new function, then click View files on the right hand side.

2.  Click the add button and create a file named `project.json`.

3.  Select project.json and add the following code to include linqtotwitter in
    the function:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
{
  "frameworks": {
    "net46":{
      "dependencies": {
        "linqtotwitter": "4.1.0"
      }
    }
   }
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Save the file.

1.  Navigate to run.csx and remove the existing code from lines 5 to 20, leaving
    only the initial `Run` method and the `using` statement.

2.  Add the following code above `Run`:

>   using System.Net;  
>   using System.Net.Http;  
>   using LinqToTwitter;  
>     
>   private static TwitterContext _twitterCtx = null;  
>   private static IDictionary<string, string> _messageMap;

1.  Add the following code inside the `Run` method:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    if (_twitterCtx == null)
    {
        await SetupTwitterClient();
    }

    if (_messageMap.TryGetValue(messageType, out string message))
    {
        try
        {
            return req.CreateResponse(HttpStatusCode.OK);
        }
        catch (TwitterQueryException ex) {
            return req.CreateResponse(HttpStatusCode.InternalServerError);
        }
    }

    return req.CreateResponse(HttpStatusCode.BadRequest, "Invalid message");
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1.  Add the following method below the `Run` method to authenticate an
    associated Twitter account:

>   private static async Task SetupTwitterClient()  
>   {  
>   SingleUserAuthorizer authorizer = new SingleUserAuthorizer  
>   {  
>   CredentialStore = new SingleUserInMemoryCredentialStore  
>   {  
>   ConsumerKey = Environment.GetEnvironmentVariable("TwitterAppKey",
>   EnvironmentVariableTarget.Process),  
>   ConsumerSecret = Environment.GetEnvironmentVariable("TwitterAppSecret",
>   EnvironmentVariableTarget.Process),  
>   AccessToken = Environment.GetEnvironmentVariable("TwitterAccessToken",
>   EnvironmentVariableTarget.Process),  
>   AccessTokenSecret =
>   Environment.GetEnvironmentVariable("TwitterAccessTokenSecret",
>   EnvironmentVariableTarget.Process)  
>   }  
>   };  
>   await authorizer.AuthorizeAsync();  
>     
>   _messageMap = new Dictionary<string, string>  
>   {  
>   ["arrived"] = "Arrived at #ServerlessConf NYC. Trying out this
>   #AzureFunctions demo cool",  
>   ["joinme"] = "You should join me at the Microsoft booth at #Serverlessconf
>   NYC",  
>   ["azurefunctions"] = "Azure Serverless is awesome!"  
>   };  
>     
>   _twitterCtx = new TwitterContext(authorizer);  
>   }

1.  Save the file.

Implement and Configure Function - JavaScript
=============================================

The JavaScript implementation will use a npm package named twitter to interact
with the Twitter api. This requires accessing the function console through the
portal to install the package and package.json file.

![C:UsersjagreenaAppDataLocalMicrosoftWindowsINetCacheContent.Wordfunction_settings_portal.png](media/34f4ba91826f68aec291acb92bcc0c73.png)

1.  In the portal, select the created function app and select the Platform
    features tab.

2.  Under development tools, click on Console.

3.  In the console, navigate to the function directory: `cd
    myHttpTriggerFunction`

4.  Create a package.json file with default values and add the twitter package
    with the following commands:

>   npm init -y  
>   npm install twitter

1.  Exit the console and select the new function, then click View files on the
    right hand side.

2.  Replace index.js with the following code:

>   let Twitter = require("twitter");  
>     
>   module.exports = function(context, req) {  
>   context.log("Azure Functions Twitter Demo");  
>     
>   var client = new Twitter({  
>   consumer_key: process.env["TwitterAppKey"],  
>   consumer_secret: process.env["TwitterAppSecret"],  
>   access_token_key: process.env["TwitterAccessToken"],  
>   access_token_secret: process.env["TwitterAccessTokenSecret"]  
>   });  
>     
>   let messageMap = {  
>   arrived: "Arrived at #ServerlessConf NYC. Trying out #AzureFunctions",  
>   joinme: "You should join me at the Microsoft booth at #Serverlessconf NYC",  
>   azurefunctions: "Azure Functions is awesome!"  
>   };  
>     
>   let messageType = context.bindingData.messageType;  
>   let statusMessage = messageMap[messageType];  
>     
>   if (statusMessage) {  
>   client.post("statuses/update", { status: statusMessage },  
>   function(error,tweet,  response) {  
>   if (error) throw error;  
>   context.log(tweet); // Tweet body.  
>   context.log(response); // Raw response object.  
>   context.res = {  
>   status: 200,  
>   body: "Tweet sent"  
>   };  
>   context.done();  
>   });  
>   } else {  
>   context.res = {  
>   status: 400,  
>   body: "Invalid request. Messing message type"  
>   };  
>   context.done();  
>   }  
>   };

1.  Save the file.

Create Twitter App
==================

1.  Sign in to Twitter, then navigate to https://apps.twitter.com/ and create a
    new Twitter app by clicking the button on the top right of the page and
    filling out the form. The website is a required field, but not needed for
    the app so you may add a placeholder site. Click on the button at the end of
    the form to create the app.

2.  In the app's details, navigate to the Keys and Access Tokens tab and click
    on the button in the Token Actions section to create an access token.

3.  You will need the Consumer Key, Consumer Secret, Access Token, and Access
    Token Secret for the next step, keep this page open to copy these values
    into to your function configuration.

Configure Function Keys
=======================

1.  In the portal, navigate to the function app that hosts the recently created
    function.

2.  In the function app overview tab, click on Application settings.

3.  Scroll down to the Application settings section, click on the "+ Add new
    setting" button and add the keys from the Twitter app page as name and value
    pairs:

    -   **TwitterAppKey**: Consumer Key

    -   **TwitterAppKeySecret**: Consumer Secret

    -   **TwitterAccessToken**: Access Token

    -   **TwitterAccessTokenSecret**: Access Token Secret

Configure Flic
==============

1.  Copy the function url by navigating to the function in the portal and
    clicking the "</> Get function URL" link. This url is needed in the Flic
    app and can be quite long. It is recommended to paste the url in a cloud
    based document for mobile access.

2.  In the Flic App, connect a button if you haven't already done so and enter
    the button settings by tapping it.

3.  For the click setting, press + to the right of the click command and add a
    Internet Request function to the button by searching in the function menu.

4.  Edit the function by adding the function url and adding one of the three
    routes that is mapped to a tweet message:

    -   **arrived**: "Arrived at #ServerlessConf NYC. Trying out
        #AzureFunctions"

    -   **joinme**: "You should join me at the Microsoft booth at
        #Serverlessconf NYC"

    -   **azurefunctions**: "Azure Functions is awesome!"

The url should look similar to this:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
https://myFunctionAppName.azurewebsites.net/api/notify/azurefunctions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1.  Press done to save the settings.

2.  Repeat steps 3-4 for the button's double click and hold settings. Avoid
    reusing the same routes for each button setting.

Triggering the Function
=======================

Based on your click command configuration, the HTTP function will send a request
to Twitter to authenticate and post a tweet to the specified account with one of
the three predefined messages. Because tweeting the same message twice in a row
is prohibited on Twitter, each button click command will only tweet once. Change
the tweet message text in the code or delete the posted tweets to create more
tweets through button clicks.
