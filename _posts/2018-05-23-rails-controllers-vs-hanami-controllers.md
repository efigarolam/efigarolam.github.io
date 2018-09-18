---
layout: post
title: "Controllers: the Rails way vs the Hanami way"
originally_published_at: https://sipsandbits.com/2018/05/23/rails-controllers-vs-hanami-controllers/
description: Recently I have become interested in Hanami and so far I have really liked the architecture design and decisions that its author, Luca Guidi, has been taking until now.
---

Recently I have become interested in [Hanami](http://hanamirb.org/) and so far I have really liked the architecture design and decisions that its author, [Luca Guidi](https://github.com/jodosha), has been taking until now.

I have decided to break the Rails vs Hanami comparison on different blog posts, one per component, in order to keep them small and concise.

This is the first post of that series, and I'm going to start with letter **C** from the MVC design pattern, **the Controllers**.

Let's start by talking about what an actual Controller is in each framework.

## The basics

### Rails

Controllers are classes that are in charge of processing inbound requests, previously handled by the **Router** and then submit the output to the requester (**client**).

Each public instance method in the Controller will process a different kind of request. See the code below:

```ruby
# app/controllers/users_controller.rb
class UsersController < ApplicationController
  def index; end     # GET /users
  def show; end      # GET /users/1
  def new; end       # GET /users/new
  def create; end    # POST /users
  def edit; end      # GET /users/1/edit
  def update; end    # PATCH /users/1
  def destroy; end    # DELETE /users/1
end
```

In this particular example, I'm showing you a **RESTful** controller. That's one of the best practices when working with **Rails**, use **RESTful** wherever you can to provide access to your application **Resources**.

### Hanami

In **Hanami**, Controllers are **modules** whose unique responsibility is to group **Actions** (which are the classes) that are in charge of processing a very specific inbound request, previously handled by the **Router** too and then submit the output to the client.

```ruby
# apps/web/controllers/users/index.rb
module Web::Controllers::Users
  class Index
    include Web::Action

    def call(params)
    end
  end
end

# apps/web/controllers/users/show.rb
module Web::Controllers::Users
  class Show
    include Web::Action

    def call(params)
    end
  end
end

...
```

With those two **Action** examples, you get the idea of what's going on.

### Differences

In **Rails**, you have one **Controller Class** with as many public instance methods as you need. 

In **Hanami**, you need one **Controller Module** and as many **Action Classes** as you need. **Hanami** Actions are only required to define a **call** public instance method.

### What approach is better and why?

In my opinion, I think that the Hanami architecture is cleaner because one class per **Action** accomplishes the **Single Responsibility Principle** which states that *one Class must be responsible for doing one thing.*

On the other side, a really known problem with **Rails' Controllers** is the trend to become really big classes over time that sometimes are hard to read and understand at first glance. Are you familiar with the *Thin Controllers, Fat models* "best practice"? Would it be better to not worry about getting a *Fat  Controller* at all?

It's pretty easy to avoid **Fat Actions** with Hanami. You are required to handle one request action per class, and if you find any code duplication within your **Controller's Actions**, Ruby has a built-in solution for that: **modules**. You can just place the repeated code in a **module** and then include it in the **Actions** classes where needed. See the example below:

```ruby
# apps/web/controllers/users/set_user.rb
module Web::Controllers::Users
  module SetUser
    def self.included(action)
      action.class_eval do
        before :set_user
      end
    end

    private

    def set_user
      @user = UserRepository.new.find(params[:id])
      halt 404 if @user.nil?
    end
  end
end

# apps/web/controllers/users/show.rb
require_relative './set_user'

module Web::Controllers::Users
  class Show
    include Web::Action
    include SetUser

    def call(params)
      # ...
    end
  end
end

# apps/web/controllers/users/edit.rb
require_relative './set_user'

module Web::Controllers::Users
  class Edit
    include Web::Action
    include SetUser

    def call(params)
      # ...
    end
  end
end
```

Solid, right?

## Exposing variables to the views/templates

The **Controllers** are in charge of exposing data to the **View** layer. Hanami and Rails do this in a very similar fashion with a small difference.

### Rails

In Rails, any instance variable that you declare in your controller is accessible in the templates (Views).

### Hanami

In Hanami, instance variables are exposed only if you say so via the `expose` class method. Here is an example of it:

```ruby
# apps/web/controllers/users/index.rb
module Web::Controllers::Users
  class Index
    include Web::Action
    expose :users # We are exposing the @users instance variable.

    def call(params)
      @users = UserRepository.new.all
      @another_instance_variable = {} # This will not be accesible in the view/template
    end
  end
end
```

## Wrapping up

By now, you could say that Hanami needs more code to accomplish the same things that you can do in Rails, and it might be true. Rails architectural design relies more on **Convention over Configuration**, that's why it feels like magic to work with Rails.

On the other hand, Hanami has fewer conventions and forces us to be more explicit with our intentions. In my opinion, having separate classes per Controller Action is a really good idea, because we just have to worry about what is happening on a single inbound request. And that architecture just feels right.

On the next blog post of the series, I will continue with the comparison of Rails Models (ActiveRecord) and Hanami Model Domain (Entities & Repositories)
