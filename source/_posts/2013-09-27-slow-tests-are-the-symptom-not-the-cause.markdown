---
layout: post
title: "Slow Tests are the Symptom, not the Cause"
date: 2013-09-27 10:02
comments: true
categories:
published: true
---

If you have a slow test suite and you are asking yourself "how can I make my tests faster?" then you are asking the wrong question. Most chances are that you have bigger problems than just slow tests. The test slowness is merely the symptom; what you should really address is the cause. Once the real cause is addressed you will find that it's easy to write new fast tests and straightforward to refactor existing tests.

<!-- more -->

It's surprising how quickly a rails app's test suite can become slow. It's important to understand the reason for this slowness early on and address the real cause behind it. In most cases the reason is excessive *coupling* between between the domain objects objects themselves and coupling between these objects and the framework.

In this refactoring walk-through we will see how small, incremental improvements to the design of the app, and specifically, *decoupling*, naturally lead to faster tests. We will extract service objects, completely remove all rails dependencies in test time and otherwise reduce the amount of coupling in the app.

Our goal is to have a simple, flexible and easy to maintain system in which objects can be replaced with other objects with minimal code changes. We will strive to achieve this goal and observe the effect of it on our tests speed.

Starting With A Fat Controller
--------------
Suppose we have a controller that's responsible for handling users signing up for a mailing list:

```ruby
class MailingListsController < ApplicationController
  respond_to :json
  def add_user
    user = User.find_by!(username: params[:username])
    NotifiesUser.run(user, 'blog_list')
    user.update_attributes(mailing_list_name: 'blog_list')
    respond_with user
  end
end
```
We first find the user (an exception is raised if the user is not found). Then we notify the user she was added to the mailing list via ```NotifiesUser``` (probably asking her to confirm). We update the user record with the name of the mailing list and then hand the ```user``` object to ```respond_with```, which will render the json representation of the user or the proper error response in case saving of the object failed.

