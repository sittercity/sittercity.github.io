---
layout:post
title:'Design Patterns, Fast Tests, and Robust Software, Part II'
date:2015-02-23 12:01:00
categories: design patters, testing, software architecture
---

<meta charset="utf-8">
<!-- not sure this charset isn't pulled in from _includes --> 

# Part II: Commands, Dependency Injection, Factories

In part I, we talked about using using the Repository and Entity patterns in order to centralize our ORM-specific code.  In this installment, we’ll talk about three other design patterns.  We’ll talk about Command Pattern, which we’ll use to do a lot of the heavy lifting in our app.  And we’ll rely on Dependency Injection to make our life easy when it comes to testing.  Finally, we’ll talk about having our Rails controller interface with a Factory that sets up out context for us.

## Commands and Dependency Injection

Many web apps start small, but the successful ones end up lasting years.  As an app goes through its lifecycle, it accumulates feature after
feature.  In a Rails app, models that were only 30 lines long quickly grow into 300 lines.  It's suddenly difficult to navigate all that terrain.
At SitterCity we prefer to write many small commands, or contexts, rather than stuffing all business logic into the model.  These context classes
will be initialized with any other classes that the context might need.  Within the Rails system, we end up with barebones models that simple manage
validations and the relationships to other models (has\_many or  belongs\_to code).  And our software consists of a bunch of Plain Old Ruby Objects that capture all the business logic involved in our app.

Let's write a context that's creates a user and sends them a welcome email.  We'll construct the class with the repository that we wrote (since
that's what's responsible for creating users in the database) and a mailer class (let's imagine that it's an instance of ActionMailer that has a
method on it named "welcome\_email").

We might end up with a context like this.

{% highlight ruby %}

class CreateUser
  def initialize(repo, mailer)
    @repo = repo
     @mailer = mailer
  end

  def call(first_name, last_name, email)
    user = @repo.create(first_name, last_name, email)
    @mailer.welcome_email(user.first_name, user.email)
    user
  end

end

{% endhighlight %}

To use this context, we’d instantiate our context with the colloborator classes we need.

{% highlight ruby %}

context = CreateUser.new(ARRepository.new, UserMail.new)
context.call(first_name, last_name, email)

{% endhighlight %}

There are two big benefits to using dependency injection here.  First, if we decide we like Sequel more than ActiveRecord, it'd be easy to just swap in a different repository and have the class behave exactly the same way.  The code below should work the same as the code above.

{% highlight ruby %}

context = CreateUser.new(SequelRepository.new, UserMail.new)
context.call(first_name, last_name, email)

{% endhighlight %}

The second big benefit is testing.  In my tests for the class, I don't have to inject the class with any Rails objects, like ActionMailer.  I can just use a mock object.  In the code below, I’ve setup two mock objects: one for the repositiory and one for the mailer. 

{% highlight ruby %}

require  'create_user'

described CreateUser do
  let(:repo) { double(:repo) }
  let(:mailer) { double(:mailer) }

  subject { described_class.new(repo, mailer) }

end

{% endhighlight %}

Our tests will verify that the appropriate methods get called on the repo and the mailer.  This give us very granular control over testing the business logic in our class.  Namely, we don’t have to worry about problems in the collaborator classes negatively effecting our tests.  If there’s a problem in those collaborator classes it will be either their unit tests or the integration tests that reveal it.  But what we want in the tests for the CreateUser class is laser focus on our own business logic. This may not be complex logic now, but as the app grows, it can become elaborate.

The second benefit is that tests run fast.  When you compare testing a PORO to running a test that depends on ActiveRecord, you’re going to find that it takes forever to load all of ActiveRecord.  The tests for your PORO class will be done running before you’re even done requiring ActiveRecord in your Rails class.  It’ll take even longer if you actually want to have your tests create records in the database.  You’d have to initialize your app everytime you want to run your tests.  You’ll find that this ultimately disincentivizes you from maintaining a thorough tests suite. On the other hand, when your unit tests finish in seconds and you’re able to dive deep into the process of hammering out your business logic through tests, you’re going to enjoy the process much more.

Using Dependency Injection to instantiate the contexts or commands that manage your business logic is a big step in both gaining control over your software and laying the foundation for good test coverage.  And given the fact that you'll be adding tests along with new feature, this strategy pays dividends over the lifespan of an app.


## Factories

In the same way that we used a repository to consolidate all ActiveRecord method calls in one class, it's a good idea to consolidate all of your contexts inside a single file. A factory, often called a
constructor, has a pretty simlple job: contruct instances of classes.  So we'll use our factory to take care of doing actually injecting in our dependencies to the contexts we want to call.  That way,
the classes that want to execute the context don't need to worry about the other classes that might be required for that context--all they need to know is how what method to call on the context (in our
case it's "call").

Among all the different classes we've created, our UserFactory alone is the only class that actually requires all the classes that we care about. It needs to require the files for the User model, the
mailer, the repository, and the entity classes.  One consequence of requiring all these files is that the tests for the Factory tend to run a slower than when we’re testing classes that get to
dependencies passed on initialization (but they'll still be faster than if we booted up the entire app).

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

You might notice that this doesn't look exactly like a typical test for a ruby class.  At SitterCity, we write factory classes that are responsible for instantiating our contexts.  We adopt the convention of keeping these factories as modules rather than as full-fledged classes.

In order to make this test pass, we would end up writing the following code for our factory.

{% highlight ruby %}

require ‘create_user’
require ‘mailer’
require ‘AR_user_repository’

module UserFactory

   self.create_user
    CreateUser.new(ARUserRepository.new, UserMailer.new)
   end


  private

end

{% endhighlight %}

The only thing the factory does is to make it easy to get an instance of our context.   

Back in our controller, we'll have something like this:

{% highlight ruby %}

class UsersController
  def create
    context = UserFactory.create_user
    user = context.call(params[:first_name], params[:last_name], params[:email])
  end
end

{% endhighlight %}

Despite the fact that creating a user involves a context class, our repository, out entity, the User model, and the mailer class, our controller
only need to know two things.  First, it needs to know that the UserFactory has a class method named "create\_user" and second, it needs to know
that the context for creating user has one public method named “call,” which accepts three arguments (last\_name, first\_name, and email).  The controller can be blissfully unaware of everything beyond those two things.
