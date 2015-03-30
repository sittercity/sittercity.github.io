---
layout: post
title: "Design Patterns, Fast Tests, and Robust Software, Part II"
date: 2015-03-23 12:01:00
categories: design patterns, testing, software architecture
---

# Part II: Commands, Dependency Injection, and Factories

In part I, we talked about using the Repository and Entity patterns in order to centralize our ORM-specific code.  In this installment, we’ll talk about three other design patterns.  We’ll talk about
the Command Pattern, which we'll leverage to create contexts or use-cases that manage the complex business logic in our app.  And we’ll rely on Dependency Injection to make our life easy when it comes to testing.  Finally, we’ll talk about having our Rails controller interact with a factory that sets up our context for us, rather than having the controller instantiate the context itself.

## Commands and Dependency Injection

Many web apps start small, but the successful ones end up lasting years.  As an app goes through its lifecycle, it accumulates feature after
feature.  In a Rails app, models that were only 30 lines long quickly grow into 300 lines.  It's suddenly difficult to navigate all that terrain.  And once we have complex events where many models
interact with one another, we're going to start to feel stifled by the rigid MVC architecture that once made development easy.

At Sittercity we prefer to write many small commands, or contexts, rather than stuffing all business logic into the model.   Often times we need a few classes to work together in a context.  So we use a strategy called Dependency Injection.  With this pattern, we initialize our class with any of the other classes that will be used in our context—we’re sowing these classes into the fabric of this context.

Let’s write a context that creates a user and sends them a welcome email.  We’re going to want to inject the classes that our context depends on:

1. a class capable of creating users in the database

2. a class capable of sending emails

For the first item above, we’re going to pass in an instance of our repository.  And for the second item, let’s imagine that our app has a mailer class that inherits from from ActionMailer.  We’ll pass in this mailer to our context too.

{% highlight ruby %}

class CreateUser
  def initialize(repo, mailer)
    @repo = repo
    @mailer = mailer
  end

end

{% endhighlight %}

As you can see, the context is initialized with instances of any class that it will need to collaborate with in order to run.

