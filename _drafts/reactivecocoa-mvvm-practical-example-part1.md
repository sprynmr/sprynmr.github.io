---
layout: post
title: ReactiveCocoa and MVVM, a practical example Part 1
tags: ReactiveCocoa
---

--MVC Graphic--

Anyone who has been developing for iOS (or Mac) for a decent length of time is familiar with MVC. It stands for **Model View Controller**, and is a proven pattern for organizing your code for complex application design. It's also proven to have a second meaning, **Massive View Controller**. It leaves a lot of developers scratching their heads as to how to keep their code nicely decoupled. After awhile many iOS developers independently come to the conclusion that they need to slim down their view controllers, and further separate concerns.

--MVVM Graphic--

That's where MVVM comes in. It stands for **Model View View-Model**, and it's here to help us create more manageable, better designed code. One of it's primary purposes is to separate out the task of getting data from models and web service calls, and transforming that data into presentation data for the view (controller). It exposes (usually via properties) just the information the view controller needs to know about to do it's job of displaying the view. It's also exclusively responsible for making changes to the data (both in the models and via API calls).

**Is it unwise to use a pattern other than the one recommended by Apple?**

**No.** For two reasons.

1. Apple hasn't provided any real guidance on how to solve for the Massive View Controller problem. They leave it up to every team of engineers to figure out how to add some more sanity to their code. From my experience there is usually no clear agreement on how to do this, and it things become messy when individuals end up cobbling together their own solution.[^apple-view-models]
2. MVVM, at least the style of MVVM that I'm going to show here, really fits nicely within the MVC pattern. It's as if we are taking MVC to the next logical step and setting up some rules for how we divide up the View Controller.

I'm not going to cover the history of MVC/MVVM as it's been covered elsewhere, and I'm not that well versed on it. I'm going to focus on how we can use it in iOS/Mac development.

##Defining MVVM

1. **Model** – Essentially the same as the model in MVC. It has no business logic, and is more or less a construct with which to hold information about an object.
2. **View** - In our world view encompasses the actual UI itself (whether that means UIView code, storyboards, or xibs), any view specific logic, and reaction to user input. This includes a lot of the responsibilities handled by the `UIViewController` in iOS, as well as `UIView` code.
3. **View Model** – This is a lot what it sounds like. It's a model not representing an object, but representing the data necessary for the view to display itself. It acts as an interpreter between all the models and web services, etc and the view itself.

##MVVM in a MVC world
I think the acryonym MVVM is somewhat poor for exactly how we'll be using it for iOS development. Let's take a look at that acronym again and see how it fits into MVC.

For the sake of this diagram, let's reverse the **V** and **C** in **MVC**, so the acronym more accurately reflects the direction of the relationships between the components. I'm going to do the same with MVVM, moving the **V(iew)** to the right side of **VM** ending up with **MVMV**. (Why these two acronyms aren't like this in the first place is a mystery to me, but I don't care enough to research it.)

**Here is a mapping of how these two patterns fit together.**

--MCV Graphic--

- You can see the view from MVVM aligns with the view and half of the view controller.
- The remaining half of the view controller aligns with the view model.

For our sake, we aren't actually removing the concept of the view controller or dropping the "controller" term. (Phew.) We are just going to carve out a bunch of it's responsibilities into the view model, and make it's life much easier. Now the view controller's only concerns are configuring and managing the various views with data from the view model, and letting the view model know when relevant user input occurs.

[^apple-view-models]: In some of the WWDC videos this year, view models were actually spotted in the sample code apple engineers had on screen. Not sure if any of that made it to the sample code itself.