The logic here is pretty straight-forward, but it's still too complicated for a controller and should be extracted out. But where to? The word ```user``` in every line in this method suggests that we should push it into the ```User``` model (that's called [Feature Envy](http://sourcemaking.com/refactoring/feature-envy)). Let's try this:

Extracting Logic to a Fat Model
--------------
```ruby
class MailingListsController < ApplicationController
  respond_to :json
  def add_user
    user = User.add_to_mailing_list(params[:username], 'blog_list')
    respond_with user
  end
end

```
```ruby
class User < ActiveRecord::Base
  validates_uniqueness_of :username

  def self.add_to_mailing_list(username, mailing_list_name)
    user = User.find_by!(username: username)
    NotifiesUser.run(user, 'blog_list')
    user.update_attributes(mailing_list_name: 'blog_list')
    user
  end
end
```
This is better: the ```User``` class is now responsible for creating and updating users. But there is a problem: now ```User``` is handling mailing list additions, as well as user notifications. These are too many responsibilities for one class. Having an active record object handle anything more than CRUD, associations and validations is a (further) violation of the [Single Responsibility Principle](http://en.wikipedia.org/wiki/Single_responsibility_principle).

The result is that business logic in active record classes is a pain to unit test. You often need to use factories or to heavily stub out methods of the object under test (don't do that), stub all instances of the class under test (don't do that either) or hit the database in your unit tests (please don't). As a result, testing active record objects can be very slow, sometimes orders of magnitude slower than testing plain ruby objects.

Now, if the code above was the entire ```User``` class and my application was small and simple I might have been happy with leaving ```User#add_to_mailing_list``` as is. But in a bit bigger rails apps that are not groomed often enough, models, controllers and domain logic tend to get tangled (coupled) together and needlessly complicate things (Rich Hickey, the inventor of clojure, calls it *incidental complexity*). This is when introducing a [service object](http://blog.codeclimate.com/blog/2012/10/17/7-ways-to-decompose-fat-activerecord-models/) is helpful:

Extracting a Service Object
--------------
```ruby
class MailingListsController < ApplicationController
  respond_to :json
  def add_user
    user = AddsUserToList.run(params[:username], 'blog_list')
    respond_with user
  end
end
```
```ruby
class AddsUserToList
  def self.run(username, mailing_list_name)
    user = User.find_by!(username: username)
    NotifiesUser.run(user, 'blog_list')
    user.update_attributes(mailing_list_name: 'blog_list')
    user
  end
end
```
We created a plain ruby object, ```AddsUserToList```, which contains the business logic from before. In the controller we call this object and not ```User``` directly.
This is an improvement, but hard-coding the name of the class of your collaborator is a bad idea since it couples the two together and makes it impossible to replace the class with a different implementation. Not surprisingly, the result of this coupling is that testing becomes harder and tests slower. Testing this service object would require us to somehow stub ```User#find_by!``` to avoid hitting the database, and probably also stub out ```NotifiesUser#run``` in order to avoid sending a real notification out.

Also, referencing the class ```User``` directly means that our unit tests will have to load active record and the entire rails stack, but even worse - the entire app and its dependencies. This load time can be a few seconds for trivial rails apps, but can sometimes be 30 seconds for bigger apps. Unit tests should be *fast* to run as part of your test suite but also fast to run individually, which means they should not load the rails stack or your application (also see [Corey Haines's talk](http://www.youtube.com/watch?v=bNn6M2vqxHE)
 on the subject).

The most straight forward way to decouple the object from its collaborators is to *inject the dependencies* of ```AddsUserToList```:

Injecting Dependencies
--------------
```ruby
class AddsUserToList
  def self.run(username, mailing_list_name, finds_user = User, notifies_user = NotifiesUser)
    finds_user.find_by!(username: username)
    notifies_user.(user, mailing_list_name)
    user.update_attributes(mailing_list_name: mailing_list_name)
    user
  end
end
```
We can now pass as an argument any class that finds a user and any class that notifies a user, which means that passing different implementations will be easy. It also means that testing will be easier. Since we supplied reasonable defaults we don't need to be explicit about these dependencies if we don't change them, and our controller can stay unchanged.

The fact that we are specifying ```User``` as the default value of finds_user in the parameter list does *not* mean that this class and all its dependents (ActiveRecord, our app and other gems) will get loaded. Ruby's *Deferred Evaluation* of the default values means that if these default values are not needed they will not get loaded, so we can run this unit test without loading rails.

Simplifying the Interface
----------------------------------

The method ```AddsUserToList#run``` receives 4 arguments. Users of this method need to know the *order* of the list. Also, it is likely that over time you'd discover you need to add more arguments. When this happens you will need to update all users of the method. A more flexible solution is to use a hash of arguments. This will make the interface more stable and ensure the number of arguments does not grow when we find that we need to add more arguments. It will also make refactoring a little easier, which is important. I often find that for many classes I end up changing from an argument list to a hash of options at some point, so why not [use it in the first place](http://www.poodr.com/)? But does it mean that we need to give up the advantages of deferred evaluation of the default values? Not at all.

We will use ```Hash#fetch```, passing a block to it, which will not get evaluated unless the queried key is absent. In our tests, the code in the block to ```fetch``` will never get evaluated, and ```User``` won't get loaded. Also, when specifying the defaults in the argument list it is not possible to evaluate more than one statement, but we can do it using ```Hash#fetch```.

One more thing: when my classes contain only one public method I don't like calling it ```run```, ```do``` or ```perform``` since these names don't convey a lot of information. In this case I'd rather call it ```call``` and use ruby's shorthand notation for invoking this method. This also enables me to pass in a proc instead of the class itself if I need it.

```ruby
class AddsUserToList
  def self.run(args)
    finds_user = args.fetch(:finds_user) { User }
    notifies_user = args.fetch(:notifies_user) { NotifiesUser }

    finds_user.find_by!(username: args.fetch(:username))
    notifies_user.(user, args.fetch(:mailing_list_name))
    user.update_attributes(mailing_list_name: args.fetch(:mailing_list_name))
    user
  end
end
```

The end result looks like this:

The Complete Refactoring
-------------------

Before:
```ruby
class MailingListsController < ApplicationController
  respond_to :json
  def add_user
    user = User.find_by!(username: params[:username])
    NotifiesUser.run(user, 'blog_list')
    user.update_attributes(mailing_list_name: 'blog_list')
    respond_with user
  end
end
```
After:
```ruby
class MailingListsController < ApplicationController
  respond_to :json
  def add_user
    user = AddsUserToList.(username: params[:username], mailing_list_name: 'blog_list')
    respond_with user
  end
end
```
```ruby
class AddsUserToList
  def self.call(args)
    finds_user = args.fetch(:finds_user) { User }
    notifies_user = args.fetch(:notifies_user) { NotifiesUser }

    finds_user.find_by!(username: args.fetch(:username))
    notifies_user.(user, args.fetch(:mailing_list_name))
    user.update_attributes(mailing_list_name: args.fetch(:mailing_list_name))
    user
  end
end
```
The Tests
--------
The class ```AddsUserToList``` can be tested using *true*, isolated unit tests: we can easily isolate the class under test and make sure it properly communicates with its collaborators. There is no database access, no heavy handed request stubbing and if we want to - no loading of the rails stack. In fact, I'd argue that any test that requires any of the above is not a unit test, but rather an integration test (see entire repo [here](https://github.com/orend/register)).

```ruby
describe AddsUserToList do
  let(:finds_user) { double('finds_user') }
  let(:notifies_user) { double('notifies_user') }
  let(:user) { double('user') }
  subject(:adds_user_to_list) { AddsUserToList }

  it 'registers a new user' do
    expect(finds_user).to receive(:find_by!).with(username: 'username').and_return(user)
    expect(notifies_user).to receive(:call).with(user, 'list_name')
    expect(user).to receive(:update_attributes).with(mailing_list_name: 'list_name')

    adds_user_to_list.(username: 'username', mailing_list_name: 'list_name', finds_user: finds_user, notifies_user: notifies_user)
  end
end
```

Here we pass in mocks (initialized with ```#double```) for each collaborator and expect them to receive the correct messages. We do not assert any values - specifically not the value of ```user.mailing_list_name```. Instead we require that ```user``` receives the ```update_attributes``` method. We need to *trust* ```user``` to update the attributes. After all, that's a unit test for ```AddsUserToList```, not for ```user```.

As you can see there is a close resemblance between the test code and the code it is testing. I don't see it as a problem. A unit test should verify that the object under test sends the correct messages to its collaborators, and in the case of ```AddsUserToList``` we have a controller-like object, and a controller's job is to... coordinate sending messages between collaborators. Sandi Metz talks about what you should and what you should not test [here](http://www.confreaks.com/videos/2452-railsconf2013-the-magic-tricks-of-testing). To use her vocabulary, all we are testing here are outgoing command messages since these are the only messages this object sends. For that reason I think the resemblance is acceptable.

I omit the controller and integration tests here, but [please don't forget them](http://solnic.eu/2012/02/02/yes-you-should-write-controller-tests.html) in your code. They will be much simpler and there will be fewer of them if you extract service objects.

Some Numbers
----------

How much faster is this test from a unit test that touches the database and loads rails and the application? Here are the results:

|               |  Single Test Runtime | Total Suite Runtime  |
| ------------- |:-------------:| :-----:|
| **'false' unit test**        |   0.0530s     |   2.5s
| **true unit test**         |   0.0005s     |   0.4s

<br>

A single test run is roughly **a hundred times faster**. The absolute times are rather small but the difference will be very noticeable when you have hundreds of unit tests or more. The total runtime in the "false" version takes roughly two seconds longer. This is the time it takes to load a trivial rails app on my machine. This will be significantly higher when the app grows in size and adds more gems.

Conclusion
----------

The 'before' version's tests are harder to write and are significantly slower since we bundle many responsibilities into a single class, the controller class. The 'After' version is easier to test (we pass mocks to override the default classes). This means that in our code in ```AddsUserToList``` we can easily replace the collaborators with other implementations in case the requirements change and require no or little code change. The controller has been reduced to performing the most basic task of coordination between a few objects.

Is the 'After' version better? I think it is. It's easier and faster to test, but even more importantly the collaborators are clearly defined and are treated as *roles*, not as specific implementations. As such, they can always be replaced by different implementations of the role they play. We now can concentrate on the *messages* passing between the different *roles* in our system.

When you practice TDD with mock objects you will almost be forced to inject your dependencies in order to mock collaborators. Extracting your business logic into service objects makes all this much easier, and further decoupling from active record makes the tests true unit tests that are also blazing fast.

This brings us closer to a lofty design goal stated by Kent Beck:

>"When you can extend a system solely by adding new objects without modifying any existing objects, then you have a system that is flexible and cheap to maintain."

Using mocks and dependency injection with TDD makes sure your system is designed for this form of modularity from the get go. You know you can replace your objects with a different implementation because this is exactly what you did in your tests when you passed in mocks. Such design guarantees that you can write true, isolated and thus fast, tests.

P.S., if you found this post useful be sure to sign up for the [newsletter](http://eepurl.com/FF4ET) to get tips about how to improve your code.

<hr>

I'd like to thank the following people for providing useful feedback about this post: Frazer Horn, Steve Klabnik, Peter Marreck, Susan Potter and Piotr Solnica.

