---
layout:post
title:'Design Patterns, Fast Tests, and Robust Software, Part III'
date:2015-02-23 12:01:00
categories: design patters, testing, software architecture
---

<meta charset="utf-8">
<!-- not sure this charset isn't pulled in from _includes --> 

#Part III: Putting it all together

In the last installment, we talked about using the Command Pattern to store our business logic and leveraging Dependency Injection to load in all of our collaborator classes.  In the first installment, we had centralized all of our ORM-specific code into a repository that returned instances of our custom entity.

##Tying up Loose Ends

The original implementation of our repository didn’t use dependency injection, but now that we undstand it’s benefits, we may as well update our repository.  

Here’s how we originally setup the repository.

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


I wrote it this way so that I could highlight the how the repository and entity work together, but now that we understand that, we may as well make this a full-fledged repository.  There are two classes that the repository depends on: the user model and our entity class.  Let’s revise our repository to inject those two dependencies into the “initialize” method.


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


Now, there’s still some big stuff left out of this repository.  For instance, we’re not doing any error handling and the only the the repository can do is to create users.  But the main structure and intent of the repository should be clear.


##The Call Chain

Now that we’ve got this giant collection of design patterns all working on concert, let’s walk through how they all interact.  At SitterCity we would test this everything is wired up correctly by using a Cucumber test.  Before we wrote any repository, entity, context, or factory, we would write a Cucumber tests that captured the process of creating a user.  That test would remain failing until we’d implemented everything else.  Afterwards, it should pass.  

Here’s what we’d see if we were dissecting the call chain.  Inside the controller, we’d see this:


{% highlight ruby %}

class UsersController

  def create
    context = UserFactory.create_user
    user = context.call(params[:first_name], params[:last_name], params[:email])
  end

end


{% endhighlight %}



And UserFactory would be the only place that actually requires all the classes that we care about, such as the User model, the mailer, the repository, and the entity.  One consequence of requiring all these files is that the tests for the Factory tend to run a slower than when we’re testing classes that get to dependencies passed on initialization.

Out tests for the factory are pretty basic.  Since the factory only creates new instances of things, we just need to verify that it’s working properly.  Something like this should be sufficient for a unit test.

{% highlight ruby %}

describe UserFactory do

  it 'constructs a create_user context' d
      expect(
     described_class.create_user
     ).to be_a CreateUser
  end

end

{% endhighlight %}


In order to make this test pass, we would end up writing the following code for our factory.


{% highlight ruby %}

require ‘create_user’
require ‘user_entity’
require ‘mailer’
require ‘AR_user_repository’

module UserFactory

   self.create_user
     CreateUser.new(user_repository, mailer)
   end


  private

   self.user_repository
     ARUserRepository.new(user_model, user_entity)
   end

  self.mailer
   UserMailer.new
  end

  self.user_model
    User
  end

  self.user_entity
    UserEntity
  end

end

{% endhighlight %}


The factory is the lynchpin for the whole operation.  That’s where we inject our custom classes with the real live collaborator classes they rely on (as opposed to the mocks we used in testing).

When the controller calls the context constructed by this factory, we’ll get to see our business logic executed.  In this case our business logic consists of creating a user in the database and sending a “welcome” email.  


{% highlight ruby %}

class CreateUser
  def initialize(user_repository, mailer)
    user_repository = repo
    mailer = mailer
  end

  def call(first_name, last_name, email)
    user = user_repository.create(first_name, last_name, email)
    mailer.welcome_email(first_name, email)
    user
  end

  private
 
  attr_reader :user_repository, :mailer

end

{% endhighlight %}

Looking inside the call method, we can see that our context uses our repository, which has a pretty generic method called “create” and returns instances of our entity class. 



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

Our call chain starts in the controller, which relies on the factory to create an instance of our context.  Our controller then calls that context, being sure to pass it the appropriate arguments.  Inside our context, we pass along the user attributes to our repository.  Our repository relies on an ORM to create the record in the database and we take the result of that process and wrap it into our custom entity.  

##Why?

So now the big question is “Why would we ever to through all that trouble, when Rails makes it so much easier to create a user?” 

The only good answer is: it depends.

Whether it makes sense to use all these patterns together depends on what you’re building.  If you’re in a 24 hour hackathon, you should just write your code as quickly as possible.  Or if you’re taking a few days to crank out an MVP, then go ahead of an just write an idiomatic Rails app.  Once you’ve validated your idea, then it may be time to bring out the big guns and start using all these design patterns.  At SitterCity, we’re supporting a decade’s worth of business requirements and we’re adding new features every week.  As a web app ages, entropy tends to seep in from the edges.  By using the collection of design patterns outlined here, SitterCity has managed to minimize the effects of that entropy.  This approach makes sense for us, but you should understand your own business requirements and goals when considering whether this is right for you.

Here are some of the benefits from adopting this approach:

* Freedom.  We want to use the best tools available.  Maybe that means a different ORM than ActiveRecord or maybe it means using another data store entirely.  If we decide that user info belongs in a   document story, we can throw everything in MongoDB and all we need  to do is to write a repository capable of making objects with Mongoid.  But we always want to have the freedom to use the best tools available for the job.
* Testing.  SitterCity prides itself on adding code through BDD and TDD. That means we have thousands of tests for each project we work on.  When you have thousands of tests, you want them to run fast.  Testing a PORO (Plain Old Ruby Object) takes seconds not minutes, whereas many Rails tests suits get bloated after a few months.
* Avoid leaning on Rails.  Rails can make dramatic changes in how it operates, but it will have minimal effect on our app because our Rails-dependent code has been centralized into a few small classes.  That allows us to upgrade to the newer versions of Rails with minimal effort, because our code doesn’t rely on Rails.
* Add features quickly.  Yes, it's a pain to setup a repo, entity, context, and factory when you've got one feature.  But you can expect to write more and more features with more and more complexity as your app lives on. Sure, creating a user is easy now, but a year from now we may be creating a user in the DB, sending a welcome email, charging a credit card, sending a text message about the charge, and initiating a call from our customer service to the new user.  We can easily extend the functionality of the CreateUser context without effecting any other part of the system. 

Many of these benefits are ones that are not visible when working on one single feature.  But as features add up and your app becomes more and more complex, you will be grateful that you have a system in place for keeping the chaos at bay.  Ultimately, this will allow you to add more features quickly and with ease.  And that’s a lot better than being caught in refactor hell, where each additional feature you write requires you restructure large portions of your original implementation.  The end goal is always to write high-quality software that is resilient to all the changes that you will inevitably confront.   Good luck out there.
