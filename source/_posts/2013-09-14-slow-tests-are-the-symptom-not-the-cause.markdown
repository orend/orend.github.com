---
layout: post
title: "Slow Tests Are the Symptom, Not the Cause"
date: 2013-09-14 22:34
comments: true
categories:
---

**TL;DR:** Extract service objects and completely remove rails dependencies in tests to achieve the fastest possible tests, but more importantly - a better design. Skip to [The Complete Refactoring](#[the-complete-refactoring]) section if you only want to see the refactoring.

<!--more-->
Starting With A Fat Controller
--------------
Suppose you have a controller that's responsible for handling users signing up for a mailing list:

```ruby
class EmailListsController < ApplicationController
  respond_to :json
  def create
    user = User.find_or_create_by(username: params[:username])
    NotifiesUser.run(user, 'blog_list')
    user.update_attributes(email_list_name: 'blog_list')
    respond_with user
  end
end
```
We first find the user or create it if it doesn't exist. Then we notify the user she was added to the mailing list via ```NotifiesUser``` (probably asking her to confirm). We update the user record with the name of the mailing list and then hand the ```user``` object to ```respond_with```, which will render the json representation of the user or the proper error response in case saving of the object failed.

The logic here is pretty straight-forward, but it's still too complicated for a controller and should be extracted out. But where to? The word ```user``` in almost every line here suggests that we should push it into the ```User``` model (that's [Feature Envy](http://sourcemaking.com/refactoring/feature-envy)). Let's try this:

Extracting Logic to a Fat Model
--------------
```ruby
class EmailListsController < ApplicationController
  respond_to :json
  def create
    user = User.addToEmailList(params[:username], 'blog_list')
    respond_with user
  end
end

```
```ruby
class User < ActiveRecord::Base
  validates_uniqueness_of :username

  def self.addToEmailList(username, email_list_name)
    user = User.find_or_create_by(username: username)
    NotifiesUser.run(user, 'blog_list')
    user.update_attributes(email_list_name: 'blog_list')
    user
  end
end
```
This is better: the ```User``` class is now responsible for creating and updating users. But there are still a few problems. The first one is that now ```User``` is handling mailing list additions, as well user notifications. These are too many responsibilities for one class. Having an active record object handle anything more than CRUD and associations is a violation of the [Single Responsibility Principle](http://en.wikipedia.org/wiki/Single_responsibility_principle).

The second problem is that business logic in active record classes is a pain to unit test. You often need to use factories or to heavily stub out methods of the object under test (don't do that), stub all instances of the class under test (don't do that either) or hit the database in your unit tests (please don't). As a result, testing active record objects can be very slow, sometimes orders of magnitude slower than testing plain ruby objects.

Now, if the code above was the entire ```User``` class and my application was small and simple I'd be perfectly happy with leaving ```User#addToEmailList``` as is. But more complex rails apps that are not groomed often enough tend to have large 'god classes' such as ```User``` or ```Order``` that attract every piece of logic that touches the model. Slow tests make an app harder to maintain and harder to work with. This is when introducing a [service object](http://blog.codeclimate.com/blog/2012/10/17/7-ways-to-decompose-fat-activerecord-models/) is helpful:

Extacting a Service Object
--------------
```ruby
class EmailListsController < ApplicationController
  respond_to :json
  def create
    user = AddsUserToList.run(params[:username], 'blog_list')
    respond_with user
  end
end
```
```ruby
class AddsUserToList
  def self.run(username, email_list_name)
    User.find_or_create_by(username: username).tap do |user|
      NotifiesUser.run(user, 'blog_list')
      user.update_attributes(email_list_name: 'blog_list')
    end
  end
end
```
We created a plain ruby object, ```AddsUserToList```, which contains the business logic from before (with ```tap``` to make things a little nicer). In the controller we call this object and not ```User``` directly.
This is an improvement, but testing this service object would require us to somehow stub ```User#find_or_create_by``` to avoid hitting the database, and probably also stub out ```NotifiesUser#run``` in order to avoid sending a real notification out. In any case, hard-coding the name of the class of your collaborator is a really bad idea since it couples your class with your collaborators forever.

Also, referencing the class ```User``` directly means that our unit tests will have to load active record and the entire rails stack, but even worse - the entire app and its dependencies. This load time can be a few seconds for trivial rails apps, but can sometimes be 30 seconds for really big rails apps. Unit tests should be *fast* to run as part of your test suite but also fast to run individually, which means they should not load the rails stack or your application (also see [Corey Haines's talk](http://www.youtube.com/watch?v=bNn6M2vqxHE)
 on the subject).

The most straight forward way to decouple the object from its collabortors is to inject the dependencies of ```AddsUserToList```:

Injecting Dependencies
--------------
```ruby
class AddsUserToList
  def self.run(username, email_list_name, creates_user = User, notifies_user = NotifiesUser)
    creates_user.find_or_create_by(username: username).tap do |user|
      notifies_user.(user, email_list_name)
      user.update_attributes(email_list_name: email_list_name)
    end
  end
end
```
We can now pass any class that creates a user and any class that notifies a user, which means testing will be easier which makes passing a different implementation of the dependencies will also be easy. Since we supplied reasonable defaults we don't need to pass these dependencies at all, and our controller can stay unchanged.

The fact that we are specifying ```User``` as the default value of creates_user in the parameter list does *not* mean that this class and all its dependants (ActiveRecord, our app and other gems) will get loaded. Ruby's *Deferred Evaluation* of the default values means that if these default values are not needed they will not get loaded, so we can run the unit test without loading rails.

Simplifying the Interface
----------------------------------

The method ```AddsUserToList#run``` receives 4 arguments. Users of this method need to know the *order* of the list. Also, it is likely that over time you'd discover you need to add more arguments. When this happens you will need to update all users of the method. A more flexible solution is to use a hash of arguments. This will make the interface more stable and esure the number of arguments does not grow when we find that we need to add more arguments. I often find that for many classes I end up changing from an argument list to a hash of options at some point, so why not [use it in the first place](http://www.poodr.com/)? But does it mean that we need to give up the advantages of deferred evaluation of the default values? Not at all.

We will use ```Hash#fetch``` which receives a block that is not evaluated unless the queried key is not present. The code in the block to ```fetch``` will never get evaluated, and ```User``` won't get loaded. In addition, if we need to evealuate more than one statement when computing the default value we can't do it in the argument list itself, but we can do it using ```Hash#fetch```.

Before I present the final code snippet I'd like to make another comment: when my classes contain only one public method I don't like calling it 'run', 'do' or 'perform' since these names don't convey a lot of information. In this case I'd rather call it 'call' and use ruby's shorthand notation for invoking this method. A nice bonus is being able to pass in a proc instead of the class itself if I need it.

```ruby
class AddsUserToList
  def self.run(args)
    creates_user = args.fetch(:creates_user) { User }
    notifies_user = args.fetch(:notifies_user) { NotifiesUser }

    creates_user.find_or_create_by(username: args.fetch(:username)).tap do |user|
      notifies_user.(user, args.fetch(:email_list_name))
      user.update_attributes(email_list_name: args.fetch(:email_list_name))
    end
  end
end
```

The end result looks like this:

The Complete Refactoring
-------------------

Before:
```ruby
class EmailListsController < ApplicationController
  respond_to :json
  def create
    user = User.find_or_create_by(username: params[:username])
    NotifiesUser.run(user, 'blog_list')
    user.update_attributes(email_list_name: 'blog_list')
    respond_with user
  end
end
```
After:
```ruby
class EmailListsController < ApplicationController
  respond_to :json
  def create
    user = AddsUserToList.(username: params[:username], email_list_name: 'blog_list')
    respond_with user
  end
end
```
```ruby
class AddsUserToList
  def self.call(args)
    creates_user = args.fetch(:creates_user) { User }
    notifies_user = args.fetch(:notifies_user) { NotifiesUser }

    creates_user.find_or_create_by(username: args.fetch(:username)).tap do |user|
      notifies_user.(user, args.fetch(:email_list_name))
      user.update_attributes(email_list_name: args.fetch(:email_list_name))
    end
  end
end
```
The Tests
--------
The class ```AddsUserToList``` can be tested using *true*, isolated unit tests: we can easily isolate the class under test and make sure it properly communicates with its collaborators. There is no database access, no heavy handed request stubbing and if we want to - no loading of the rails stack. In fact, I'd argue that any test that requires any of the above is not a unit test, but rather an integration test (see entire repo [here](https://github.com/orend/register)).

```ruby
describe AddsUserToList do
  let(:creates_user) { double('creates_user') }
  let(:notifies_user) { double('notifies_user') }
  let(:user) { double('user') }
  subject(:adds_user_to_list) { AddsUserToList }

  it 'registers a new user' do
    expect(creates_user).to receive(:find_or_create_by).with(username: 'username').and_return(user)
    expect(notifies_user).to receive(:call).with(user, 'list_name')
    expect(user).to receive(:update_attributes).with(email_list_name: 'list_name')

    adds_user_to_list.(username: 'username', email_list_name: 'list_name', creates_user: creates_user, notifies_user: notifies_user)
  end
end
```

Here we pass in mocks (doubles) for each collaborator and expect them to receive the correct messages. We do not assert any values - specifically not the value of ```user.email_list_name```. Instead we require that ```user``` receives the ```update_attributes``` method. We need to *trust* ```user``` to update the attributes. After all, that's a unit test for ```AddsUserToList```, not for ```user```.

As you can see there is a close resemblance between the test code and the code it is testing. I don't see it as a problem. A unit test should verify that the object under test sends the correct messages to its collaborators, and in the case of ```AddsUserToList``` we have a controller-like object, and a controller's job is to... coordinate sending messages between collaborators. Sandi Metz talks about what you should and what you shuold not test [here](http://www.confreaks.com/videos/2452-railsconf2013-the-magic-tricks-of-testing). To use her vocabulary, all we are testing here are outgoing command messages since these are the only messages this object sends. For that reason I think this resemblance is acceptable.

I omit the integration (controller) tests here, but [please don't forget them](http://solnic.eu/2012/02/02/yes-you-should-write-controller-tests.html) in your code. They will be much simpler and there will be fewer of them if you extract service objects.

Some Numbers
----------

How much faster is this test from a unit test that touches the database and loads rails and the application? Here are the results:

|               |  Single Test Runtime | Total Suite Runtime  |
| ------------- |:-------------:| :-----:|
| **Before**        |   0.0530s     |   2.5s |
| **After**         |   0.0005s     |   0.4s |

A single test run is roughly **a hundred times faster**. The absolute times are rther small but the difference will be very noticeable when you have hundreds of unit tests or more. The total runtime in the "before" version takes roughly two seconds longer. This is the time it takes to load a trivial rails app on my machine. This will be significantly higher when the app grows in size and adds dependent gems.

Conclusion
----------

The 'Before' version's tests are harder to write and are significantly slower. It also bundles many responsibiilties into a single class, the Controller class. The 'After' version is easier to test (we pass mocks to override the default classes). This means that in our code in ```AddsUserToList``` we can easily replace the collaborators with others if we need to, in case the requirements change. The controller has been reduced to performing the most basic task of collecting input and invoking the correct mehtods to excercise here.

Is the 'After' version better? I think it is. It's easier and faster to test, but even more importantly the collaborators are clearly defined and are treated as *roles*, not as specific implementations. As such, they can always be replaced by different implementations of the role they play. We now can concerate on the *messages* passing between the different *roles* in our system.

When you practice TDD with mock objects you will almost be forced to inject your dependencies in order to mock collaborators. Extracting your business logic into service objects makes all this much easier, and further decoupling from active record makes the tests true unit tests that are also blazing fast.

This brings us closer to a lofty design goal stated by Kent Beck:

>"When you can extend a system solely by adding new objects without modifying any existing objects, then you have a system that is flexible and cheap to maintain."

