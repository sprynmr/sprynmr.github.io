---
layout: post
title: ReactiveCocoa and MVVM, an Introduction
tags: ReactiveCocoa, View-Model
---

## MVC

Anyone who has been developing software for a decent length of time is familiar with **MVC**. It stands for **Model View Controller**, and is a proven pattern for organizing your code in complex application design. It's also proven to have a second meaning in iOS development: **Massive View Controller**. It leaves a lot of developers scratching their heads as to how to keep their code nicely decoupled and organized. As a whole, iOS developers have come to the conclusion they need to [slim down their view controllers](http://www.objc.io/issue-1/), and further separate concerns; but how?

## MVVM

That's where **MVVM** comes in. It stands for **Model View View-Model**, and it's here to help us create more manageable, better designed code.

In some situations it doesn't make much sense to deviate from the Apple recommended way of programming an app. I don't mean that it's frowned upon, I just mean the gains might be minimal and the cost may be too high. For example I wouldn't recommend implementing your own view controller baseclass and trying to handle the view lifecycle yourself.

In that spirit, I want to address the question: **Is it unwise to use an application design pattern other than the one (MVC) recommended by Apple?**

**No.** For two reasons.

1. Apple hasn't provided any real guidance on how to solve for the Massive View Controller problem. They leave it up to us to figure out how to add some more sanity to our code.[^apple-view-models] MVVM is a pretty awesome way of accomplishing that.
2. MVVM, at least the style of MVVM that I'm going to show here, fits really nicely within the MVC pattern. It's as if we are taking MVC to the next logical step.

I'm not going to cover the history of MVC/MVVM as it's been covered elsewhere, and I'm not that well versed on it. I'm going to focus on how we can use it in iOS/Mac development.

### Defining MVVM

1. **Model** – The model doesn't really change in MVVM. Depending on what you preference is, your models may or may not encapsulate some additional business logic responsibilities. I tend to use it more as a construct with which to hold information representing a data-model object, and keep any consolidated logic for creating/managing models in a separate manager type class.
2. **View** - The view encompasses the actual UI itself (whether that means `UIView` code, storyboards, or xibs), any view specific logic, and reaction to user input. This includes a lot of the responsibilities handled by the `UIViewController` in iOS, not just `UIView` code and files.
3. **View-Model** – The term itself can lead to confusion, as it's a mashup of two terms we already know, but it's something entirely different. It's not a model in the traditional data-model structure sense (which again, is just my preference). One of its responsibilities *is* that of a static model representing the data necessary for the view to display itself; but it's also responsible for gathering, interpreting, and transforming that data. This leaves the view (controller) with a more clearly defined task of presenting the data supplied by the view-model.

### More about the view-model
The term **view-model** really is quite poor for our purposes. A better term might be **"View Coordinator"**.[^DaveLee] You can think of it almost like the team of researchers and writers behind a news anchor on tv. It gathers the raw data from the necessary sources (database, web service calls, etc), applies logic, and massages that data into presentation data for the view (controller). It exposes (usually via properties) only the information the view controller needs to know about to do it's job of displaying the view (ideally you are not exposing your data-model objects). It's also responsible for making changes to the data upstream (e.g. updating models/database, API POST calls).


## MVVM in a MVC world
Like the term view-model, I think the acronym MVVM is somewhat poor for exactly how we'll be using it for iOS development. Let's take a look at that acronym again and see how it fits into MVC.

