---
layout: post
title: ReactiveCocoa and MVVM, an Introduction
tags: ReactiveCocoa
---

##MVC

Anyone who has been developing for iOS (or Mac) for a decent length of time is familiar with **MVC**. It stands for **Model View Controller**, and is a proven pattern for organizing your code in complex application design. It's also proven to have a second meaning in iOS development: **Massive View Controller**. It leaves a lot of developers scratching their heads as to how to keep their code nicely decoupled and organized. As a whole, iOS developers have come to the conclusion they need to [slim down their view controllers](http://www.objc.io/issue-1/), and further separate concerns; but how?

##MVVM

That's where **MVVM** comes in. It stands for **Model View View-Model**, and it's here to help us create more manageable, better designed code. 

**Is it unwise to use a pattern other than the one recommended by Apple?**

**No.** For two reasons.

1. Apple hasn't provided any real guidance on how to solve for the Massive View Controller problem. They leave it up to us to figure out how to add some more sanity to our code.[^apple-view-models] MVVM is a pretty awesome way of accomplishing that.
2. MVVM, at least the style of MVVM that I'm going to show here, fits really nicely within the MVC pattern. It's as if we are taking MVC to the next logical step.

I'm not going to cover the history of MVC/MVVM as it's been covered elsewhere, and I'm not that well versed on it. I'm going to focus on how we can use it in iOS/Mac development.

##Defining MVVM

1. **Model** – Essentially the same as the model in MVC. It has no business logic, and is more or less a construct with which to hold information representing a data-model object.
2. **View** - The view encompasses the actual UI itself (whether that means UIView code, storyboards, or xibs), any view specific logic, and reaction to user input. This includes a lot of the responsibilities handled by the `UIViewController` in iOS, as well as `UIView` code.
3. **View-Model** – This is a lot what it sounds like. It's not a model in the traditional data-model object sense, but a model representing the data necessary for the view to display itself. It acts as an interpreter between all the models and web services, etc and the view itself. 

###More about the view-model
One of it's primary purposes is getting data from models and web service calls, and transforming that data into presentation data for the view (controller). It exposes (usually via properties) only the information the view controller needs to know about to do it's job of displaying the view. It's also responsible for making changes to the data (e.g. updating models/database, API calls).


##MVVM in a MVC world
I think the acryonym MVVM is somewhat poor for exactly how we'll be using it for iOS development. Let's take a look at that acronym again and see how it fits into MVC.

For the sake of this diagram, let's reverse the **V** and **C** in **MVC**, so the acronym more accurately reflects the direction of the relationships between the components, leaving us with **MCV**. I'm going to do the same with **MVVM**, moving the **V(iew)** to the right side of **VM** ending up with **MVMV**.

**Here is a mapping of how these two patterns fit together:**

--MCV-MVVM Graphic--

- You can see how the view from MVVM aligns with the view, and half of the view controller from MVC.
- The remaining half of the view controller aligns with the view-model.

For our sake, we aren't actually removing the concept of the view controller or dropping the "controller" term. (Phew.) We are just going to carve out a bunch of it's responsibilities into the view-model, and make the view controller's life much easier. Now the view controller's only concerns are configuring and managing the various views with data from the view-model, and letting the view-model know when relevant user input occurs. The view controller doesn't need to know about web service calls, Core Data, model objects[^exposing-models], etc.

The view-model will live as a property on the view controller. The view controller knows about the view-model, but the view-model knows nothing about the view controller. You should already feel better about this design as we have better separation of concerns going on here.

##View-Model and View Controller, together but separate

Let's look at a simple view-model's header to get a better idea of what our new component looks like. For our simple scenario, let's build a twitter client that lets you lookup the most recent tweets by entering any twitter user username and hitting "Go". Our example will:

- Have a UITextField where the user can enter a twitter username, and a "Go" button
- Have a UIImageView and a UILabel that display the avatar and name of the current user being viewed
- Have a UITableView below that where we view the most recent tweets
- Allow for infinite scrolling

The header of our view-model might look like this:

{% gist sprynmr/433c36a0796f17ec1946 %}

Pretty straightforward stuff. Notice all those **glorious `readonly` properties**? The view-model exposes the minimum information necessary to our view controller, and the view controller really doesn't care how the view-model got that information. Neither do we. Just assume the typical web service calls, validation, and data manipulation and storage.

####Things the view-model isn't doing
- Acting directly on the view controller in any form

###The View Controller

####The view controller would use data FROM the view-model to:

- Toggle the `enabled` property on the "Go" button when the `usernameValid` value changes
- Adjust the `alpha` on the button to be .5 when `usernameValid` equates to NO
- Update the `text` on the UILabel with the string from `userFullName`
- Update the `image` on the UIImageView with the value from `userAvatarImage`
- Configure the table view cells with the objects in `tweets` (More on that later)
- Supply a "loading" cell if `finishedLoading` is NO when reaching the bottom of the table view

####The view controller would act on the view-model as follows:

- Update the only `readwrite` property on the view-model, `username`, whenever the text in the UITextField changes
- Call the `getTweetsForCurrentUsername` on the view-model when the "Go" button is tapped
- Call `loadMoreTweets` on the view-model when the last cell in the table is reached

####Things the view controller isn't doing:

- Making web service calls
- Managing the array of tweets
- Determining what a valid username is
- Formatting the user's first and last name into a full name
- Downloading the user avatar and turning it into a UIImage [^downloading-image]
- Breaking a sweat

##Child View-Models

I mentioned configuring table view cells with objects from the `tweets` array on the view model. Typically you would expect those to be data-model objects representing tweets. You may have been wondering about that, since we try not to expose data-model objects via the MVVM pattern[^exposing-models].

**A view-model doesn't have to represent everything on the screen.** You can use child view-models to represent smaller, potentially more encapsulated portions of the screen. This is especially beneficial if that bit of view (e.g. a table cell) could be re-used in your app and/or represent multiple data-model objects.

You don't always need child view-models. For example, I might use a table header view to render the top section of our view controller. It's not a reusable component, so I might just pass in the same view-model we are already using for the view controller.

In our example, the `tweets` array will be full of child view-models that might look like this:

{% gist sprynmr/adfd5eb3775a225a2011 %}

##Mom, where do view-models come from?

So when and where are view-models created? Do view controllers create their own view-models?

**View-models begat view-models.** Strictly speaking, you should create a view-model for your top view-controller in your app delegate. From there on out, when presenting a new view controller, or bit of view that's represented by a view-model, you ask the current view-model to create it for you.

Say we wanted to add a profile view controller that would show whenever you tapped an avatar on one of the tweets. We might add a method to our current view-model that looked something like this:

{% highlight objective-c %}
- (MYTwitterUserProfileViewModel *) viewModelForUserID:(NSString *)userID;
{% endhighlight %}

and use it like this:

{% gist sprynmr/194ab0c97500592c3954 %}


##Reactive Cocoa

- So how do we update our view controller when these static properties on the view model change?

##Calling things up the chain

- Still some questions about this

###Additional notes

- Exposing the model
- Transformatinos take place on the view-model
- Model changes must go through the view model
- Binding data from parent view-models to child view-models


[^apple-view-models]: In some of the WWDC videos this year, view-models were actually spotted in the sample code apple engineers had on screen. Not sure if any of that made it to the sample code itself.
[^exposing-models]: In practice it's sometimes pragmatic to expose models via the view-model header instead of duplicating a large amount of properties. More on that later.
[^downloading-image]: You could expose the URL instead of the image if you are used to using a category on UIImageView for loading images from the network, but I'd encourage moving away from that.