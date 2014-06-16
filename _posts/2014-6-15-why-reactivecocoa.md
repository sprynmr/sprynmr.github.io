---
layout: post
title: Why Reactive(Cocoa)?
---
Reason about our code
Multiple entry points all over the file (hard to reason about)

I had very little understanding of what the benefits might be when I began to learn ReactiveCocoa (RAC); but I love learning new hip things and RAC came with buzzwords! It turned out to be far more than a utility class, in fact facilitating a very different programming paradigm. What I hope to do here is provide some insight on what I've learned, and why you should at least investigate the Reactive style of programming. (There has been some fuss recently over the exact terms used to describe the "style" of programming found in ReactiveCocoa, but I still think Reactive is a pretty good term myself so I'm not going to shy away from it yet.)

To start you should probably go read the article ["Inputs and Outputs"](http://blog.maybeapps.com/post/42894317939/input-and-output) by [Josh Aber](https://twitter.com/joshaber). It's an **extremely** well written article that I can only hope to supplement with a slightly different explanation. (I'll be rehashing a bit of what he's covered there.)

##Inputs and Outputs
So what does Josh mean when he says it's all Inputs and Outputs? The entirety of our job, the meat and potatoes of what we are doing when we build an app, is waiting for events to happen that provide some sort of information (inputs), and then acting on some combination of those inputs and generating some kind of output. Inputs come in all kinds. They provide us with varying levels and types of information:

{% gist sprynmr/db164f896e8615e0abbc %}

This is just a sampling of the different kinds of inputs we deal with daily. The information they provide can be as simple as "This event happened", or be accompanied by a lot of detail about the event "The user entered these new characters in the text field". There are certainly reasons for all the different patterns above, not the least of which is the evolution of our craft and tools; but when it comes down to it they are *ALL just telling us some event happened*, and sometimes providing extra information about that event.

Outputs can be anything, but a few typical examples include updating a value on a server, storing something in core data; or most importantly, and most typically, **updating the UI**.

The problem is that we rarely (read: never) are updating our output based on *just one input/event*. I can't put it any better than the way Josh stated it: 

>"To put it another way, the output at any one time is the result of combining all inputs. The output is a function of all inputs up to that time."

I will add one huge point pertinent to this article. We *often* don't really care what orders these inputs came in from an application design perspective, but from an implementation perspective it's something we are constantly dealing with.

## Paper Tape Computing (Linear Programming)
The issue here is one of **time**, or more accurately, the timeline of execution. Basically, we program in linear fashion, never wandering all that far from the way things were done on [paper tape computers](https://www.youtube.com/watch?v=uqyVgrplrno) or punch cards. We have an understanding that there is a run loop, and that our code is going to be placed on a timeline and executed in order (ignoring multiple processes for the sake of argument.) Even our awesome block callbacks and delegates are just giving us another snippet of time on the timeline where we can execute code. In fact all our code is driven by inputs (events) just giving us another chance to insert some paper tape, as shown in this beautiful (and super simplified) diagram.

![Taking turns on the paper tape computer][code-timeline]

Our sophisticated, "asynchronous" callbacks and other events are still occurring in a linear fashion. The period of time in which the data from these events is available to us is limited to a small spot on our linear execution timeline (scope).[^1] Even if all these events occurred at the exact same millisecond, the CPU would still have to handle one at a time. Our code is stuck executing in single file, with no knowledge of what happened before it.

Whenever one of these events happens, we likely need to generate output. To do that, we need to combine the new information from this event with all the information from previous events relevant to this particular output.

![Events over time][events]

But how do we do that? The information from those previous events isn't available in the same scope as the current event we're dealing with.[^2] There's no elegant mechanism provided for accessing past events, or combining the information from all the relevent events. Since these inputs are essentially isolated from each other in this linear style of programming, we have to imperatively (directly) go and check the status of any dynamic information that other inputs could have changed beforehand. That is to say any time there is an event (input), and we're given a chance to run some more paper tape, we do a lot of this type of thinking in code: 

>"Ok lets check what we've got here. User tapped a button eh? Well we're going to want to show the menu then, but a couple things first: Is this user logged in? Good they are. And has the data for this menu loaded in or are we still pulling that from the server? Ahh good we already have that. Finally, the menu isn't already showing is it? Ok it isn't, in which case I'm going to go ahead and show it. Let's write down that it's showing now, so I know in the future."

Basically anytime we get the opportunity to run a bit more code via some event, we are doing so with the memory of a goldfish.[^3] Each time we start with no idea what's happened previously and have to go check a bunch of statuses to build enough context to correctly generate the output we need. The system is "dumb" and doesn't know what information we might need for that particular output, so it can't track it for us. We are left to our own devices to come up with a solution, and end up tracking any possibly relevant inputs ourselves using **state**.

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

To make matters worse, this type of code will often produce UI only bugs that are incredibly hard, if not impossible, to identify in any automated way. After a few times through this wringer, we've probably all ended up writing centralized update methods that check a bunch of properties and update things accordingly. We end up with methods like this littered throughout our code: `updateViewCommentStatus` `updateChangeAnswerButtonText`. Any time we add another event (input), we make sure it's updating all the appropriate states (e.g. properties) and throw in the appropriate centralized update methods. Now our previously clean looking timeline is getting unmanagable:

![State change timeline][state-change]

How can we possibly keep this all straight in our heads? We are expending an awful lot of brain power dealing with the consequences of this linear code execution thing, this modern version of the paper tape computer. Despite the power of computers today, we are still doing a heck of a lot of work on their behalf. We're programming to the way the computer hardware works; to the way that the run loop is creating a timeline and the cpu is processing bits in a single file. We are architecting our code around low level implementation details of computing. What if we were to harness the power of the computer, let it do more of the work, and allow ourselves to think and design our apps in a more sane manner?

What if we abstract away the whole pesky notion of linear code execution, explain to the computer what combinations of inputs we are interested in for a particular output, and let the computer track state over time for us? Seems like something the computer would be good at; and we're certainly awful at it.

##Compositional Event System (Non-Linear Programming)

I'm not a classically trained, CS Degree wielding programmer. I did a lot of multimedia before ending up programming for my day job. In that time I've seen a lot of tools make the jump from linear to a non-linear style of thinking and working. I'll talk about tools in another post, but even auto-layout is a great example of moving towards a non-linear (and reactive) approach. Instead of waiting for certain events to occur, then checking the status of a bunch of views and imperatively updating frames, you just **describe** what the relationships between all the views are ahead of time and let the computer do the work.

In the discussion about the correct terminology to describe, RAC and similar styles of programming, the term "Compositional Event System" has been thrown around. I think it fits really well. Much like the shifts in the multimedia tools above, what we really want to do is remove (for the most part) the need to account for the timeline of when events (inputs) happen, and therefor the need to manually keep track of them with state. We want to make the computer do that for us.

We want to define the relationship between events (inputs), define their subsequent outputs, and then let the computer **react** appropriately whenever those events occur. State still exists somewhere in this event system, but it's largely abstracted away from our day to day work. That's a big win. The paper tape still exists, but it's all under the hood.

When you think about it, this concept is actually very simple, and not entirely foreign. Inputs and outputs [^4]:

![A home-made 4-bit computer logic board with lights][logic-board]
[Source](http://www.waitingforfriday.com/index.php/4-Bit_Computer)

##ReactiveCocoa

So how does ReactiveCocoa abstract away the timeline for us and step into the world of Non-Linear Programming? It does this by wrapping all those patterns of input from above in one unified interface called a Signal (`RACSignal`). Then it gives us operators so that we can combine, split, and filter those events.

Here's an example of the typical pattern of combining various inputs into one output. At a high level, we have a menu that shows when the user taps a button, but there are a bunch of conditions that must be met before that view can be shown:

{% highlight objective-c %}
// a central function that checks all our states and generates the appropriate output
- (void) checkAndUpdateMenuStatus {
    if (self.menuShouldBeShowing && !self.isMenuShowing 
        && self.menuDataIsLoaded && self.userIsLoggedIn) {
        [self showMenu];
    } else if (!self.menuShouldBeShowing && self.isMenuShowing) {
        [self hideMenu];
    }
}
// sets initial states and sets up our notification observation
- (void) viewDidLoad {
    // Set initial states
    // Let's assume you can't get to this page without being logged in
    self.userIsLoggedIn = YES;
    self.isMenuShowing = NO;
    self.menuDataIsLoaded = NO;
    self.menuShouldBeShowing = NO;
    // Need to handle in case the user logs out while on this page
    [[NSNotificationCenter defaultCenter] addObserverForName:kUserLoggedOutNotification 
                                                      object:nil 
                                                       queue:nil 
                                                  usingBlock:^(NSNotification *note) {
        self.userIsLoggedIn = NO;
        [self checkAndUpdateMenuStatus];
    }];
    // set the initial state (somewhat unnecessary since our menu starts hidden
    // but a good safety check)
    [self checkAndUpdateMenuStatus];
}
// Loads the menu data from the network
- (void) loadMenuData {
    [TCPAPI fetchUserMenuData onComplete:^(NSArray *objects, NSError *error) {
        if(!error) {
            [self hideLoadingView];
        } else {
            self.menuDataIsLoaded = YES;
            [self checkAndUpdateMenuStatus];
        }
    }];
}
// handles showing and hiding a loading view
- (void) startLoadingView {
    if(self.isLoadingShowing) return;
    self.isLoadingShowing = YES;
    // do work to show loading view
}
- (void) hideLoadingView {
    if(!self.isLoadingShowing) return;
    self.isLoadingShowing = NO;
    // do work to hide loading view
}
- (void) showMenu {
    // show menu
    self.isMenuShowing = YES;
}
- (void) hideMenu {
    // hide menu
    self.isMenuShowing = NO;
}
- (IBAction) userTappedMenuButton:(UIButton *menuButton) {
    // kick off loading of our menu data lazily if it isn't loaded yet
    if(!self.menuDataIsLoaded) {
        [self loadMenuData];
    }
    self.menuShouldBeShowing = !self.menuShouldBeShowing;
    [self checkAndUpdateMenuStatus];
}
{% endhighlight %}

In this example, we're doing all kinds of direct work with state so that we can combine all the events that have happened in our execution timeline and update our output properly. Our code to manage our state and to update our output is *ALL* over the place. Even for a simple example in a short file such as this, it's very hard to keep straight the potential combinations and order of inputs that affect the showing of the menu.

At least we've centralized the state checking and updating of the output to a single method `checkAndUpdateMenuStatus`, but it is still quite the excercise to get and keep your head wrapped around this design. Step away from your code for a few months (weeks? days?) and then you have to read your entire source file to build context around how all the variables/flows work together.

Here is how we might do this in RAC. Don't worry about the specifics of how exactly all these bits are working. I'll cover that in another post:

{% highlight objective-c %}
- (void) viewDidLoad {
    [super viewDidLoad];
    
    // RACCommand is a special object that returns a signal, provides an additional `executing` signal until it completes, and allows for 
    // multiple subscriptions to the returned signal without triggering multiple side effects (API call is only made once)
    self.fetchUserMenuDataCommand = [[RACCommand alloc] initWithSignalBlock:^RACSignal *(id input) {
            // this API returns a RACSignal
            return [TCAPI fetchUserMenuData];
        }];

    // Every time the command is executed it returns a new signal
    // the `switchToLatest` operator gives us the most recent signal, which is all we care about
    // We bind the result of that signal to our property (derive state instead of manually setting it)
    // This IS state, but it's necessary, derived state that we aren't managing manually
    RAC(self, menuData) = [self.fetchUserMenuDataCommand.executionSignals switchToLatest];

    // A signal that sends a value if the user logs out
    // starts with an initial value of @YES (the user starts logged in)
    RACSignal *userIsLoggedIn = [[[[NSNotificationCenter defaultCenter] rac_addObserverForName:kUserLoggedOutNotification object:nil]
        mapReplace:@NO]
        startWith: @YES];

    @weakify(self);
    RACCommand *menuButtonPressed = [[RACCommand alloc] initWithEnabled:userIsLoggedIn signalBlock:^RACSignal *(id input) {
            @strongify(self);
            // We already have implicit state `hidden` on the menu we don't have to manage
            if (self.menuView.hidden) {
                if (self.menuData != nil) {
                    return [RACSignal return: @YES];
                }
                return [[self.fetchUserMenuDataCommand execute:nil]
                    mapReplace:@YES];
            } else {
                return @NO;
            }
        }];

    // Use `switchToLatest` to make sure we're listening to the most recent signal
    // This signal will send values either @YES or @NO from the command that we'll use to
    RACSignal *menuButtonSignal = [menuButtonPressed.executionSignals switchToLatest];

    // We also want to hide the menu if the user logs out while it's showing
    // So we merge in the values from the `userIsLoggedIn` signal and ignore @YES
    // We'll use this combined signal to fire the `showMenu` selector below
    RACSignal *showHideMenuSignal = [RACSignal merge:@[[menuButtonSignal], [userIsLoggedIn ignore:@YES]]];

    // Automatically fires the selector with the value by the signal the menuButtonPressed command returns
    // `showMenu` implementation left out, but it takes a BOOL and will either show or hide a view
    [self rac_liftSelector:@selector(showMenu:) withSignals: showHideMenuSignal];

    // We also automatically call the `showLoading` selector when our network command is still executing (before it's signal returns it's value)
    // `showLoading` implementation left out, but it takes a BOOL and will either show or hide a loading view
    [self rac_liftSelector:@selector(showLoading:) withSignals: self.fetchUserMenuDataCommand.executing];
}
{% endhighlight %}

[^1]: 
    Technically in nested inputs such as the two API calls here, the child would have access to the parent event data via a closure. This is a little helpful for outputs that have only a dependency on their ancestors, but not a very robust solution (Ugh):
    {% gist sprynmr/4104d506db2189b742ed %}

[^2]: Aside from nesting. See [^1]
[^3]: Unfortunately for this analogy, goldfish can actually remember up to several months. Even three seconds of memory sort of defeats the point. More like a goldfish missing the part of it's brain responsible for memory. [Reference](http://www.dailymail.co.uk/sciencetech/article-1106884/Three-second-memory-myth-Fish-remember-months.html)
[^4]: This is intentionally simplified, and doesn't really worry about the reality of side effects required in the real world of iOS/Mac Development

[code-timeline]: /assets/images/code-timeline.png "Taking turns on the paper tape computer"
[events]: /assets/images/events.png "We're all just dots on this groovy timeline, man."
[state]: /assets/images/state.png "Alright. Shut it down. We blew it."
[state-change]: /assets/images/state-change.png "Take Illustrator away from this guy."
[logic-board]: /assets/images/logic-board.jpg "Man that seems satisfying."