For the sake of this diagram, let's reverse the **V** and **C** in **MVC**, so the acronym more accurately reflects the direction of the relationships between the components, leaving us with **MCV**. I'm going to do the same with **MVVM**, moving the **V(iew)** to the right side of **VM** ending up with **MVMV**. (I'm sure there's a reason these acronyms aren't in this more sensible order in the first place.)

**Here is a simple mapping of how these two patterns fit together in iOS:**

<img src="/assets/images/MCVMVMV.svg" width="700" />

- I tried to keep the size of the blocks to (very) roughly the amount of work they are responsible for.
- [Notice how huge the view controller is?](http://i0.kym-cdn.com/photos/images/newsfeed/000/228/269/demotivational-posters-theres-your-problem.jpg)
- You can see the big chunk of responsibilities that overlap between our giant view controller and the view-model.
- You can also see how part of the view controller footprint overlaps with the view in MVVM.

You might be relieved to know we aren't actually removing the concept of the view controller or dropping the "controller" term to match MVVM. (Phew.) We are just going to carve out that chunk of overlapping responsibilities into the view-model, and make the view controller's life much easier.

What we really end up with is **MVMCV**. **M**odel **V**iew-**M**odel **C**ontroller **V**iew. I'm sure I'm giving someone fits with my free spirited application design pattern hacking.

![Creating a new pattern](/assets/images/MCVMVMV.gif)

**Our result:**

<a href="/assets/images/MVMCV.svg"><img src="/assets/images/MVMCV.svg" width="700" /></a>

Now the view controller's only concerns are configuring and managing the various views with data from the view-model, and letting the view-model know when relevant user input occurs that needs to change data upstream. The view controller doesn't need to know about web service calls, Core Data, model objects[^exposing-models], etc.

The view-model will live as a property on the view controller. The view controller knows about the view-model and it's public properties, but the view-model knows nothing about the view controller. You should already feel better about this design as we have better separation of concerns going on here.

Another way to help you understand how our components fit together, and where the responsibilities fall, is to look at a layer diagram of our new application building blocks.

[![MVVM Layer Architecture](/assets/images/mvvm-layers.svg)](/assets/images/mvvm-layers.svg)
(Courtesy [Dave Lee @kastiglione](https://twitter.com/kastiglione))

## View-Model and View Controller, together but separate

Let's look at a simple view-model header to get a better idea of what our new building block looks like. For a simple scenario, let's build a fake twitter client that lets someone lookup the most recent replies at any twitter user by entering their username and hitting "Go". Our example interface will:

- Have a `UITextField` where the user can enter a twitter username, and a "Go" `UIButton`
- Have a `UIImageView` and a `UILabel` that display the avatar and name of the current user being viewed
- Have a `UITableView` below that where we view the most recent replies (tweets)
- Allow for infinite scrolling


<img src="/assets/images/tweeboatplus.svg" width="320" />

### The Example View-Model

The header of our view-model might look like this:

{% gist sprynmr/433c36a0796f17ec1946 %}

Pretty straightforward stuff. Notice all those **glorious `readonly` properties**? The view-model exposes the minimum information necessary to our view controller, and the view controller really doesn't care how the view-model got that information. *Neither do we for now*. Just assume the typical web service calls, validation, data manipulation and storage that you are used to.

#### Things the view-model isn't doing
- Acting directly on the view controller in any form or notifying it directly of changes

### The View Controller

#### The view controller would use data FROM the view-model to:

- Toggle the `enabled` property on the "Go" button when the `usernameValid` value changes
- Adjust the `alpha` on the button to be .5 when `usernameValid` equates to NO (1.0 when `YES`)
- Update the `text` on the UILabel with the string from `userFullName`
- Update the `image` on the UIImageView with the value from `userAvatarImage`
- Configure the table view cells with the objects in `tweets` (More on that later)
- Supply a "loading" cell if `allTweetsLoaded` is NO when reaching the bottom of the table view

#### The view controller would act on the view-model as follows:

- Update the only `readwrite` property on the view-model, `username`, whenever the text in the UITextField changes
- Call `getTweetsForCurrentUsername` on the view-model when the "Go" button is tapped
- Call `loadMoreTweets` on the view-model when the loading cell in the table is reached

#### Things the view controller isn't doing:

- Making web service calls
- Managing the `tweets` array
- Determining what a valid username is
- Formatting the user's first and last name into a full name
- Downloading the user avatar and turning it into a UIImage [^downloading-image]
- Breaking a sweat

*Notice again how the total onus is on the view controller to act on the changes in the view-model.*

## Child View-Models

I mentioned configuring table view cells with objects from the `tweets` array on the view-model. Typically you would expect those to be data-model objects representing tweets. You may have been wondering about that, since we try not to expose data-model objects via the MVVM pattern[^exposing-models].

**A view-model doesn't have to represent everything on the screen.** You can use child view-models to represent smaller, potentially more encapsulated portions of the screen. This is especially beneficial if that bit of view (e.g. a table cell) could be re-used in your app and/or represent multiple data-model objects.

You don't always need child view-models. For example, I might use a table header view to render the top section of our "tweetboat plus" app. It's not a reusable component, so I might just pass in the same view-model we are already using for the view controller to that custom header view. It would use the information it needed from that view-model and ignore the rest. This is an especially awesome way to keep your subviews in sync, as they can all be effectively working with the same exact context of information, and observing the exact same properties for updates.

In our example app, the `tweets` array will be full of child view-models that might look like this:

{% gist sprynmr/adfd5eb3775a225a2011 %}

You might think that looks too much like a regular "Tweet" data-model object. Why the work of converting it to a view-model? Even when similar, the view-model lets us limit the information exposed to only what we need, provide additional properties that might be transformed data, or calculate data specific for this view. (Again, it's also nice when possible to not expose the mutable data-model objects, as we want the view-model itself to be responsible for making any changes to those, not the view or view controller.)

### Mom, where do view-models come from?

So when and where are view-models created? Do view controllers create their own view-models?

#### View-models begat view-models.

Strictly speaking, you should create a view-model for your top view-controller in your app delegate. When presenting a new view controller, or bit of view that's represented by a view-model, you ask the current view-model to create the child view-model for you.

![Child View-Model Diagram](/assets/images/child-view-models.svg)

Say we wanted to add a profile view controller that would show whenever you tapped the avatar in the top portion of our app. We might add a method to our primary view-model that looked something like this:

{% highlight objective-c %}
- (MYTwitterUserProfileViewModel *) viewModelForCurrentUser;
{% endhighlight %}

and use it like this in our primary view controller:

{% gist sprynmr/194ab0c97500592c3954 %}

In this example I want to present a profile view controller of our current user, but my profile view controller needs a view-model. My main view controller here doesn't know all of the necessary data about this user to build the associated view-model (nor should it), so it asks it's own view-model to do the dirty work for it of creating the new view-model.

#### Lists of view-models

In the case of our tweet cells, I would typically create all the child view-models for the corresponding cells ahead of time, when the data driving the screen (probably via a web service call in this case) was gathered. So in our scenario, `tweets` on the main view-model would be an array of `MYTweetCellViewModel` objects. In `cellForRowAtIndexPath` on my table view, I would simply grab the view-model at the correct index, and assign it to the view-model property on my cell.

## Functional Core, Imperative Shell

This view-model approach to application design is a stepping stone on the path to a type of application design coined ["Functional Core, Imperative Shell"](https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell)[^andy] by [Gary Bernhardt](http://twitter.com/garybernhardt).

### Functional Core

The view-model is the ["functional core"](http://www.smashingmagazine.com/2014/07/02/dont-be-scared-of-functional-programming/), though pragmatically in iOS/Objective-C it's tricky to get to the purely functional level (Swift provides some additional functionality that will get us closer). The general idea is to make your view-models have as little dependence and impact on the rest of the "application world" as possible. What does that mean? Think of simple functions you probably learned when first studying programming. They accepted maybe one or two parameters, and output a result value. **Data in, data out.** Maybe the function did some math, or combined a first and last name. No matter what else was going on in the application, the same input would create the same output. That's the functional aspect.

That's what we are striving for with view-models. They contain the logic and functionality for transforming data and storing it's output in properties. Ideally the same input (e.g. web service response) will derive the same output (property values). This means eliminating as many factors as possible by which the rest of the "application world" might affect the output, [such as using a lot of state](http://www.sprynthesis.com/2014/06/15/why-reactivecocoa/). **A great first step would be not including UIKit.h in your view-model header.**[^uikit-header] UIKit is, by it's nature, going to affect a lot of the application world. It contains many "side-effects", whereby changing one value or calling one method will trigger many indirect (even unknowable) changes.

### Imperative (Declarative?) Shell

The imperative shell is where we do all the state-changing, application-world-altering dirty work needed to turn our view-model data into something for the user on the screen. This is our view (controller), where we have pragmatically isolated the UIKit work. I would still make it a point to eliminate as much state as possible and do this work in a declarative nature using something like ReactiveCocoa, but iOS and UIKit are imperative by design.[^table-data-source]

### Testable Core

Units tests on iOS are a nasty, hacky, hot mess. At least that was the conclusion I came to when I began working with them. I even read a book or two on the subject, but my eyes glazed over when they started mocking and swizzling view controllers to try and make some of their logic testable. I eventually relegated my unit tests to my models and any companion model management classes.

The biggest advantage of this functional core(ish) view-model, aside from the number of bugs eliminated every time you reduce state, *is that it becomes extremely unit testable*. If you have methods that should generate the same output everytime they are supplied the same input, that fits extremely well into the world of unit tests. We now have our data gathering/logic/transforming extracted away from the complexities of a view controller. That means no crazy mock objects, method swizzling, or other insane workarounds (hopefully) are required to build really good tests.

## Connecting Everything

**So how do we update our views when public properties on the view-model change?**

Most of the time we'll initialize view controllers with their corresponding view-model. Something along the lines of what we just saw above:

{% highlight objective-c %}
MYTwitterUserProfileViewController *profileViewController =
    [[MYTwitterUserProfileViewController alloc] initWithViewModel: userProfileViewModel];
{% endhighlight %}

Sometimes you don't have the option of passing the view-model in during initialization, e.g. in the case of storyboard segues or cell dequeuing. For these you can expose a public writeable view-model property on the view (controller) in question.

{% highlight objective-c %}
MYTwitterUserCell *cell =
    [self.tableView dequeueReusableCellWithIdentifier:@"MYTwitterUserCell" forIndexPath:indexPath];
// grab the cell view-model from the vc view-model and assign it
cell.viewModel = self.viewModel.tweets[indexPath.row];
{% endhighlight %}

In cases where we can pass in the view-model before a hook like `init` or `viewDidLoad`, we could initialize the state of all our UI elements with the properties off of the view-model.

{% gist sprynmr/cead1f81935f18b2acb5 %}

Great! We've configured our initial values. What about when data on the view-model changes? How will the go button ever become enabled? How will our user label and avatar ever get populated with the results of the network calls?

We could expose the view controller to the view-model so it can call an "updateUI" method on it when  relevant data changes or something similar. (Don't do this.) Make the view controller a delegate on the view-model? Fire a notification on the view-model when anything changes? *Nooope.*

Our view controller knows about some changes being made. We could use delegate methods off of the `UITextfield` to update the state of the button by checking the view-model every time there is a character change.

{% gist sprynmr/8a019580d7a3fc829746 %}

This sort of solves for the scenario where the only thing affecting `isUsernameValid` on the view-model is the textfield changing. What if there are other variables/actions that alter the `isUsernameValid` state? What about network calls in the view-model? Maybe we could add completion handlers to our method calls on the view-model so we can update everything on the UI at that point? What about using the venerable, cumbersome KVO methods?

We could probably, eventually, connect all the contact points on the view-model and view controller using various mechanisms we are familiar with, but you already know [that's not where this is headed](http://www.sprynthesis.com/2014/06/15/why-reactivecocoa/). It creates a large amount of entry points into our code where we have to fully recreate the context of our application state just to do a simple UI update.

##Enter ReactiveCocoa

ReactiveCocoa (RAC) is here to save our bacon, and just maybe give us a little sanity back. Let's look at how.

Consider controlling the flow of information through a new user screen that updates the status of the submit button when the form is valid. Here's how you might currently do that type of task:

<a href="/assets/images/new-user-form-imperative.svg"><img src="/assets/images/new-user-form-imperative.svg" width="800" /></a>

You end up carefully threading your simple logic through many different bits of otherwise unrelated contexts in your code by using state. See all the different entry points into your flow? (And this is just *one* thread of logic for *one* UI element.) [The abstraction we are using to program isn't smart enough](http://www.sprynthesis.com/2014/06/15/why-reactivecocoa/) to track the relationships of all these things for us, so we end up doing it (poorly) ourselves.

**Let's look at the declarative version:**

<a href="/assets/images/new-user-form-declarative.svg"><img src="/assets/images/new-user-form-declarative.svg" width="900" /></a>

This may look like an old school CS diagram for documenting the flow of our application. With a declarative style of programming, we use a higher level abstraction that lets us actually program much closer to the way we design the flow in our minds. We make the computer do more of the work for us. The actual code now resembles this diagram closely.

### RACSignal

`RACSignal` (signal) is the building block for all of RAC. It is an object that represents information that we will eventually receive. When you have a concrete representation of the information you will receive at some point in time, **you can go ahead and apply logic and build your information flow up front (declarative)**, instead of having to wait for that event to occur (imperative).

**A signal takes all those async methods for controlling the flow of information through your app (delegates, callback blocks, notifications, KVO, target/action event observers, etc) and unifies them under one interface.** *This just flat out makes sense.* Not only that, it gives you the ability to transform/split/combine/filter that information easily as it flows through your application.

![Replace standard async tools](/assets/images/replace-async-tools.svg)

#### So what is a signal? This is a signal:

![A signal doing nothing](/assets/images/signal-no-subscribers.svg)

A signal is an object that sends out a stream of values. But our signal here isn't doing anything. That's because it doesn't have any subscribers. A signal will only send out information if it has a subscriber listening (er, subscribed) to it. It will send that subscriber zero or more "next" events containing the value, followed by either a "complete" event or an "error" event. (A signal is similar to a "promise" in some other languages/toolkits, but far more powerful as it isn't limited to only sending a return value once to it's subscribers.)

![A signal with a subscriber](/assets/images/signal-with-subscriber.svg)

Like I mentioned before, you can filter, transforms, split and combine those values as you see necessary. Different subscribers may need to use the values sent via the signal in different ways.

![A signal with two subscribers](/assets/images/signal-map.svg)

#### Where do signals get the values they are sending along?

Signals are bits of asynchronous code that wait for something to happen, and then send the resulting value to their subscribers. You can create them manually with the `createSignal:` class method on `RACSignal`:

{% gist sprynmr/fedd52e32a6ead20369c %}

Here I'm creating a signal using a (fake) network operation that has success and fail blocks.[^defer] I use the provided `subscriber` object on which I `sendNext:` and `sendCompleted:` for the success block, or `sendError:` if the failure block fires. I can now subscribe to this signal and I will receive the json value or an error when the response comes back.

Luckily, the creators of RAC actually use their own library to build real things (fathom that), so they have a strong idea what's needed in our daily work. They have provided us with a wealth of mechanisms to pull signals off of the existing asynchronous patterns we commonly use. Just don't forget that if you have an asynchronous task that isn't covered with some built in signal, you can *easily* create it with `createSignal:`, and similar methods.

One such provided mechanism is the `RACObserve()` macro. (If you don't like macros, you can easily look under the hood and use the slightly more verbose representation. It's still great. There are solutions for [using the RAC library with swift](http://www.scottlogic.com/blog/2014/07/24/mvvm-reactivecocoa-swift.html) too, until we get it's [swifty replacement](https://github.com/ReactiveCocoa/ReactiveCocoa/pull/1382).) This macro is the RAC replacement for the woeful KVO APIs. You just pass in the object and keypath of the property you want to observe on that object. Given those parameters, `RACObserve` generates a signal that immediately sends the current value of that property (once it gets a subscriber), and any further changes to that property.

{% highlight objective-c %}
    RACSignal *usernameValidSignal = RACObserve(self.viewModel, usernameIsValid);
{% endhighlight %}

![A signal created with RACObserve](/assets/images/signal-racobserve.svg)

This is just one tool provided to create signals. There are several out of the box ways to pull signals off built in control flow mechanisms:

{% gist sprynmr/94472f0285139056da26 %}

Remember you can easily create your own signals as well, including [replacing other delegates](http://spin.atomicobject.com/2014/02/03/objective-c-delegate-pattern/) that may not have built in support. Just think how cool it is that we can now pull signals off all these disconnected async/control flow tools and combine them together! *These can become nodes in that declarative diagram we saw above. That's really exciting.*

#### What is a subscriber?

Simply put, a subscriber is the bit of code that is waiting for the signal to send along it's values so it can do something with them. (It can also act on the "complete" and "error" events too).

Here is a simple subscriber, created by passing a block to the `subscribeNext` instance method on a signal. Here we are observing the current value of a property on an object via the signal created with the `RACObserve()` macro, and assigning it to an internal property.

{% gist sprynmr/fd3b14a49582c4aa9af1 %}

Notice that RAC only deals in objects, not primitives like `BOOL`. Not to worry though, as RAC mostly handles the conversions for you.

Luckily this sort of binding behavior was also recognized by the RAC creators as a common necessity, so they provided another macro `RAC()`. Similar to `RACObserve()`, you provide the object and the parameter you want the incoming value bound TO, and it does the work under the hood of creating a subscriber and updating the value of the property. Our sample now looks like this:

{% highlight objective-c %}
- (void) viewDidLoad {
    //...
    RAC(self, usernameIsValid) = RACObserve(self.viewModel, isUsernameValid);
}
{% endhighlight %}

{% highlight objective-c %}
{% endhighlight %}

But that's a little silly given our goal here. We don't really want to store the value from the signal in a property (and thus create state), what we really want to do is update our UI with the information gleaned from that value.

#### Transforming the streams

Now we're getting into the methods that RAC provides us with for transforming our stream of values. We're going to make use of the `map` instance method on `RACSignal`.

{% gist sprynmr/a731e8026c2143eaf2e3 %}

So now we've bound the changes that occur to `isUsernameValid` on our view-model DIRECTLY to the enabled property on our goButton. How cool is that? The binding to alpha is even cooler, because we are transforming our value into something relevant for the `alpha` property on the button using the `map` method. (Notice we return an `NSNumber` here instead of a primitive float. This is basically the one spot where you need to take care of converting your primitive to an object for RAC because it can't derive that for you.)

### Multiple subscribers, side effects, and expensive operations

One important thing to realize when subscribing to signal chains, is that every time a new value is sent through that chain, it is actually sent once per subscriber. It makes sense when you realize that as far as we are concerned, these values sent by a signal aren't stored anywhere (aside from the internal RAC implementation). When the signal needs to send out a new value, it loops through all its subscribers and sends them each that value[^simplification].

Why is this important? It means that any side effects you might have in your signal chain somewhere, any transformations that affect the application world, will occur multiple times. This is often unexpected by users new to RAC. (It also goes against the idea of building functionally – data in, data out).

A contrived example might be a signal for a button tap event that updates a counter property on `self` somewhere in the signal chain. If there are multiple subscribers to this signal chain, that property is going to be incremented more than you intend. You should strive to eliminate side effects from your signal chains as much as possible. When side effects are unavoidable, there are mechanisms in place you can use to prevent this. I'll explore that in another article.

In addition to side effects, you need to pay attention to signal chains with expensive operations and variable data. Network requests are an example of all three:

1. Network requests affect the networking layer of your app (side effect).
2. Network requests introduce variable data into your signal chain. (Two identical requests could return different data.)
3. Network requests are slow.

As an example, you may have a signal that sends a value each time a button is tapped, and you want to transform that value into the results of a network request. If you have multiple subscribers doing something with the value returned by that signal chain, you will be making multiple network requests.

![A signal with side effects happening twice](/assets/images/signal-side-effect.svg)

Network requests obviously aren't an uncommon need. As you would expect, RAC provides solutions for these situations, namely `RACCommand` and multicasting. I'll get into both more in my next post.


## Tweetboat Plus

Now that the short introduction (ahem) is out of the way, lets look at how we might connect our view-model and view controller using ReactiveCocoa.

{% gist sprynmr/3af94e7bb7065c47393b %}

Lets talk a walk through this example.

{% highlight objective-c %}
RAC(self.viewModel, username) = [myTextfield rac_textSignal];
{% endhighlight %}

Here we are pulling a signal off of the `UITextField` using a method provided by the RAC library. This line is binding the read-write `username` property on the view-model to any updates from our textfield as the user types.

{% highlight objective-c %}
RACSignal *usernameIsValidSignal = RACObserve(self.viewModel, usernameValid);

RAC(self.goButton, alpha) = [usernameIsValidSignal
    map: ^(NSNumber *valid) {
        return valid.boolValue ? @1 : @0.5;
    }];

RAC(self.goButton, enabled) = usernameIsValidSignal;
{% endhighlight %}
Here we create a signal `usernameIsValidSignal` using `RACObserve` on the view-model `usernameValid` property. Any time this property changes, it will send a new `@YES` or `@NO` down the pipe. We take that value and bind it to two properties on the `goButton`. First we update the alpha to either 1 or .5  for YES and NO respectively (remember we have to pass an `NSNumber` back here). Then we bind it directly to the `enabled` property, because the YES or NO will work perfectly there without any transformation.

{% highlight objective-c %}
RAC(self.avatarImageView, image) = RACObserve(self.viewModel, userAvatarImage);

RAC(self.userNameLabel, text) = RACObserve(self.viewModel, userFullName);
{% endhighlight %}

Next we are creating bindings for the image view and user label in our header, by creating signals using the `RACObserve` macro again on the corresponding properties on our view-model.

{% highlight objective-c %}
@weakify(self);
[[[RACSignal merge:@[RACObserve(self.viewModel, tweets),
                     RACObserve(self.viewModel, allTweetsLoaded)]]
    bufferWithTime:0 onScheduler:[RACScheduler mainThreadScheduler]]
    subscribeNext:^(id value) {
        @strongify(self);
        [self.tableView reloadData];
    }];
{% endhighlight %}
This one looks a little tricky, so let's spend some extra time on it. We want to update our table view whenever the `tweets` array or the `allTweetsLoaded` properties change on the view-model. (In this example, we're going with the simple method of reloading the entire table.) So we merge the signals created by observing those two properties into a greater signal that will now send a value when either of these properties change. (Typically you want a signal's values to be homogeneous, not mixed like this signal's values would be. This will likely be enforced with RAC swift, but here we don't care about the actual value sent, we are just using it to trigger the reload on the table view.)

So the scary looking part here is probably the `bufferWithTime:onScheduler:` method chained in there. This is needed to work around one problem in UIKit. We need to track both properties, `tweets` and `allTweetsLoaded`, in case one changes and not the other (we need to reload the table either way). The catch is that sometimes both of these properties will change at the same exact time, which means both signals in our greater merged signal will send a value, and `reloadData` will get called twice in the same run loop. UIKit doesn't like that. `bufferWithTime:` catches any next values sent over the time specified, and then sends them along in a group to the subscriber when that time has elapsed. By passing 0 as the time, `bufferWithTime:` will catch all values sent by our merged signal in a particular run-loop and then send them along.[^NSTimer] Don't worry about the scheduler for now, just think of it as specifying that these values must get delivered on the main thread. Now we are ensuring `reloadData` only gets called once per run-loop.

*Note the strong weak dance I'm doing with the `@weakify`/`@strongify` macros. This is VERY important with all these blocks we are creating. `self` **WILL** be strongly captured and you **WILL** get retain cycles if you are not extremely conscious of breaking these cycles when using `self` in  a RAC block.*

{% highlight objective-c %}
[[self.goButton rac_signalForControlEvents:UIControlEventTouchUpInside]
    subscribeNext: ^(id value) {
        @strongify(self);
        [self.viewModel getTweetsForCurrentUsername];
    }];
{% endhighlight %}
This is an area where `RACCommand` will come into play in my next post, but for now we just manually call `getTweetsForCurrentUsername` on the view-model whenever the go button is touched.

We already covered the first part of `cellForRowAtIndexPath`, so I'll just cover the loading cell here:

{% highlight objective-c %}
MYLoadingCell *cell =
    [self.tableView dequeueReusableCellWithIdentifier:@"MYLoadingCell" forIndexPath:indexPath];
[self.tableView loadMoreTweets];
return cell;
{% endhighlight %}
This is another area where we will leverage `RACCommand` in the future, but for now we are calling the `loadMoreTweets` method on the view-model. We will just trust that the view-model prevents multiple calls internally if the cell is hidden and shown multiple times.

{% highlight objective-c %}
- (void) awakeFromNib {
    [super awakeFromNib];

    RAC(self.avatarImageView, image) = RACObserve(self, viewModel.userAvatarImage);
    RAC(self.userNameLabel, text) = RACObserve(self, viewModel.tweetAuthorFullName);
    RAC(self.tweetTextLabel, text) = RACObserve(self, viewModel.tweetContent);
}
{% endhighlight %}
This should be fairly straightforward now, aside from one thing I want to point out. We are binding an image and strings to the appropriate properties on our UI, but note that `viewModel` is on the right side of the comma in the `RACObserve` macro. These cells will end up getting reused and new view-models will be assigned. Instead of listening for the `viewModel` property to change and then re-setting up our bindings everytime, if we put `viewModel` on the right side of the comma, `RACObserve` is going to take care of that for us. So we only set up this binding ONCE and let Reactive Cocoa do the rest. This is a good thing to keep in mind for performance with bindings on table cells. In practice I've had no issues even with lots of table cells screaming around.

#### BONUS - Eliminating even more state

In some cases you can eliminate even more state on your view-model by exposing `RACSignal`s instead of properties like strings and images. Then your view controller wouldn't have to create it's own with `RACObserve`, and could just leverage those signals straightaway. Be aware that if your signal sends a value before you have subscribed/bound it in your UI, that you won't receive that "initial" value.

##Conclusion

This post may have been a little overwhelming. Don't let that scare you off. There's a lot to cover here, but it's really GOOD stuff, and a great way to stretch your brain. This is a decidedly *different* style of programming, and it takes awhile to stop automatically trying to solve problems with imperative solutions. Even if you don't use this style of programming regularly at first, I think it's helpful to understand, and reminds us that there are very different ways to solve our programmer puzzles.

Next up I will examine a bit of the internals of the view-model that weren't covered here, and introduce `RACCommand` (in hopefully a MUCH shorter post). Then we'll dive into a real world example of a fairly complicated screen from an app I built called [Three Cents](http://www.threecentsapp.com/) where we mix in network calls, core data, multiple UI states and more!

![Three Cents Explore Screen](assets/images/ThreeCentsExplore.gif)

###Further reading:

- [Introduction to MVVM](http://www.objc.io/issue-13/mvvm.html) by Ash Furrow
- [Functional Reactive Programming on iOS](https://leanpub.com/iosfrp/) by Ash Furrow
- [A sample app by Ash Furrow](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=4&cad=rja&uact=8&ved=0CDEQFjAD&url=https%3A%2F%2Fgithub.com%2FAshFurrow%2FC-41&ei=LpmCVKWELcPHsQTFt4CACw&usg=AFQjCNG5auARK8X5_zewrRvDxW_drVxceg&sig2=haR60-wDD_y3X8aK0epaCg&bvm=bv.80642063,d.cWc)
- [MVC, MVVM, FRP, And Building Bridges](http://cocoamanifest.net/articles/2013/10/mvc-mvvm-frp-and-building-bridges.html) by Jonathan Penn
- [MVVM Tutorial with ReactiveCocoa](http://www.raywenderlich.com/74106/mvvm-tutorial-with-reactivecocoa-part-1) by Colin Eberhardt on the Ray Wenderlich site.
- [Basic MVVM with ReactiveCocoa](http://cocoasamurai.blogspot.com/2013/03/basic-mvvm-with-reactivecocoa.html) by Colin Wheeler
- [On MVVM, and Architecture Questions](http://twocentstudios.com/blog/2014/06/08/on-mvvm-and-architecture-questions/) by Chris Trott

---

[^apple-view-models]: In some of the WWDC videos this year, view-models were actually spotted in the sample code apple engineers had on screen. Not sure if any of that made it to the sample code itself.
[^exposing-models]: In practice it's sometimes pragmatic to expose models via the view-model header instead of duplicating a large amount of properties. More on that later.
[^downloading-image]: You could expose the URL instead of the image if you are used to using a category on UIImageView for loading images from the network. That does give a more clear break between the view-model and UIKit, but I view the UIImage itself more as data, and less as the exact presentation of that data. These aren't hard and fast lines.
[^DaveLee]: Thanks to [Dave Lee](https://twitter.com/kastiglione) for exposing this as a good point to cover and for the "View Coordinator" term.
[^uikit-header]: This is a great principle, but there are some grey areas. For instance, you may consider a UIImage "data" and not presentation information. (I like this approach.) In this case, you will need UIKit.h so you can work with the UIImage class.
[^andy]: I recently was fortunate enough to listen [Andy Matuschak](http://andymatuschak.org) give a talk along the lines of this concept where he makes a case for a "Thick value layer, thin object layer". The concept is similar, but focuses on how we can remove objects, and their stateful side-effecty nature, and build a more functional, testable value layer with new data structures in swift.
[^simplification]: This is a simplified explanation for how signal chains actually work, but the basic idea is true.
[^table-data-source]: The table data source is a great example of this, as it's delegate pattern forces the use of state on the delegate to be able to provide information to the table view when requested. In fact the delegate pattern in general forces a whole lot of use of state.
[^almost]: Our nested API block could have access to it's parent's context.
[^defer]: I could also use the `defer` class method on `RACSignal` if I didn't want my network request to happen until there was a subscriber.
[^NSTimer]: NSTimer works the same way. That's no coincidence, as `bufferWithTime:` is built using an `NSTimer`.
