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

In that spirit I want to address the question: **Is it unwise to use an application design pattern other than the one (MVC) recommended by Apple?**

**No.** For two reasons.

1. Apple hasn't provided any real guidance on how to solve for the Massive View Controller problem. They leave it up to us to figure out how to add some more sanity to our code.[^apple-view-models] MVVM is a pretty awesome way of accomplishing that.
2. MVVM, at least the style of MVVM that I'm going to show here, fits really nicely within the MVC pattern. It's as if we are taking MVC to the next logical step.

I'm not going to cover the history of MVC/MVVM as it's been covered elsewhere, and I'm not that well versed on it. I'm going to focus on how we can use it in iOS/Mac development.

### Defining MVVM

1. **Model** – The model doesn't really change in MVVM. Depending on what you preference is, your models may or may not encapsulate some additional business logic responsibilities. I tend to use it more as a construct with which to hold information representing a data-model object, and keep any consolidated logic for managing/manipulating models in a separate manager type class.
2. **View** - The view encompasses the actual UI itself (whether that means `UIView` code, storyboards, or xibs), any view specific logic, and reaction to user input. This includes a lot of the responsibilities handled by the `UIViewController` in iOS, not just `UIView` code and files.
3. **View-Model** – The term itself can lead to confusion, as it's a mashup of two terms we already know, but it's something entirely different. It's not a model in the traditional data-model structure sense (which again, is just my preference). One of its responsibilities *is* that of a model representing the data necessary for the view to display itself; but it's also responsible for gathering, interpretting, and transforming that data. This leaves the view (controller) with a more clearly defined task of presenting the data supplied by the view-model.

### More about the view-model
The term **view-model** really is quite poor for our purposes. A better term might be **"View Coordinator"**.[^DaveLee] You can think of it almost like the team of researchers and writers behind a news anchor on tv. It gathers the raw data from the necessary sources (database, web service calls, etc), applies logic, and massages that data into presentation data for the view (controller). It exposes (usually via properties) only the information the view controller needs to know about to do it's job of displaying the view. It's also responsible for making changes to the data upstream (e.g. updating models/database, API POST calls).


## MVVM in a MVC world
Like the term view-model, I think the acronym MVVM is somewhat poor for exactly how we'll be using it for iOS development. Let's take a look at that acronym again and see how it fits into MVC.

