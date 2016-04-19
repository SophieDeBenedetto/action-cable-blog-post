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

## Building a Chat with Action Cable

### Getting Started: Application Architecture

In this example, we'll build a basic chatting application in which users can log in or sign up to create a username, then create a new chatroom or choose from an existing chatroom and start messaging. We'll use Action Cable to ensure that our chat feature is real-time: any users in a given chatroom will see new messages (their own and the messages of other users) appear in the chat, without reloading the page or engaging in any other action to request new content. 

You can follow along below, check out the code for this project on [GitHub](https://github.com/SophieDeBenedetto/action-cable-example), visit my deployment [here](https://action-cable-example.herokuapp.com/) or, deploy your own copy by clicking this button. 


### Starting a new Rails 5 app with Action Cable

At the time of writing, Rails 5 is in Beta 3. So, to start a new Rails 5 app, we need to do the following. 

* First, make sure you have installed and are using Ruby 2.3.0. 
* Then:

```bash
$ gem install rails --pre
Successfully installed rails-5.0.0.beta3
Parsing documentation for rails-5.0.0.beta3
Done installing documentation for rails after 1 seconds
1 gem installed
```

* Now we're ready to generate our new app!

```bash
rails new action-cable-example --database=postgresql
```

This will generate a shiny new Rails 5 app, complete with all kinds of goodies in our Gemfile. Open up the Gemfile and you'll notice that we have:

* The latest version of Rails

```ruby
# Gemfile
gem 'rails', '>= 5.0.0.beta3', '< 5.1'
```

* Redis (which we will need to un-comment out): 

```ruby
# Gemfile
# gem 'redis', '~> 3.0'
```

* Puma (since Action Cable needs a threaded server):

```ruby
# Gemfile
gem 'puma'
```

Make sure you've un-commented out the Redis gem from your Gemfile, and `bundle install`. 
 
### Domain Model
 
Our domain model is simple: we have users, chatrooms and messages. 
 
A **chatroom** will have a topic and it will have many messages. A **message** will have content, and it will belong to a user and belong to a chatroom. A **user** will have a username, and it will have many messages. 

The remainder of this tutorial will assume that we have already generated the migrations necessary to create the chatrooms, messages and users table, and that we have already defined our Chatroom, User and Message model to have the relationships described above. Let's take a quick look at our models before we move on. 

```ruby
# app/models/chatroom.rb

class Chatroom < ApplicationRecord
  has_many :messages, dependent: :destroy
  has_many :users, through: :messages
  validates :topic, presence: true, uniqueness: true, case_sensitive: false
  before_validation :sanitize, :slugify
end
```

```ruby
# app/models/message.rb

class Message < ApplicationRecord
  belongs_to :chatroom
  belongs_to :user
end
```

```ruby
# app/models/user.rb

class User < ApplicationRecord
  has_many :messages
  has_many :chatrooms, through: :messages
  validates :username, presence: true, uniqueness: true
end
```

### Application Flow

We'll also assume that our routes and controllers are up and running. A user can log in with a username, visit a chatroom and create new messages via a form on the chatroom's show page. 

So, the `#show` action of the Chatrooms Controller sets up both a `@chatroom` instance as well as a new, empty `@message` instance that will be used to build our form for a new message, on the chatroom's show page:

```ruby
# app/controllers/chatrooms_controller.rb

class ChatroomsController < ApplicationController
  ...
  
  def show
    @chatroom = Chatroom.find_by(slug: params[:slug])
    @message = Message.new
  end
end
```

Our chatroom show page renders the partial for the message form:

```erb
# app/views/chatrooms/show.html.erb

<div class="row col-md-8 col-md-offset-2">
  <h1><%= @chatroom.topic %></h1>

<div class="panel panel-default">
  <% if @chatroom.messages.any? %>
    <div class="panel-body" id="messages">
      <%= render partial: 'messages/message', collection: @chatroom.messages%>
    </div>
  <%else%>
    <div class="panel-body hidden" id="messages">
    </div>
  <%end%>
</div>

  <%= render partial: 'messages/message_form', locals: {message: @message, chatroom: @chatroom}%>
</div>
```

And our form for a new message looks like this:

```erb
# app/views/messages/_message_form.html.erb

<%=form_for message, remote: true, authenticity_token: true do |f|%>
  <%= f.label :your_message%>:
  <%= f.text_area :content, class: "form-control", data: {textarea: "message"}%>

  <%= f.hidden_field :chatroom_id, value: chatroom.id %>
  <%= f.submit "send", class: "btn btn-primary", style: "display: none", data: {send: "message"}%>
<%end%>
```

Our new message forms posts to the create action of the Messages Controller

```ruby
class MessagesController < ApplicationController

  def create
    message = Message.new(message_params)
    message.user = current_user
    if message.save
      # do some stuff
    else 
      # redirect_to chatrooms_path
    end
  end

  private

    def message_params
      params.require(:message).permit(:content, :chatroom_id)
    end
end
```

Okay, now that we have our app up and running, let's get those real-time messages working with Action Cable 

## Implementing Action Cable

Before we define our very own Messages Channel and start working directly with Action Cable code, let's take a quick tour of the Action Cable infrastructure that Rails 5 provides for us. 

When we generated our brand new Rails 5 application, the following directory was generated for us:

```bash
├── app
    ├── channels
        ├── application_cable
        ├── channel.rb
        └── connection.rb
```

Our `ApplicationCable` module has a `Channel` and a `Connection` class defined for us. 

The `Connection` class is where we would authorize the incoming connection––for example if we wanted to establish a channel that required user authorization, like for a given user's inbox. We'll also leave this class alone as any user can join any chatroom at any time. 

```ruby
module ApplicationCable
  class Connection < ActionCable::Connection::Base
  end
end
```


However, the Messages Channel that we will define shortly will inherit from `ApplicationCable::Channel`.

The `Channel` class is where we would place shared logic among any additional channels that we will define. We'll only be creating one channel, the Messages Channel, so we'll leave this class alone. 

```ruby
# app/channels/channel.rb

module ApplicationCable
  class Channel < ActionCable::Channel::Base
  end
end
```

Okay, let's move on to writing our own Action Cable code. 

First things first, setting up the instance of the Action Cable server. 

### Configuring The Action Cable Server

#### Step 1: Establish the Socket Connection

First, we need to mount the Action Cable server on a sub-URI of our main application. 

In `routes.rb`:

```ruby
Rails.application.routes.draw do

  # Serve websocket cable requests in-process
  mount ActionCable.server => '/cable'

  resources :chatrooms, param: :slug
  resources :messages
  
  ...

end
```

Now, Action Cable will be listening for WebSocket requests on `ws://localhost:3000/cable`. It will do so by using the Rack socket hijacking API. When our main application is instantiated, an instance of Action Cable will also be created. Action Cable will, per our instructions in the `route.rb` file, establish a WebSocket connection on `localhost:3000/cable`, and begin listening for socket requests on that URI. 

Now that we've established the socket connection on the server-side, we need to create the client of the WebSocket connection, called the **consumer.**

#### Step 2: 


 

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