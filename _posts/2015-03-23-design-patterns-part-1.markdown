---
layout: post
title: "Resilient Software Part I"
date: 2015-03-23 12:01:00
categories: design patterns, testing, software architecture
author: "Ben Downey"
---

# Long Lasting Software

Writing high quality software means expecting change: changes to business requirements, changes to the web framework you’ve chosen, changes to the
libraries you depend on and changes to the coding language.  At Sittercity, we often rely on Rails for our web framework, but our Rails projects don't looks like most Rails projects.  When we're building a Rails app, we try to leverage the good parts of Rails and avoid the bad.  We construct views and controllers in a conventional way, but handle persistent storage much differently than an idiomatic rails app.

Part of anticipating change means that we structure our software in a way that minimizes its reliance on the web framework.  To accomplish this, we
rely on a constellation of design patterns that includes repositories, entities, dependency injection, factories, and commands (sometimes called
contexts or use-cases).  I won't spend a lot of time explaining each pattern, so if these are new to you, then I'd suggest you look at the relevant
chapters of [this book](http://designpatternsinruby.com).  Most of what we sketch out in these posts will be a rough approximation of
what you would write in a real project.  I’m omitting details about namespacing, error handling, and testing nuances so that we can focus on the big
picture. In this series of blog posts, I’ll explain how these patterns work in concert to provide us an app that is responsible, resilient, and robust.

In order to explain this architecture, let's use the example of creating a user.

## Repositories and Entities

Most Rails apps use the ORM that ships with Rails, ActiveRecord.  To create a user, we'd have something that looks like this.

{% highlight ruby %}

    User.create(first_name: "John", last_name: "Doe", email: "JohnDoe@Example.com")

{% endhighlight %}

At Sittercity, we often want the flexibility to change parts of the app, and that includes changing the ORM.  So, we'll use the repository pattern to concentrate all ActiveRecord queries into single class.  Instead of the code you saw above, we might have a class that looks like this:


{% highlight ruby %}

class ARUserRepository

  def create(first_name, last_name, email)
    User.create(first_name: first_name, last_name: last_name, email)
  end

end

{% endhighlight %}

If we decide that we don't want to use ActiveRecord anymore, there won't be a lot of work involved.  Let's say that we want to use the Sequel ORM.  We'd end up writing another repository that had the
exact same interface as the ARUserRepository.  We don't have to have an explicit interface, we can just rely on [duck typing](http://en.wikipedia.org/wiki/Duck_typing). If you’re unfamiliar with the idea of interfaces, just know that in the world of Ruby, we can say that two classes have the same interface if they have
the same public methods available to them.) 

{% highlight ruby %}

class SequelUserRepository

  def create(first_name, last_name, email)
    $users.insert(first_name: first_name, last_name: last_name, email: email:)
  end

end

{% endhighlight %}
 
Or, let's imagine we decide that the data we want to store would fit best in a document store like MongoDB.  We'd just write another repo and as long as it implements the same interface as other repositories, we can swap it in without changing any other parts of the app.

In conjunction with repositories, we also use entities.  An entity is a custom class that possess all the attributes of an object that we care about.  As you know, when you save an ActiveRecord object,  Rails will return you an Active Record object. 


{% highlight ruby %}

user = User.new(first_name: "John", last_name: "Doe", email: "JohnDoe@Example.com")
=> #<User id: nil, first_name: "John", last_name: "Doe", email: "JohnDoe@Example.com")
user.save
=> #<User id: 1, first_name: "John", last_name: "Doe", email: "JohnDoe@Example.com")

{% endhighlight %}

Note that start with a new instance of an object that inherits from ActiveRecord::Base and when we save it, we’re still dealing with an object that inherits from ActiveRecord::Base. The main difference
is that it has has its id field populated, indicating that it saved successful to the database.

But what’s the point of using the generic interface of our repository if we’re going to end up returning AR objects?

Entities to the rescue!  

We want our app passing around generic entities whose behavior is completely under our control--unlike ActiveRecord instances.  Using ActiveRecord instances couples our code tightly to the Rails framework.  We don't want that.  After we create a resource in our repository, we take the response and translate it into a custom entity.  

Here's an example of what an entity class might look like.

{% highlight ruby %}

module Entity
  class UserEntity < Struct.new(:id, :first_name, :last_name, :email)

    def initialize(attrs={})
      super(attrs[:id],
            attrs[:first_name],
            attrs[:last_name],
            attrs[:email]
            )
    end

  end
end

{% endhighlight %}

What we've got here is a class that is basically a struct that fits our specific needs.  Ruby's struct class is really useful for situations like this. Structs are easy to initialize and they allow
us to access properties in a simple way.  First, let's initialize an entity

{% highlight ruby %}
  user = UserEntity.new(email: 'jdoe@example.com')
{% endhighlight %}

And when we want to access an attribute of the struct, we can do it like this.

{% highlight ruby %}
  user.email # => jdoe@example.com
{% endhighlight %}

If you've seen ActiveRecord in action, then this should look familiar to you.

Next, let's integrate our entity into our repository. 

{% highlight ruby %}
class ARUserRepository

  def create(first_name, last_name, email)
    ar_user_object = User.create(first_name: first_name, last_name: last_name, email: email)
    user = translate_record(ar_user_object)
    user
  end

  def translate_record(record)
    UserEntity.new(record.attributes)
  end

end

{% endhighlight %}

Now that our repository returns an instance of our entity class, we have successfully insulated our software from the world of ActiveRecord.  

With the repository and entity patterns working in tandem, we’re now in control of the behavior of our user resource.  And when it comes to anticipating change, we have the benefit of having all of our
ActiveRecord queries and actions centralized in a single class.  Any time there are changes to ActiveRecord, we only have to update that class.  That’s a lot easier than searching through file after
file and updating each query individually.  Ultimately, there will still be a few tweaks to this repository class, but what we’ve got here is a great start at taking control of our software.
