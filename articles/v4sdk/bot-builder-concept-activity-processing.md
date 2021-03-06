---
title: Activity processing | Microsoft Docs
description: Understand activity processing in the bot SDK.
author: jonathanfingold
ms.author: jonathanfingold
manager: kamrani
ms.topic: article
ms.prod: bot-framework
ms.date: 03/22/2018
monikerRange: 'azure-bot-service-4.0'
---

# Activity processing
[!INCLUDE [pre-release-label](../includes/pre-release-label.md)]

Information is exchanged between a bot and the user as activities. Each activity received by your bot application is passed to a bot adapter, which handles giving activity information to your bot as well as sending any responses to the user.

> [!IMPORTANT]
> Activities, particularly those that [we generate](#generating-responses) during a bot turn, are handled asynchronously. It's a necessary part of building a bot; if you need to brush up on how that all works, check out [async for .NET](https://docs.microsoft.com/en-us/dotnet/csharp/async) or [async for JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) depending on your language choice.

## The bot adapter

A bot is directed by it's **adapter**, which can be thought of as the conductor for your bot. The adapter is responsible for directing incoming and outgoing communication, authentication, and so on. The adapter differs based on it's environment (the adapter internally works differently locally versus on Azure) but in each instance it achieves the same goal. In most situations we don't work with the adapter directly, such as when creating a bot from a template, but it's good to know it's there and what it does. 

The bot adapter encapsulates authentication processes and sends activities to and receives activities from the Bot Connector Service. When your bot receives an activity, the adapter wraps up everything about that activity, creates a [context object](#turn-context), passes it to your bot's application logic, and sends responses generated by your bot back to the user's channel.

## Authentication

The adapter authenticates each incoming activity the application receives, using information from the activity and the `Authentication` header from the REST request.

The adapter uses a connector object and your application's credentials to authenticate the outbound activities to the user.

Bot Connector Service authentication uses JWT (JSON Web Token) `Bearer` tokens and the **MicrosoftAppID** and **MicrosoftAppPassword** that Azure creates for you when you create a bot service or register your bot. Your application will need these credentials at initialization time, to allow the adapter to authenticate traffic.

> [!NOTE]
> If you are running or testing your bot locally, for example, using the Bot Framework Emulator, you can do so without configuring the adapter to authenticate traffic to and from your bot.

## Turn context

When an adapter receives an activity, it generates a **turn context** object, which provides information about the incoming activity, the sender and receiver, the channel, the conversation, and other data needed to process the activity. The adapter then passes this context object to the bot. The context object persists for the length of a turn, and provides information on the following:

* Conversation - Identifies the conversation and includes information about the bot and the user participating in the conversation. 
* Activity - The requests and replies in a conversation are all types of activities. This context provides information about the incoming activity, including routing information, information about the channel, the conversation, the sender, and the receiver. 
* Intent - Represents what the bot thinks the user wants to do, if your bot is set up to provide this. Intent is determined by an intent recognizer, such as [LUIS](bot-builder-concept-luis.md), and can be used it to change conversation topics. 
* State - Your bot can use conversation properties to keep track of where the bot is in a conversation or its [state](bot-builder-storage-concept.md) and user properties to save information about a user.  
* Custom information – If you extend your bot either by implementing middleware or within your bot logic, you can make additional information available in each turn.

You can add middleware, which are layers of plug-ins added to your bot and your core bot logic. Middleware is discussed in the next section.

The middleware and the bot logic use the context object to retrieve information about the activity and act accordingly. The middleware and the bot can also update or add information to the context object, such as to track state for a turn, a conversation, or other scope. The SDK provides some _state middleware_ that you can use to add state persistence to your bot.

The context object can also be used to send a response to the user, and get a reference to the adapter to create a new conversation or continue an existing one.

> [!NOTE]
> Your application and the adapter will handle requests asynchronously; however, your business logic does not need to be request-response driven.

## Middleware

Middleware is simply a class that sits between the adapter and your bot logic, added to your adapter's middleware collection during initialization. The SDK allows you to write your own middleware or add reusable components of middleware created by others. What can you do in middleware? Just about anything... Every activity coming into or out of your bot flows through middleware. 

The adapter processes and directs incoming activities in through the bot middleware pipeline to your bot’s logic and then back out again. As each activity flows in and out of the bot, each piece of middleware can inspect or act upon the activity, both before and after the bot logic runs.

The question that often comes up: what should go in middleware versus your bot logic? Middleware is pretty flexible, so ultimately that is up to you. Let's cover a couple of good uses for middleware.

#### Looking at or acting on every activity

There are plenty of situations that require our bot to do something on every activity, or for every activity of a certain type. For example, we may want to log every message activity our bot receives, or provide a fallback response if the bot has not otherwise generated a response this turn. Middleware is a great place for this, with its ability to act both before and after the rest of the bot logic has executed.

#### Modifying or enhancing the turn context

Certain conversations can be much more fruitful if the bot has more information than what is provided in the activity. Middleware in this case could look at the conversation state information it has so far, query an external data source, and append that to the context object before passing execution on to the bot logic.
For example, middleware could identify the conversation details, such as the conversation ID and state, and then query a directory service for information. The middleware could add the user object received from that external query to the context object and pass it on, providing more data about the user and allowing the bot to better handle the request.

Middleware may fall into both of the uses above, or into another use completely;
it all depends on how you want your bot to be structured and what your bot is trying to achieve.
Middleware can establish or persist state, react to incoming requests, or short circuit the pipeline. 
Some candidates for middleware include:
-	**storage or persistence**: Use middleware to store or persist values after a turn, or update some value based off what happened that turn.
-   **error handling or failing gracefully**: If an exception is thrown, middleware can send a user friendly message for that exception.
-	**translation**: This middleware could detect and translate incoming messages, and set up outgoing messages to be translated back into the detected incoming language.

You can define your own middleware, or the SDK defines some middleware that represents application-independent components, such as:
-	State middleware that persists and restores state information on the context object. The SDK provides conversation and user state middleware.
-	Intent recognizers that analyze user messages to extrapolate the user's intent. The SDK provides a LUIS recognizer.
-	Query recognizers that analyze user messages to match against a query knowledge base. The SDK provides a QnA Maker recognizer.
-	Translation middleware that can recognize the input language and translate to another language, such as one that your application understands.

## The bot middleware pipeline

For each activity, the adapter calls middleware in the **order in which you added it**. The adapter passes in the context object for the turn and a _next_ delegate, and the middleware calls the delegate to pass control to the next middleware in the pipeline. Middleware also has an opportunity to do things after the _next_ delegate returns before completing the method. You can think of it as each middleware object has the first-and-last chance to act with respect to the middleware objects that follow it in the pipeline.

For example:
- 1st middleware object’s turn handler executes code before calling _next_.
    - 2nd middleware object’s returnquest handler executes code before calling _next_.
        - The bot’s turn handler executes and returns
    - 2nd middleware object’s returnquest handler executes any remaining code before returning.
- 1st middleware object’s turn handler executes any remaining code before returning.

If middleware doesn’t call the next delegate, the adapter does not call any of the subsequent middleware or bot turn handlers, and the pipeline short circuits.

Once the bot middleware pipeline completes, the turn is over, and the turn context goes out of scope.

Middleware or the bot can generate responses and register response event handlers, and esponses are handled in separate processes.

### Short circuiting

An important idea around middleware (and event handlers that you'll see below) is _short circuiting_. If execution is to continue through the layers that follow it, middleware (or a handler) is required to pass execution on by calling it's _next_ delegate.  If the next delegate is not called within that middleware (or handler), the current pipeline will short circuit and subsequent layers are not executed.

## Generating responses

The context object provides activity response methods to allow code to respond to an activity:
-	The _send activity_ and _send activities_ methods sends one or more activities to the conversation.
-	If supported by the channel, the _update activity_ method updates an activity within the conversation.
-	If supported by the channel, the _delete activity_ method removes an activity from the conversation.

Each response method runs in an asynchronous process. When it is called, the activity response method clones the associated event handler list before starting to call the handlers, which means it will contain every handler added up to this point but will not contain anything added after the process starts.

This also means the order of your responses is not guaranteed, particularly when one task is more complex than another. If your bot can generate multiple responses to an incoming activity, make sure that they make sense in whatever order they are received by the user.

> [!IMPORTANT]
> The thread handling the primary bot turn deals with disposing of the context object when it is done. If a response (including its handlers) take any significant amount of time and try to act on the context object, they may get a `Context was disposed` error. **Be sure to `await` any activity calls** so the primary thread will wait on the generated activity before finishing it's processing and disposing of the turn context.

## Response event handlers

In addition to the bot and middleware logic, response handlers (also sometimes referred to as activity handlers) can be added to the context object. These handlers are called when the associated response happens on the current context object, before executing the actual response. These handlers are useful when you know you'll want to do something, either before or after the actual event, for every activity of that type for the rest of the current response.

> [!WARNING]
> Be careful to not call an activity response method from within it's respective response event handler, for example, calling the send activity method from within an _on send activity_ handler. Doing so can generate an infinite loop.

Each new activity gets a new thread to execute on. When the thread to process the activity is created, the list of handlers for that activity is copied to that new thread. No handlers added after that point will be executed for that specific activity event.

The adapter manages the handlers registered on a context object very similarly to how it manages the middleware pipeline.

For each response event, the adapter calls handlers in the order in which they were added. The adapter passes in the context object for the turn and a _next_ delegate and the handler calls the delegate to pass control to the next registered event handler. If a handler doesn’t call the next delegate, the adapter does not call any of the subsequent event handlers; the event **short circuits**, and the adapter does not send the response to the channel.

## Next steps

Now that you're familiar with some key concepts of a bot, let's dive into the details of how a bot can send proactive messages.

> [!div class="nextstepaction"]
> [Proactive Messaging](bot-builder-proactive-messages.md)