For the sake of this diagram, let's reverse the **V** and **C** in **MVC**, so the acronym more accurately reflects the direction of the relationships between the components, leaving us with **MCV**. I'm going to do the same with **MVVM**, moving the **V(iew)** to the right side of **VM** ending up with **MVMV**. (I'm sure there's a reason these acronyms aren't in this more sensible order in the first place.)

**Here is a simple mapping of how these two patterns fit together:**

<img src="/assets/images/MCVMVMV.svg" width="700" />

- I tried to keep the size of the blocks to (very) roughly the amount of work they are responsible for.
- [Notice how huge the view controller is?](http://i0.kym-cdn.com/photos/images/newsfeed/000/228/269/demotivational-posters-theres-your-problem.jpg)
- You can see the big chunk of responsibilities that overlap between our giant view controller and the view-model.
- You can also see how part of the view controller footprint overlaps with the view in MVVM.

You might be relieved to know we aren't actually removing the concept of the view controller or dropping the "controller" term to match MVVM. (Phew.) We are just going to carve out that chunk of overlapping responsibilities into the view-model, and make the view controller's life much easier.

What we really end up with is **MVMCV**. **M**odel **V**iew-**M**odel **C**ontroller **V**iew. I'm sure I'm giving someone fits with my free spirited application design pattern hacking.

![Creating a new pattern](/assets/images/MCVMVMV.gif)

**Our result:**

<img src="/assets/images/MVMCV.svg" width="700" />

Now the view controller's only concerns are configuring and managing the various views with data from the view-model, and letting the view-model know when relevant user input occurs that needs to change data upstream. The view controller doesn't need to know about web service calls, Core Data, model objects[^exposing-models], etc.

The view-model will live as a property on the view controller. The view controller knows about the view-model and it's public properties, but the view-model knows nothing about the view controller. You should already feel better about this design as we have better separation of concerns going on here.

## View-Model and View Controller, together but separate

Let's look at a simple view-model header to get a better idea of what our new component looks like. For our simple scenario, let's build a twitter client that lets a user lookup the most recent replies at any twitter user by entering their username and hitting "Go". Our example interface will:

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
- Supply a "loading" cell if `finishedLoading` is NO when reaching the bottom of the table view

#### The view controller would act on the view-model as follows:

- Update the only `readwrite` property on the view-model, `username`, whenever the text in the UITextField changes
- Call the `getTweetsForCurrentUsername` on the view-model when the "Go" button is tapped
- Call `loadMoreTweets` on the view-model when the last cell in the table is reached

#### Things the view controller isn't doing:

- Making web service calls
- Managing the array of tweets
- Determining what a valid username is
- Formatting the user's first and last name into a full name
- Downloading the user avatar and turning it into a UIImage [^downloading-image]
- Breaking a sweat

*Notice again how the total onus is on the view controller to act on the changes in the view-model.*

## Child View-Models

I mentioned configuring table view cells with objects from the `tweets` array on the view model. Typically you would expect those to be data-model objects representing tweets. You may have been wondering about that, since we try not to expose data-model objects via the MVVM pattern[^exposing-models].

**A view-model doesn't have to represent everything on the screen.** You can use child view-models to represent smaller, potentially more encapsulated portions of the screen. This is especially beneficial if that bit of view (e.g. a table cell) could be re-used in your app and/or represent multiple data-model objects.

You don't always need child view-models. For example, I might use a table header view to render the top section of our "tweetboat plus" app. It's not a reusable component, so I might just pass in the same view-model we are already using for the view controller to that custom view. It would use the information it needed and discard the rest. This is an especially awesome way to keep your subviews in sync, as they can all be effectively working with the same exact context of information.

In our example app, the `tweets` array will be full of child view-models that might look like this:

{% gist sprynmr/adfd5eb3775a225a2011 %}

### Mom, where do view-models come from?

So when and where are view-models created? Do view controllers create their own view-models?

#### View-models begat view-models.

Strictly speaking, you should create a view-model for your top view-controller in your app delegate. Assuming all other view controllers are effectively "children" of your primary view controller; When presenting a new view controller, or bit of view that's represented by a view-model, you ask the current view-model to create the necessary view-model for you.

Say we wanted to add a profile view controller that would show whenever you tapped the avatar in the top portion of our app. We might add a method to our current view-model that looked something like this:

{% highlight objective-c %}
- (MYTwitterUserProfileViewModel *) viewModelForCurrentUser;
{% endhighlight %}

and use it like this:

{% gist sprynmr/194ab0c97500592c3954 %}

So to review, I want to present a profile view controller of our current user, but my profile view controller needs a view-model. Well my main view controller here doesn't know all of the necessary data about this user to build a view-model (nor should it), so it asks it's own view-model to do the dirty work for it.

#### Lists of view-models

In the case of our tweet cells, I would typically create all the view-models for the corresponding cells ahead of time when the data driving the screen (probably via a web service call in this case) was gathered. So in our scenario, `tweets` on the main view-model would be an array of `MYTweetCellViewModel` objects. So in `cellForRowAtIndexPath` on my table view, I would simply grab the view-model at the correct index, and assign it to the view-model property on my cell.

## Functional Core, Imperative Shell

This view-model approach to application design is a stepping stone on the path to a type of application design coined ["Functional Core, Imperative Shell"](https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell)[^andy] by [Gary Bernhardt](http://twitter.com/garybernhardt)  -- Right?--.

### Functional Core

The view-model is the ["functional core"](http://www.smashingmagazine.com/2014/07/02/dont-be-scared-of-functional-programming/), though pragmatically in iOS/Objective-C it's tricky to get to the purely functional level. The general idea is to make your view-models have as little dependence and impact on the rest of the "application world" as possible. What does that mean? Think of simple functions you probably learned when first studying programming. They accepted maybe one or two parameters, and output a result value. **Data in, data out.** Maybe the function did some math, or combined a first and last name. No matter what else was going on in the application, the same input would create the same output. That's the functional aspect.

That's what we are striving for with view-models. They contain the logic and functionality for transforming data and storing it's output in properties. Ideally the same input (e.g. web service response) will derive the same output (property values). This means eliminating as many factors as possible by which the rest of the "application world" might affect the output, [such as using a lot of state](http://www.sprynthesis.com/2014/06/15/why-reactivecocoa/). **A great first step would be not including UIKit.h in your view-model header.**[^uikit-header] UIKit is, by it's nature, going to affect a lot of the application world. It contains many "side-effects", whereby changing one value or calling one method will trigger many indirect (even unknowable) changes.

### Imperative (Declarative?) Shell

The imperative shell is where we do all the state-changing, application-world-altering dirty work needed to turn our view-model data into something for the user on the screen. This is our view (controller), where we have pragmatically isolated the UIKit work. I would still make it a point to eliminate as much state as possible and do this work in a declarative nature using something like ReactiveCocoa, but we are still operating in an imperative world with iOS and UIKit.

### Testable Core

I imagine I'm not the only one that looked at units tests on iOS for awhile, even read books on the subject, and came away thinking it was nasty, hacky process.

The biggest advantage of this functional core(ish) view-model, aside from the number of bugs eliminated everytime you reduce state, *is that it becomes extremely unit testable*. If you have methods that should generate the same output everytime they are supplied the same input, that fits extremely well into the world of unit tests. We now have our data gathering/logic/transforming extracted away from the complexities of a view controller. That means no crazy mock objects, method swizzling, or other insane workarounds (hopefully) are required to build really good tests.

## Connecting Everything

**So how do we update our view controller when public properties on the view model change?**

Most of the time we'll initialize view controllers with their corresponding view-model. Something along the lines of what we just saw above:

{% highlight objective-c %}
MYTwitterUserProfileViewController *profileViewController =
    [[MYTwitterUserProfileViewController alloc] initWithViewModel: userProfileViewModel];
{% endhighlight %}

Sometimes you don't have the option of passing the view-model in during initialization, e.g. in the case of storyboard segues or cell dequeuing. For these you can expose a public writeable view-model property on the view (controller) in question.

{% highlight objective-c %}
MYTwitterUserCell *cell =
    [self.tableView dequeueReusableCellWithIdentifier:@"MYTwitterUserCell" forIndexPath:indexPath];
// grab the cell view model from the vc view model and assign it
cell.viewModel = self.viewModel.tweets[indexPath.row];
{% endhighlight %}

In cases where we can pass in the view-model before a hook like `init` or `viewDidLoad`, we could initialize the state of all our UI elements with the properties off of the view-model.

{% gist sprynmr/cead1f81935f18b2acb5 %}

Great! We've configured our initial values. What about when data on the view-model changes? How will the go button ever become enabled? How will our user label and avatar ever get populated with the results of the network calls?

We could expose the view controller to the view-model so it can call an "updateUI" method on it when  relevant data changes or something similar. **Noooope.** Fire a notification? **Nooope**

We could use delegate methods off of the UITextfield to update the state of the button by checking the view-model every time there is a character change.

{% gist sprynmr/8a019580d7a3fc829746 %}

This sort of solves for that one scenario where the only thing affecting the `isUsernameValid` on the view-model is via the textfield changing. What if there are other variables/actions that alter the `isUsernameValid` state? What about network calls? Maybe we could add completion handlers to our method calls on the view-model so we can update everything on the UI at that point? What about using the venerable, cumbersome KVO methods?

We could probably, eventually, connect all the contact points on the view-model and view controller using various mechanisms we are familiar with, but you already know [that's not where this is headed](http://www.sprynthesis.com/2014/06/15/why-reactivecocoa/). It creates a large amount of entry points into our code where we have to fully recreate the context of our application state just to do a simple UI update.

##Enter ReactiveCocoa

ReactiveCocoa (RAC) is here to save our bacon, and just maybe give us a little sanity back. Let's look at how.

### RACSignal

`RACSignal` (signal) is the building block for all of RAC. It is an object that represents information that we will eventually receive.

It takes all those methods for controlling the flow of information through your app (delegates, callback blocks, notifications, KVO, target/action event observers, etc) and unifies them under one interface. *This just flat out makes sense.* Not only that, it gives you the ability to transform/split/combine/filter that information easily as it flows through your application.

Instead of designing a screen in an app like this:

--Diagram of information flow in an imperative system. Somehow represent all the entry points and having to reference state.--

We get to design our screen like this:

--Diagram showing the flow of a couple signal chains in a node like format being transformed and combined--

This is so much better, because this is the way we actually design in our minds. Well, it was before we spent far too many years [doing extra work for the computer](http://www.sprynthesis.com/2014/06/15/why-reactivecocoa/).

#### So what is a signal? This is a signal:

--Diagram of a electronic looking box doing nothing--

A signal is an object that sends out a stream of values. But our signal here isn't doing anything. That's because it doesn't have any subscribers. A signal will only send out information if it has a subscriber listening (er, subscribed) to it. It will send that subscriber zero or more "next" events containing the value, followed by either a "complete" event or an "error" event. (A signal is similar to a "promise" in some other languages/toolkits, but far more powerful as it isn't limited to only sending a return value once to it's subscribers.)

--Diagram of our signal with a subscriber now plugged into it via a cord, and values moving along that cord--

Like I mentioned before, you can filter, transforms, split and combine those values as you see necessary.

--Diagram of a signal chain with some funny machines/gears in the middle transforming our initial value and splitting it out into two subscribers (maybe using the isUsernameValid code above as the value and subscribers)--

#### Where do signals get the values they are sending along?

Signals are bits of asynchronous code that wait for something to happen, and then send the resulting value to their subscribers. You can create them manually with the `createSignal` class method on `RACSignal`:

--Code showing creating a signal for something asynchronous like a network call--

Luckily, the creators of RAC actually use their own library to build real things (fathom that), so they have a strong idea what's needed in our daily work. They have provided us with a wealth of mechanisms to pull signals off of the existing asynchronous patterns I mentioned above. Just don't forget that if you have an asynchronous task that isn't covered with some built in signal, you can *easily* create it with `createSignal`, and similar methods.

One such provided mechanism is the `RACObserve()` macro. (If you don't like macros, you can easily look under the hood and use the slightly more verbose representation. It's still great.) This macro is the RAC replacement for the woeful KVO APIs. You just pass in the object and keypath of the property you want to observe on that object. Given those parameters, `RACObserve` generates a signal that immediately send the current value of that property (once it gets a subscriber), and any further changes to that property.

{% highlight objective-c %}
    RACSignal *usernameValidSignal = RACObserve(self.viewModel, isUsernameValid);
{% endhighlight %}

-- diagram showing our signal box with an eyeball watching an a view-model property. --

-- Should I give quick examples of getting signals for the other asynch mechanisms like delegates and events? --

#### What is a subscriber?

Simply put, a subscriber is the bit of code that is waiting for the signal to send along it's values so it can do something with them. (It can also act on the "complete" and "error" events too).

Here is a simple subscriber, created by passing a block to the `subscribeNext` instance method on a signal. Here we are observing the current value of a property on an object via the signal created with the `RACObserve()` macro, and assigning it to an internal property.

{% gist sprynmr/fd3b14a49582c4aa9af1 %}

Notice that RAC only deals in objects, not primitives like `BOOL`. Not to worry though, as RAC mostly handles the conversions for you.

Luckily this sort of binding behavior was also recognized by the RAC creators as a common necessity, so they provided another macro `RAC()`. Similar to `RACObserve()`, you provide the object and the parameter you want the incoming value bound TO, and it does the work under the hood of creating a subscriber and updating the value. Our sample now looks like this:

{% highlight objective-c %}
- (void) viewDidLoad {
    //...
    RAC(self, usernameIsValid) = RACObserve(self.viewModel, isUsernameValid);
}
{% endhighlight %}

But that's a little silly given our goal here. We don't really want to store the value from the signal in a property (and thus create state), what we really want to do is update our UI with the information gleaned from that value.

#### Transforming the streams

Now we're getting into the methods that RAC provides us with for transforming our stream of values. We're going to make use of the `map` instance method on `RACSignal`.

{% gist sprynmr/a731e8026c2143eaf2e3 %}

So now we've bound the changes that occur to `isUsernameValid` on our view-model DIRECTLY to the enabled property on our goButton. How cool is that? The binding to alpha is even cooler, because we are transforming our value into something relevant using the `map` method. (Notice we return an object here. This is basically the one spot where you need to take care of converting your primitive to an object for RAC because it can't derive that for you.)

#### Eliminating even more state

Talk about exposing signals on our view-model instead of plain properties to even further reduce state.

#### Multiple subscribers and side effects

-- diagram of a side effect --

## Tweetboat Plus

Connect the rest of the code

-----

##Remaining:

- RACCommand
    - I think I'm going to try and save RACCommand for my next post, and hopefully signal of signals as well.

- Connect the code together with RAC and short explanations
    - Point out how context is created
    - Explain derived state?


###Additional notes to maybe add somewhere

- Model changes must go through the view model
- Binding data from parent view-models to child view-models
- How does a button in a subview trigger something on the main view-model?
    - Responder chain is helpful
    - Potentially calling up through the view-model chain, but feels weird
    - Typical delegates and block callbacks

---

[^apple-view-models]: In some of the WWDC videos this year, view-models were actually spotted in the sample code apple engineers had on screen. Not sure if any of that made it to the sample code itself.
[^exposing-models]: In practice it's sometimes pragmatic to expose models via the view-model header instead of duplicating a large amount of properties. More on that later.
[^downloading-image]: You could expose the URL instead of the image if you are used to using a category on UIImageView for loading images from the network. That does give a more clear break between the view-model and UIKit, but I view the UIImage itself more as data, and less as the exact presentation of that data. These aren't hard and fast lines.
[^DaveLee]: Thanks to [Dave Lee](https://twitter.com/kastiglione) for exposing this as a good point to cover and for the "View Coordinator" term.
[^uikit-header]: This is a great principle, but there are some grey areas. For instance, you may consider a UIImage "data" and not presentation information. (I like this approach.) In which case you will need UIKit.h so you can work with the UIImage class.
[^andy]: I recently was fortunate enough to listen [Andy Matuschak](http://andymatuschak.org) give a talk along the lines of this concept where he makes a case for a "Thick value layer, thin object layer". The concept is similar, but focuses on how we can remove objects, and their stateful side-effecty nature, and build a more functional, testable value layer with new data structures in swift.