Moving all the business logic into contexts or use-cases will ultimately change the saw that our Rails model classes look.   We end up with barebones models that simply manage
validations and the relationships to other models (has\_many or  belongs\_to code).  And our software consists of a bunch of Plain Old Ruby Objects (POROs), which is a big win.  For more on POROs,
checkout [this post](http://blog.steveklabnik.com/posts/2011-09-06-the-secret-to-rails-oo-design).

So what will our context look like?  Well, we know it has an initialize method on it.  Beyond that, we only want one public method.  With the command pattern, it’s often something like “execute,” “call”, or “run.”  Any experienced developer that sees a class with a public method like this will immediately know that you’ve using the command pattern.  

We’re going to pass some arguments to the “call” method.  The idea here is that the collaborator classes are passed in to the context when the class instantiates, but when it comes to the things that change we’ll pass those to the call method.

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

To use this context, we’d instantiate it with the collaborator classes we need.

{% highlight ruby %}

context = CreateUser.new(ARRepository.new, UserMail.new)
context.call(first_name, last_name, email)

{% endhighlight %}

There are two big benefits to using dependency injection here.  First, if we decide we like Sequel more than ActiveRecord, it’s  easy to just swap in a different repository and have the class behave exactly the same way.  The code below should work the same as the code above.

{% highlight ruby %}

context = CreateUser.new(SequelRepository.new, UserMail.new)
context.call(first_name, last_name, email)

{% endhighlight %}

The second big benefit is testing. At Sittercity we have a lot of tests.  We want them to be high quality tests that run fast.  Using dependency injection helps us achieve both these goals.

In my tests for the class, I don't have to inject the class with any Rails objects, like ActionMailer.  I can just use a mock object.  In the code below, I’ve setup two mock objects: one for the repository and one for the mailer. 

{% highlight ruby %}

require  'create_user'

described CreateUser do
  let(:repo) { double(:repo) }
  let(:mailer) { double(:mailer) }

  subject { described_class.new(repo, mailer) }

end

{% endhighlight %}

Our tests will verify that the appropriate methods get called on the repo and the mailer. 

{% highlight ruby %}

	described CreateUser do
	  let(:repo) { double(:repo) }
	  let(:mailer) { double(:mailer) }
		let(:first_name) { ‘john’ }
    let(:last_name) { ‘doe }
    let(:email)  { ‘someEmail@example.com’ }

	  it ‘creates a user’ do
			expect(repo).to receive(:create!).with(
				first_name,
        last_name,
        email
			)

	    subject.call(first_name, last_name, email)
	  end
	end

{% endhighlight %}

This give us very granular control over testing the business logic in our class.  Namely, we don’t have to worry about problems in the collaborator classes negatively effecting our tests.  For instances, if there’s a problem with how we’ve created our repository, either their unit tests for that class or the integration tests that reveal it.  But what we want in the tests for the CreateUser class is laser focus on our own business logic. This may not be complex logic now, but as the app grows, it will become complex.

The second benefit is that tests run fast.  When you compare testing a PORO to running a test that traditional Rails test you’re going to find that it takes forever to load all of ActiveRecord. The tests for your PORO class will be done running before your class has even loaded ActiveRecord.  

It’ll take even longer if you actually want to have your tests create records in the database.  You’d have to initialize your app every time you want to run your tests.  You’ll find that this ultimately disincentivizes you from maintaining a thorough test suite. On the other hand, when your unit tests finish in seconds and you’re able to dive deep into the process of hammering out your business logic through tests, you’re going to enjoy the process much more.

Using Dependency Injection to instantiate the contexts or commands that manage your business logic is a big step in both gaining control over your software and laying the foundation for good test coverage.  And given the fact that you'll be adding tests along with new feature, this strategy pays dividends over the lifespan of an app.


## Factories

In the same way that we used a repository to consolidate all ActiveRecord method calls in one class, it's a good idea to consolidate all of your contexts inside a single file. A factory, often called a
constructor class, has a pretty simple job: construct instances of classes.  So we'll use our factory to take care of doing actually injecting in our dependencies to the contexts we want to call.  That way,
the classes that want to execute the context don't need to worry about the other classes that might be required for that context--all they need to know is how what method to call on the context (in our
case it's "call").

Among all the different classes we've created, our UserFactory alone is the only class that explicitly requires all the classes that we care about.  At the top the factory class, you will see 

{% highlight ruby %}

  require ‘create_user’

{% endhighlight %}

The factory needs to require the files for the User model, the
mailer, the repository, and the entity classes.  One consequence of requiring all these files is that the tests for the Factory tend to run a slower than when we’re testing classes that get to
dependencies passed on initialization (but they'll still be faster than if we booted up the entire app).

Our tests for the factory are pretty basic.  Since the factory only creates new instances of things, we just need to verify that it’s working properly.  Something like this should be sufficient for a unit test.

{% highlight ruby %}

describe UserFactory do

  it 'constructs a create_user context' d
      expect(
     described_class.create_user
     ).to be_a CreateUser
  end

end

{% endhighlight %}

Some developers don’t even think it’s worth writing test for factories, since tests are meant to verify behavior and factories have no behavior.  It’s really up to you as to what you feel is best for your app.

At Sittercity, we write factory classes that are responsible for instantiating our contexts.  We adopt the convention of keeping these factories as modules rather than as full-fledged classes.

In order to make our test pass, we would end up writing the following code for our factory.

{% highlight ruby %}

require ‘create_user’
require ‘mailer’
require ‘AR_user_repository’

module UserFactory

   self.create_user
    CreateUser.new(ARUserRepository.new, UserMailer.new)
   end

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

Despite the fact that creating a user involves a context class, our repository, our entity, the User model, and the mailer class, our controller
only need to know two things.  First, it needs to know that the UserFactory has a class method named "create\_user" and second, it needs to know
that the context for creating user has one public method named “call,” which accepts three arguments (last\_name, first\_name, and email).  The controller can be blissfully unaware of everything beyond those two things.

## Conclusion

	Our use of the command pattern plays an integral role in decoupling our software of the web framework.  We no longer rely on business logic that lives inside Rails models.  Instead, we have flexible and robust contexts that describe what our app does.  These are not bound to any particular model.  Instead, they are constructed with any dependencies they might have. Using dependency injection allows for this, while also providing an opportunity to write screaming fast tests.  Finally, we pull it all together with a factory that any class can easily call.  We’re laying down a foundation that will give us back control of our software and make it resilient to the changes that await it.
