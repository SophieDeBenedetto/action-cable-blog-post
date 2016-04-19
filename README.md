# Real Time Rails: Implementing WebSockets in Rails 5 with Action Cable


Recent years have seen the speedy rise of "the real-time web". In fact, when we think of many of the popular web apps we use every day, we are probably thinking of real-time features––new posts appearing at the top of your Facebook news feed, new emails appearing in your Gmail inbox, new Tweets showing up at the top of your feed. All of this without you having to so much as lift a finger to click a button. 

While we may take features such as these for granted, they represent a significant departure from the HTTP protocol. With HTTP, the client (i.e. the browser) sends a request to the server for content. The server sends back a response––a string of HTML or JSON, for example––to the client.

Real-time web, in contrast, loosly describes a system in which users recieve new information from the server as soon as it is created.  

There are a number of strategies and technologies for implementing such real-time functionality, but WebSocket protocol has been rising to prominence since its development in 2009. 

### What are WebSockets?

WebSockets are a protocol built on top of TCP. They hold the connection to the server open so that the server can send information to the client––not only in response to a request from the client. Web sockets allow for bi-directional (called "full duplex") communication between the client and the server by creating a persistant connection between the two.

How is such a bi-directional connection initiated? A client makes a request to the server and, if that request contains an `upgrade` header that specifies `websocket`, then the browser will initiate a socket connection, which is held open between the client and the server. 

Up until very recently, implementing WebSocket protocol in Rails was difficult. There was no native support, and any real-time feature required integrating third party libraries and strategies like Faye or Javascript polling. 

However, with the development of Action Cable, and its recent integration into Rails 5, we now have a full-stack, easy-to-use implementation of WebSockets that follows the Rails design patterns we've come to rely on. 

### The Path to Real Time Rails

In his keynote talk at RailsConf 2015, DHH unveiled Action Cable, since calling it "the highlight of Rails 5". 

Having used various other technologies and tools to build real-time features into Rails application (including both Faye and Javascript polling), I (and many others) really felt the lack of a native-to-Rails real-time offering.

So why does it feel like Action Cable was such a long time coming? 

First off, let's not forget that, in the words of DHH from that same RailsConf talk, "dealing with WebSockets is a pain in the [you know what]". Although it wasn't necessarily a pleasure to code, you *could* build real-time features into Rails with the tools we've already mentioned in this post. In fact, Campfire, Base Camp's own chatting application, has successfully, and we have to imagine not *too* painfully, used polling since it was first developed ten years ago. 


In that same keynote talk, DHH asked: "If you can make WebSockets even less work than polling, why wouldn't you do it?". DHH has talked in the past about being a developer "prepper", he packs just enough tools in his backpack to get something up and running. Polling met the needs of his team, and many others, for many years. But, as more and more consumers and developers began demanding real time functionality, and as newer frameworks like Phoenix arrived to meet that demand, Rails 
felt the need to deliver. In fact, Action Cable draws some inspiration from Phoenix channels. 

Having tried to work with earlier version of Action Cable, before it was merged into Rails 5, I would say that it *wasn't* easier than polling. Now, however, its very easy to implement and aligns nicely with the other design patterns we've become so comfortable with in Rails. 

So, how does the "highlight" of Rails 5 work, and what's it like to implement? Let's take a closer look

### Introducing Action Cable

Action Cable

> ...seamlessly integrates websockets with the rest of your Rails application. It allows for real-time features to be written in Ruby in the same style and form as the rest of your Rails application, while still being performant and scalable. It's a full-stack offering that provides both a client-side JavaScript framework and a server-side Ruby framework. You have access to your full domain model written with ActiveRecord or your ORM of choice.[*](https://github.com/rails/rails/tree/master/actioncable)

This last piece is one of the key features that makes Action Cable so comfortable to work with––we have access to all of our models from within our WebSocket workers, effectively layering Action Cable on top of our existing Rails architecture. 

#### Action Cable Under the Hood

Before we dive in to some code, let's take a closer look at how Action Cable opens and maintains the WebSocket connection inside our Rails 5 application. 

