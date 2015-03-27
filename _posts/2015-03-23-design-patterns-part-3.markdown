---
layout: post
title: "Design Patterns, Fast Tests, and Robust Software, Part III"
date: 2015-03-23 12:01:00
categories: design patterns, testing, software architecture
---

#Part III: Putting it all together

In the last installment, we talked about using the Command Pattern to manage our business logic and leveraging Dependency Injection to load in all of our collaborator classes.  Inside our command, or context, we made a call to our repository.  We talked about what a repository does in Part I.  It centralizes all of our ORM-specific code into a single class that returned instances of our custom entity.  So what’s next?  Let’s take an overview of what we’ve done.

But first…

## Loose Ends

The original implementation of our repository didn’t use dependency injection.  That's because I wanted to focus on explaining exactly what a repository class is and how it works with an entity class.

Here's what our first pass at a repository looked like.

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


There are two classes that the repository depends on: the user model and our entity class.  Let’s revise our repository to inject those two dependencies.  To do this, we'll just override the default
"initialize" method that belongs to any Ruby class.


{% highlight ruby %}

class ARUserRepository

  def initialize(user_model, user_entity)
    @user_model = user_model
    @user_entity = user_entity
  end

  def create(first_name, last_name, email)
    ar_user_object = User.create!(first_name: first_name, last_name: last_name, email: email)
    user = translate_record(ar_user_object)
    user
  end

  private
 
  attr_reader :user_model, :user_entity

  def translate_record(record)
    user_entity.new(record.attributes)
  end

end

{% endhighlight %}


Now, there’s still some big stuff left out of this repository.  For instance, we’re not doing any error handling and the only thing this repository can do is to create users.  But the main structure
of the repository should be clear.


##The Call Chain

Now that we’ve got this giant collection of design patterns all working in concert, let’s walk through how they all interact.  At Sittercity we would test that everything is wired up correctly by using a Cucumber test.  Before we wrote any repository, entity, context, or factory, we would write a Cucumber tests that captured the process of creating a user.  That test would remain failing until we’d implemented everything else.  Afterwards, it should pass.  

Here’s what we’d see if we were dissecting the call chain.  Inside the controller, we’d see this:


{% highlight ruby %}

class UsersController

  def create
    context = UserFactory.create_user
    @user = context.call(params[:first_name], params[:last_name], params[:email])

    if @user
      redirect_to account_settings_path
    else
      render :new
    end
  end

end


{% endhighlight %}


The controller is doing two things. First, it's using the UserFactory to instantiate a our CreateUser context.  That context has to be instantiated with the correct dependencies and the factory takes care of that for us.  This allows the controller to make a pretty simple method call over to the factory and get what it wants.  Second, the controller is executing the context, by using the "call" method and passing along the appropriate arguments.

Let's dig int this first operation, where the factory constructs our context for us. Here's what he had originally written.


{% highlight ruby %}

module UserFactory

   self.create_user
    CreateUser.new(ARUserRepository.new, UserMailer.new)
   end

end

{% endhighlight %}


But we've changed the repository so that it too is using dependency injection.  So our final factory might look more like this.


{% highlight ruby %}

module UserFactory

   self.create_user
     CreateUser.new(user_repository, mailer)
   end

  private

   self.user_repository
     ARUserRepository.new(user_model, user_entity)
   end

  self.mailer
   UserMailer.new #our mailer
  end

  self.user_model
    User #our ActiveRecord model
  end

  self.user_entity
    UserEntity #our custom entity
  end

end

{% endhighlight %}


The factory is the lynchpin for the whole operation.  That’s where we all of our contexts get created with their real, live, collaborator classes (as opposed to the mocks we used in testing).

Now that our controller has an instance of the CreateUser context in its hands, let's take a look at what code is executed when the controller calls our context. In this case our business logic consists of creating a user in the database and sending
a “welcome” email. 


{% highlight ruby %}

