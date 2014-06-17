---
layout: post
title: Why Reactive(Cocoa)?
tags: ReactiveCocoa
---

When I started working with ReactiveCocoa (RAC) I had very little understanding of what the benefits might be, or what "composing and transforming streams of values" meant. I soon discovered that RAC was far more than another utility class. Instead, it is a framework that facilitates a very different programming paradigm. What I hope to do here is provide some insight on what I've learned, and why you should at least investigate the reactive style of programming. (There has been some fuss recently over the exact terms used to describe the "style" of programming found in ReactiveCocoa, but I still think "Reactive" is a pretty good term myself so I'm not going to shy away from it yet.)

##Why

Anyone who builds things for a living should always be striving to improve their skillset. This means investigating new ways of doing and thinking whenever possible. As programmers one of the hardest things we have to do on a regular basis is to be able to reason about our code[^wwdcvid]; to have a mental map of the implementation used to solve a complex requirement. To me, this is the biggest gain provided by the reactive model of programming. It can remove a significant amount of complexity from our programs and make our code easier to reason about.

For a more authoratative read about reactive programming, you should go read the article ["Inputs and Outputs"](http://blog.maybeapps.com/post/42894317939/input-and-output) by [Josh Aber](https://twitter.com/joshaber), one of the creators of ReactiveCocoa. It's an **extremely** well written article that I can only hope to supplement with a slightly different explanation. (I'll be rehashing a bit of what he's covered there.)

##Inputs and Outputs
So what does Josh mean when he says it's all Inputs and Outputs? The entirety of our job, the meat and potatoes of what we are doing when we build an app, is waiting for events to happen that provide some sort of information (inputs), and then acting on some combination of those inputs and generating some kind of output. Inputs come in all kinds. They provide us with varying levels and types of information:

{% gist sprynmr/db164f896e8615e0abbc %}

This is just a sampling of the different kinds of inputs we deal with daily. The information they provide can be as simple as "This event happened", or be accompanied by a lot of detail about the event such as "The user entered these new characters in the text field". There are certainly reasons for all the different patterns above, not the least of which is the evolution of our craft and tools; but when it comes down to it they are *ALL just telling us some event happened*, and sometimes providing extra information about that event.


Outputs can be anything, but a few typical examples include updating a value on a server, storing something in core data; or most importantly, and most typically, **updating the UI**.

The problem is that we rarely (read: never) are updating our output based on *just one input/event*. I can't put it any better than the way Josh stated it: 

>"To put it another way, the output at any one time is the result of combining all inputs. The output is a function of all inputs up to that time."

I will add one point pertinent to this article. We *often* don't really care what orders these inputs came in from an application design perspective, but from an implementation perspective it's something we are constantly dealing with. In addition, these inputs are littered up and down our source file, all with slightly different interfaces for the developer to remember. We now have many related inputs looking very unrelated in our code, making the work of mentally mapping out their relationships difficult. In other words, it makes it hard to reason about our code.

## Paper Tape Computing (Linear Programming)
The issue here is one of **time**, or more accurately, the timeline of execution. Basically, we program in linear fashion, never wandering all that far from the way things were done on [paper tape computers](https://www.youtube.com/watch?v=uqyVgrplrno) or punch cards. We have an understanding that there is a run loop, and that our code is going to be placed on a timeline and executed in order (ignoring multiple processes for the sake of argument.) Even our awesome block callbacks and delegates are just giving us another snippet of time on the timeline where we can execute code. In fact, all our code is driven by inputs (events) just giving us another chance to insert some paper tape, as shown in this beautiful (and super simplified) diagram.

![Taking turns on the paper tape computer][code-timeline]

Our asynchronous callbacks and other events are still occurring in a linear fashion. The period of time in which the data from these events is available to us is limited to a small spot on our linear execution timeline (scope).[^nested] Even if all these events occurred at the same exact millisecond, the CPU would still have to handle one at a time. Our code is stuck executing in single file, with no knowledge of what happened before it.

Whenever one of these events happens, we likely need to generate output. To do that, we need to combine the new information from this event with all the information from previous events relevant to this particular output. Our blocks and delegate calls may be asynchronous, but we still have to deal with the consequence of where they occured on our timeline.

![Events over time][events]

But how do we do that? The information from those previous events isn't available in the same scope as the current event we're dealing with.[^aside-from-nesting] There's no elegant mechanism provided for accessing past events, or combining the information from all the relevent events. Since these inputs are essentially isolated from each other in this linear style of programming, we have to imperatively (directly) go and check the status of any dynamic information that other inputs could have changed beforehand. That is to say any time there is an event (input), and we're given a chance to run some more paper tape, we do a lot of this type of thinking in code: 

>"Ok lets check what we've got here. User tapped a button eh? Well we're going to want to show the menu then, but a couple things first: Is this user logged in? Good they are. And has the data for this menu loaded in or are we still pulling that from the server? Ahh good we already have that. Finally, the menu isn't already showing is it? Ok it isn't, in which case I'm going to go ahead and show it. Let's write down that it's showing now, so I know in the future."

Basically anytime we get the opportunity to run a bit more code via some event, we are doing so with the memory of a goldfish.[^goldfish] Each time we start with no idea what's happened previously and have to go check a bunch of statuses to build enough context to correctly generate the output we need. The system is "dumb" and doesn't know what information we might need for that particular output, so it can't track it for us. We are left to our own devices to come up with a solution, and end up tracking any possibly relevant inputs ourselves using **state**.

##STATE
State is what makes our job as programmers *VERY* hard sometimes. To be honest, I didn't recognize this at all. That is to say, I knew when I had lots of different events and variables affecting my UI that it became really hard to manage; but I didn't recognize it in a philosophical, definable way. It's so ingrained in me that I just thought it was part of the deal with programming, and that maybe I wasn't the best programmer for constantly struggling with it. 

This is not the case. **Everyone sucks at managing state.** It is the source of an endless amount of bugs and blank stares. "I understand WHY it crashed. I just don't understand how the user could have possibly gotten it in that state." &#8592; Programmer adds another state check.

So what is state? Are we just talking about enums and state machines like `TCQuestionDetailViewControllerState`? Nope. This is all state:

{% highlight objective-c %}
@interface TCQuestionDetailViewController ()
@property (nonatomic, strong) TCQuestion *question;
// .. even more properties
@property (nonatomic, assign) TCQuestionDetailViewControllerState state;
@property (nonatomic, assign) BOOL didAnswerQuestion;
@property (nonatomic, assign) BOOL questionFullyLoaded;
@property (nonatomic, assign) BOOL inResultsView;
@property (nonatomic, assign) BOOL isInModal;
@property (nonatomic, assign) BOOL loadingNextPageOfComments;
@property (nonatomic, assign) BOOL commentsAreShowing;
@property (nonatomic, assign) BOOL commentsLoadingShowing;
@property (nonatomic, assign) BOOL flippingAnswers;
@property (nonatomic, assign) BOOL isOwnerOfQuestion;
@property (nonatomic, assign) BOOL keyboardShowing;
@end
{% endhighlight %}

That should have made you cringe. I'm SURE you've seen (and written) similar code. If not you are a WAY better programmer than I am. UI's are often very complex in code even when they appear simple to the user. In fact, the very reason a user may LOVE your app is the amount of power it delivers in an extremely simple interface.

> Value is created by swallowing complexity for the user.  *- CEO of my first startup*

The problem here is that every time you add another state, you are increasing the possible number of combinations in an **EXPONENTIAL** manner. And that's assuming your states are just BOOLs. I had an "aha!" moment when I saw a slide in a talk by [Justin Spahr-Summers](http://twitter.comjspahrsummers).

![State combinations increasing exponentially][state]

I looked at my code, like the interface above, and realized I was giving myself an unbelievably difficult task, requiring super-geek like abilities to pull off (to the tune of 4000+ different combinations I needed to handle if they were all relevant to the output). Obviously this is HIGHLY error prone.

To make matters worse, this type of code will often produce UI only bugs that are incredibly hard, if not impossible, to identify in any automated way. After a few times through this wringer, we've probably all ended up writing centralized update methods that check a bunch of properties and update things accordingly. We end up with methods like this littered throughout our code: `updateViewCommentStatus` `updateChangeAnswerButtonText`. Any time we add another event (input), we make sure it's updating all the relevant states (e.g. properties) and call the appropriate centralized update methods.

![State change timeline][state-change]

We are expending an awful lot of mental energy dealing with the consequences of this linear code execution thing, this modern version of the paper tape computer. Despite the power of computers today, we are still doing a heck of a lot of work on their behalf. We're programming to the way the computer hardware works; to the way that the run loop is creating a timeline and the cpu is processing bits in a single file. We are architecting our code around low level implementation details of computing. What if we were to harness the power of the computer, let it do more of the work, and allow ourselves to think and design our apps in a more sane manner?

What if we abstract away the whole pesky notion of linear code execution, explain to the computer what combinations of inputs we are interested in for a particular output, and let the computer track state over time for us? Seems like something the computer would be good at; and we're certainly awful at it.

##Compositional Event System (Non-Linear Programming)

I'm not a classically trained, CS Degree wielding programmer. I did a lot of multimedia before ending up programming for my day job. In that time I've seen a lot of tools make the jump from linear to a non-linear style of thinking and working. I'll talk about tools in another post, but even auto-layout is a great example of moving towards a non-linear (and reactive) approach. Instead of waiting for certain events to occur, then checking the status of a bunch of views and imperatively updating frames, you just **describe** what the relationships between all the views are ahead of time and let the computer do the work.

In the discussion about the correct terminology to describe RAC and similar styles of programming, the term "Compositional Event System" has been brought up. I think it fits really well. Much like the shift in the multimedia tools I mentioned, what we really want to do is remove (for the most part) the need to account for the timeline of when events (inputs) happen, and therefor the need to manually keep track of them with state. We want to make the computer do that for us.

We want to define the relationship between events (inputs), define their subsequent outputs, and then let the computer **react** appropriately whenever those events occur. State still exists somewhere in this event system, but it's largely abstracted away from our day to day work. That's a big win. The paper tape still exists, but it's all under the hood.

When you think about it, this concept is actually very simple, and not entirely foreign. Inputs and outputs [^simplified]:

![A home-made 4-bit computer logic board with lights][logic-board]
[Source](http://www.waitingforfriday.com/index.php/4-Bit_Computer)

##ReactiveCocoa

So how does ReactiveCocoa abstract away the timeline for us and step into the world of Non-Linear Programming? It does this by wrapping all those patterns of input from above in one unified interface called a Signal (`RACSignal`). 

Think of a signal as a representation of future values from your input. It's similar to a promise/future which you may have used, but more robust in that it can continually send values (it isn't limited to just one). While this is similar to a KVO block repeatedly being executed as it observes changes, the difference here is that we now have an object representing those future values. We can take that representation and combine it with other signals to build relationships among our events, and describe how data will flow through our app. This is done with powerful operators that can transform and filter our data into new signals, eventually arriving at the data we need for our output.

###Example

Here's a non-trivial example of the typical pattern of combining various inputs into one output imperatively. At a high level, we have a menu that shows when the user taps a button, but there are a bunch of conditions that must be met before that view can be shown:

{% gist sprynmr/876844b9f94530205951 %}

In this example, we're doing all kinds of direct work with state so that we can combine all the events that have happened in our execution timeline and update our output properly. Our code to manage our state and to update our output is *ALL* over the place. Even for a simple example in a short file such as this, it's very hard to keep straight the potential combinations and order of inputs that affect the showing of the menu. Our code is very hard to reason about.

At least we've centralized the state checking and updating of the output to a single method `checkAndUpdateMenuStatus`, but it is still quite the excercise to get and keep your head wrapped around this design. Step away from your code for a few months (weeks? days?) and then you have to read your entire source file to build context around how all the variables/flows work together.

Here is how we might do this in RAC. Don't worry about the specifics of the syntax. Pay attention instead to the concepts:

{% gist sprynmr/63a86ef44a25ee1e10e2 %}

This code will certainly look foreign if you are new to ReactiveCocoa. You should be able to pick up the idea of using a `RACSignal` to get representations of future data. `userIsLoggedIn` will send a `@YES` initially and then a `@NO` if the user logs out. `menuButtonSignal` will send a `@YES` if the menu should be shown or a `@NO` if it should be hidden. We merge those two signals together so we now have a stream of either `@YES`s or `@NO`s that we are using to trigger the `showMenu:` method. `showLoading` is also automatically triggered whenever it receives a `@YES` or `@NO` value from the `.executing` signal on our API command. `RACCommand`s are immensely useful, but also a little beyond the scope of this article, so don't worry about them too much.

Hopefully you can take away some of the beauty of describing the relationships of your inputs, the flow of data through your logic, and your outputs, all in one place. Even with a sizeable amount of comments, I still end up with less code[^lesscode]. I'm not worrying about the timeline of what event happens when (as much) or dealing with entry points all across my file. I can come back to this code in a couple months and quickly grok what is going on. It is a great deal easier to reason about my code.

---
**Footnotes**


[^nested]: 
    Technically in nested inputs such as the two API calls here, the child would have access to the parent event data via a closure. This is a little helpful for outputs that have only a dependency on their ancestors, but not a very robust solution (Ugh):
    {% gist sprynmr/4104d506db2189b742ed %}

[^aside-from-nesting]: Aside from nesting. See [^nested]
[^goldfish]: Unfortunately for this analogy, goldfish can actually remember up to several months. Even three seconds of memory sort of defeats the point. More like a goldfish missing the part of it's brain responsible for memory. [Reference](http://www.dailymail.co.uk/sciencetech/article-1106884/Three-second-memory-myth-Fish-remember-months.html)
[^simplified]: This is intentionally simplified, and doesn't really worry about the reality of side effects required in the real world of iOS/Mac Development
[^wwdcvid]: As I was almost finished with this article, I watched the 2014 WWDC session ["Advanced iOS Application Architecture and Patterns"](http://devstreaming.apple.com/videos/wwdc/2014/229xx77tq0pmkwo/229/229_hd_advanced_ios_architecture_and_patterns.mov?dl=1) by [Andy Matuschak] (http://www.twitter.com/andy_matuschak). It's a great session, and highly relevant to the topic at hand. In fact the whole initial segment of the talk where he talks about the importance of being able to "reason about our code" could just as easily be a segment about "Why ReactiveCocoa".
[^lesscode]: This isn't always the case. But even if you end up with a little more code, it usually means substantially more clarity.

[code-timeline]: /assets/images/code-timeline.png "Taking turns on the paper tape computer"
[events]: /assets/images/events.png "We're all just dots on this groovy timeline, man."
[state]: /assets/images/state.png "Alright. Shut it down. We blew it."
[state-change]: /assets/images/state-change.png "Take Illustrator away from this guy."
[logic-board]: /assets/images/logic-board.jpg "Man that seems satisfying."