Action Cable can be run on a stand-alone server, or we can configure it to run on its own processes within the main application server. In this post, we'll be taking a look at the second approach. 

Action Cable uses the Rack socket hijacking API to take over control of connections from the application server. Action Cable then manages connections internally, in a multithreaded manner, layering as many channels as you care to define over that socket connection. 

For every instance of your application that spins up, an instance of Action Cable is created, using Rack to open and maintain a persistent connection, and using a channel mounted on a sub-URI of your main application to subscribe, or stream, from certain areas of your application, and publish, or broadcast, to other areas. Action Cable offers server-side code to broadcast certain content (think new messages or notifications) over the channel, to a subscriber. The subscriber is instantiated on the client-side with a handy JavaScript function that uses jQuery to append new content to the DOM. 

Lastly, Action Cable uses Redis as a data store for transient data, syncing content across instances of your application. 

![](Figure 1)

Now that we have a basic understanding of how Action Cable works, we'll build out a basic chatting application in Rails 5, taking a closer look at how Action Cable behaves along the way. 

### Getting Started

 





I. Real Time Communication and Rails
 * Rails and WebSocket protocol: 
    * what are websockets and why do we need them? 
    * Ruby and websockets don't necessarily play nice, life before action cable was hard. we had some options--faye/private_pub etc., but a lot of       
      hacking was required.
 
II. What is Action Cable: brief intro/definition

III. Let's Build It! Setting up a basic chatting application in Rails 5 with Action Cable
  * Brief set up: messages, chatrooms, user resources
  * Implementing Action Cable to broadcast and stream new messages to the appropriate chatroom
       * set up the new message broadcast on message create
       * establish the messages channel, tell it to stream messages broadcast from create
       * client side: subscribe to messages stream from the client side
  * Configuring Action Cable to run on our main application server
       * Action Cable is running its own separate processes on a sub-URI of your main server
       * How to configure application to do so: mount Action Cable server in your routes file, configure action cable + redis, on the client side, create the          web socket consumer

IV. Deploying to Heroku
   * Configuring Puma (or any threaded server)
   * Add-ons: Redis
   * Action Cable in development vs. production environment
   * That's it!
"dealing with websockets is a pain in the ass"
-DHH

"there must be a great implementation that I can just rip off" (Phoenix)

"If you can make websockets even less work than polling, why wouldn't you do it?"

Campfire polls once every 3 seconds. 

"What if there was something where we could take all 3 use cases, in the same app, and use the same system to build it?"

AC: every user creates one connection to the app, one cable. that cable is the ws connection. one session = one connection. Over that cable, we layer different channel. Those diff channels are those diff use cases (inbox, notifications, chat). Same socket connection, different channels using the connection. 

Concept of the broadcaster: the mechanism on the server side that sends stuff through these channels, to the user. That connection uses Redis pub/sub becuase it is "stupid simple and easy to use". 

Once connection per session. When you make that connection, you have to authenticate it. Initiate http connection, then switch over to WS connection using the cookie that was just established. 

Channel: I want to consume _this_ stuff. 
create new client-side channel by extending from AC.Channel. Relate that channel to a DOM element, using a data-behavior or data-attribute. Initially, instantiate this channel as a persistent connection. 

Important: received function: subscribe to the server side and say "i want everything that is coming in to this chatroom, i.e. all the messages that are sent by the messages broadcast". Have to have a corresponding channel on the server-side. When JS function is instantiated, it automatically matches up to the server side channel, calls subscribed, which subscribes to a specific redis pub-sub channel. That redis pub/sub channel has to have a name. (keep messages in sync across app instances)/data store for transient data. 

broadcast to redis channel, stream from redis channel

SELF RELIANCE: Examples of self-reliance. Using an integrated system to allow tiny teams to create full-stack app. 

you want that swiss army knife in your backpack to be able to do all sorts of things, because you are only able to carry a few things. "That's unpure"

EventMachine does the connection logic: create and disconnect WS connection. Not great for application logic. Pairs with Thread to use existing models/integrated system of your main application. 