class CreateUser
  def initialize(user_repository, mailer)
    user_repository = repo
    mailer = mailer
  end

  def call(first_name, last_name, email)
    user = user_repository.create(first_name, last_name, email)
    mailer.welcome_email(user.first_name, user.email)
    user
  end

  private
 
  attr_reader :user_repository, :mailer

end

{% endhighlight %}

Inside the call method, we can see that our context uses our repository, which has a pretty generic method called “create.”  The context expects that calling the "create" method will return some
object that it can call "first_name" and "email" on.  We know that object is our custom entity class.  Here’s what we came up with for our repository.


{% highlight ruby %}

class ARUserRepository

  def initialize(user_model, user_entity)
    @user_model = user_model
    @user_entity = user_entity
  end

  def create(first_name, last_name, email)
    ar_user_object = User.create!(first_name: first_name, last_name: last_name, email: email)
    user = translate_record(ar_user_object)
    user
  end

  private
 
  attr_reader :user_model, :user_entity

  def translate_record(record)
    user_entity.new(record.attributes)
  end

end

{% endhighlight %}

Our call chain starts in the controller, which relies on the factory to create an instance of our context.  Our controller then calls that context, being sure to pass it the appropriate arguments.

Inside our context, we pass the user attributes to our repository.  Our repository relies on an ORM (in this case, ActiveRecord) to create the record in the database and we take the result of that process and wrap it into our custom entity.  A populated instance of this entity is then returned to our controller.

##Why?

So even if we understand this call chain, we may have one lingering question, which is “Why would we ever to through all that trouble when Rails makes it so much easier to create a user?” 

The answer is: it depends. We might want it.  We might not.

Whether it makes sense to use all these patterns together depends on what you’re building.  If you’re in a 24 hour hackathon, you should just write your
code as quickly as possible.  Go with ActiveRecord. If you’re cranking out an MVP so you can validate an idea, then just write an idiomatic Rails app.  You shouldn't worry about all these design pattens. But once you’ve validated your idea, then it's time to think about how you're going to architect your
app for the long term.  At Sittercity, using these design patterns makes sense for us, given the length of time the business has been around and the rate at which we add features. But before you implement the architecture described here, you should understand your own business requirements.

When you're weighing wether to use this constellation of design patterns, try to balance the efforts with writing factories, contexts, repositories, and
entities against these benefits.

* **Freedom**  We want to use the best tools available.  Maybe that means a different ORM than ActiveRecord or maybe it means using another data store entirely.  If we decide that user info belongs in a document store, all we need  to do is to write a new repository.  We always want to have the freedom to use the best tools available for the job.
* **Testing**  At Sittercity, we pride ourselves on adding code through [BDD](http://en.wikipedia.org/wiki/Behavior-driven_development) and [TDD](http://en.wikipedia.org/wiki/Test-driven_development). That means we have thousands of tests for each project we work on.
  And when you have thousands of tests, you want them to run fast.  Testing a PORO (Plain Old Ruby Object) takes seconds--not minutes, whereas Rails test suites get bloated within a few months.
* **Avoid leaning on Rails**  We don't want to be held hostage by our web framework.  By minimizing and centralizing Rails-specific code, we don't have
  to worry about our web framework changing the rules on us.  Rails can make dramatic changes in how it operates and it won't effect us.  Plus, it
  leaves us free to upgrade to newer versions of Rails with minimal effort. 
* **Speedy features Development**  Yes, it's a pain to setup a repo, entity, context, and factory when you've got one feature.  But you can expect to write more and more features with more and more complexity as your app lives on. Sure, creating a user is easy now, but a year from now we may be creating a user in the DB, sending a welcome email, charging a credit card, sending a text message about the charge, and initiating a call from our customer service to the new user.  We can easily extend the functionality of the CreateUser context without effecting any other part of the system. 

Many of these benefits are ones that are not visible when working on one single feature.  But as features add up and your app becomes more and more
complex, you will be grateful that you have a system in place for keeping the chaos at bay.  At Sittercity, we’re supporting a decade’s worth of
business requirements and we’re adding new features every week.  The end goal is always to write high-quality software that is resilient to all the changes that you will inevitably confront.   Good luck out there.
