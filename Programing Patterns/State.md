Confession time: I went a little overboard and packed way too much into this chapter. It’s ostensibly about the [State](http://en.wikipedia.org/wiki/State_pattern) design pattern, but I can’t talk about that and games without going into the more fundamental concept of _finite state machines_ (or “FSMs”). But then once I went there, I figured I might as well introduce _hierarchical state machines_ and _pushdown automata_.

That’s a lot to cover, so to keep things as short as possible, the code samples here leave out a few details that you’ll have to fill in on your own. I hope they’re still clear enough for you to get the big picture.

Don’t feel sad if you’ve never heard of a state machine. While well known to AI and compiler hackers, they aren’t that familiar to other programming circles. I think they should be more widely known, so I’m going to throw them at a different kind of problem here.

## [We’ve All Been There](https://www.gameprogrammingpatterns.com/state.html#we've-all-been-there)

We’re working on a little side-scrolling platformer. Our job is to implement the heroine that is the player’s avatar in the game world. That means making her respond to user input. Push the B button and she should jump. Simple enough:

```gdscript
func handle_input(input):
	if input == PRESS_B:
		y_velocity = JUMP_VELOCITY
		set_graphics(IMAGE_JUMP)
```

Spot the bug?

There’s nothing to prevent “air jumping” — keep hammering B while she’s in the air, and she will float forever. The simple fix is to add an `isJumping_` Boolean field to `Heroine` that tracks when she’s jumping, and then do:

```gdscript
func handle_input(input):
	if input == PRESS_B:
		if not is_jumping:
			is_jumping = true
			# Jump...
```


There should also be code that sets `isJumping_` back to `false` when the heroine touches the ground. I’ve omitted that here for brevity’s sake.

Next, we want the heroine to duck if the player presses down while she’s on the ground and stand back up when the button is released:

```gdscript
func handle_input(input):
	if input == PRESS_B:
		# Jump if not jumping...
	elif input == PRESS_DOWN:
		if not is_jumping:
			set_graphics(IMAGE_DUCK)
	elif input == RELEASE_DOWN:
		set_graphics(IMAGE_STAND)
```

Spot the bug this time?

With this code, the player could:

1. Press down to duck.
2. Press B to jump from a ducking position.
3. Release down while still in the air.

The heroine will switch to her standing graphic in the middle of the jump. Time for another flag…

```gdscript
func handle_input(input):
	if input == PRESS_B:
		if not is_jumping and not is_ducking:
			# Jump...
	elif input == PRESS_DOWN:
		if not is_jumping:
			is_ducking = true
			set_graphics(IMAGE_DUCK)
	elif input == RELEASE_DOWN:
		if is_ducking:
			is_ducking = false
			set_graphics(IMAGE_STAND)
```

Next, it would be cool if the heroine did a dive attack if the player presses down in the middle of a jump:

```gdscript
func handle_input(input):
	if input == PRESS_B:
		if not is_jumping and not is_ducking:
			# Jump...
	elif input == PRESS_DOWN:
		if not is_jumping:
			is_ducking = true
			set_graphics(IMAGE_DUCK)
		else:
			is_jumping = false
			set_graphics(IMAGE_DIVE)
	elif input == RELEASE_DOWN:
		if is_ducking:
			# Stand...
```

Bug hunting time again. Find it?

We check that you can’t air jump while jumping, but not while diving. Yet another field…

Something is clearly wrong with our approach. Every time we touch this handful of code, we break something. We need to add a bunch more moves — we haven’t even added _walking_ yet — but at this rate, it will collapse into a heap of bugs before we’re done with it.

Those coders you idolize who always seem to create flawless code aren’t simply superhuman programmers. Instead, they have an intuition about which _kinds_ of code are error-prone, and they steer away from them.

Complex branching and mutable state — fields that change over time — are two of those error-prone kinds of code, and the examples above have both.

## [Finite State Machines to the Rescue](https://www.gameprogrammingpatterns.com/state.html#finite-state-machines-to-the-rescue)

In a fit of frustration, you sweep everything off your desk except a pen and paper and start drawing a flowchart. You draw a box for each thing the heroine can be doing: standing, jumping, ducking, and diving. When she can respond to a button press in one of those states, you draw an arrow from that box, label it with that button, and connect it to the state she changes to.

![A flowchart containing boxes for Standing, Jumping, Diving, and Ducking. Arrows for button presses and releases connect some of the boxes.](https://www.gameprogrammingpatterns.com/images/state-flowchart.png)

Congratulations, you’ve just created a _finite state machine_. These came out of a branch of computer science called _automata theory_ whose family of data structures also includes the famous Turing machine. FSMs are the simplest member of that family.

The gist is:

- **You have a fixed _set of states_ that the machine can be in.** For our example, that’s standing, jumping, ducking, and diving.
    
- **The machine can only be in _one_ state at a time.** Our heroine can’t be jumping and standing simultaneously. In fact, preventing that is one reason we’re going to use an FSM.
    
- **A sequence of _inputs_ or _events_ is sent to the machine.** In our example, that’s the raw button presses and releases.
    
- **Each state has a _set of transitions_, each associated with an input and pointing to a state.** When an input comes in, if it matches a transition for the current state, the machine changes to the state that transition points to.
    
    For example, pressing down while standing transitions to the ducking state. Pressing down while jumping transitions to diving. If no transition is defined for an input on the current state, the input is ignored.
    

In their pure form, that’s the whole banana: states, inputs, and transitions. You can draw it out like a little flowchart. Unfortunately, the compiler doesn’t recognize our scribbles, so how do we go about _implementing_ one? The Gang of Four’s State pattern is one method — which we’ll get to — but let’s start simpler.

My favorite analogy for FSMs is the old text adventure games like Zork. You have a world of rooms that are connected to each other by exits. You explore them by entering commands like “go north”.

This maps directly to a state machine: Each room is a state. The room you’re in is the current state. Each room’s exits are its transitions. The navigation commands are the inputs.

## [Enums and Switches](https://www.gameprogrammingpatterns.com/state.html#enums-and-switches)

One problem our `Heroine` class has is some combinations of those Boolean fields aren’t valid: `isJumping_` and `isDucking_` should never both be true, for example. When you have a handful of flags where only one is `true` at a time, that’s a hint that what you really want is an `enum`.

In this case, that `enum` is exactly the set of states for our FSM, so let’s define that:

```gdscript
enum State {
	STATE_STANDING,
	STATE_JUMPING,
	STATE_DUCKING,
	STATE_DIVING,
}
```

Instead of a bunch of flags, `Heroine` will just have one `state_` field. We also flip the order of our branching. In the previous code, we switched on input, _then_ on state. This kept the code for handling one button press together, but it smeared around the code for one state. We want to keep that together, so we switch on state first. That gives us:

```gdscript
func handle_input(input):
	match state:
		State.STATE_STANDING:
			if input == PRESS_B:
				state = State.STATE_JUMPING
				y_velocity = JUMP_VELOCITY
				set_graphics(IMAGE_JUMP)
			elif input == PRESS_DOWN:
				state = State.STATE_DUCKING
				set_graphics(IMAGE_DUCK)
		State.STATE_JUMPING:
			if input == PRESS_DOWN:
				state = State.STATE_DIVING
				set_graphics(IMAGE_DIVE)
		State.STATE_DUCKING:
			if input == RELEASE_DOWN:
				state = State.STATE_STANDING
				set_graphics(IMAGE_STAND)
```

This seems trivial, but it’s a real improvement over the previous code. We still have some conditional branching, but we simplified the mutable state to a single field. All of the code for handling a single state is now nicely lumped together. This is the simplest way to implement a state machine and is fine for some uses.

In particular, the heroine can no longer be in an _invalid_ state. With the Boolean flags, some sets of values were possible but meaningless. With the `enum`, each value is valid.

Your problem may outgrow this solution, though. Say we want to add a move where our heroine can duck for a while to charge up and unleash a special attack. While she’s ducking, we need to track the charge time.

We add a `chargeTime_` field to `Heroine` to store how long the attack has charged. Assume we already have an `update()` that gets called each frame. In there, we add:

```gdscript
func _process(delta):
	if state == State.STATE_DUCKING:
		charge_time += 1
		if charge_time > MAX_CHARGE:
			super_bomb()
```

If you guessed that this is the [Update Method](https://www.gameprogrammingpatterns.com/update-method.html) pattern, you win a prize!

We need to reset the timer when she starts ducking, so we modify `handleInput()`:

```gdscript
func handle_input(input):
	match state:
		State.STATE_STANDING:
			if input == PRESS_DOWN:
				state = State.STATE_DUCKING
				charge_time = 0
				set_graphics(IMAGE_DUCK)
			# Handle other inputs...
```

All in all, to add this charge attack, we had to modify two methods and add a `chargeTime_` field onto `Heroine` even though it’s only meaningful while in the ducking state. What we’d prefer is to have all of that code and data nicely wrapped up in one place. The Gang of Four has us covered.

## [The State Pattern](https://www.gameprogrammingpatterns.com/state.html#the-state-pattern)

For people deeply into the object-oriented mindset, every conditional branch is an opportunity to use dynamic dispatch (in other words a virtual method call in C++). I think you can go too far down that rabbit hole. Sometimes an `if` is all you need.

There’s a historical basis for this. Many of the original object-oriented apostles like _Design Patterns_‘ Gang of Four, and _Refactoring_‘s Martin Fowler came from Smalltalk. There, `ifThen:` is just a method you invoke on the condition, which is implemented differently by the `true` and `false` objects.

But in our example, we’ve reached a tipping point where something object-oriented is a better fit. That gets us to the State pattern. In the words of the Gang of Four:

> Allow an object to alter its behavior when its internal state changes. The object will appear to change its class.

That doesn’t tell us much. Heck, our `switch` does that. The concrete pattern they describe looks like this when applied to our heroine:

### [A state interface](https://www.gameprogrammingpatterns.com/state.html#a-state-interface)

First, we define an interface for the state. Every bit of behavior that is state-dependent — every place we had a `switch` before — becomes a virtual method in that interface. For us, that’s `handleInput()` and `update()`:

GDScript has no interfaces or virtual methods, so use base classes:
```gdscript
class_name HeroineState

func handle_input(heroine, input):
	pass

func update(heroine):
	pass
```

### [Classes for each state](https://www.gameprogrammingpatterns.com/state.html#classes-for-each-state)

For each state, we define a class that implements the interface. Its methods define the heroine’s behavior when in that state. In other words, take each `case` from the earlier `switch` statements and move them into their state’s class. For example:

```gdscript
class_name DuckingState
extends HeroineState

var charge_time = 0

func handle_input(heroine, input):
	if input == RELEASE_DOWN:
		heroine.set_graphics(IMAGE_STAND)

func update(heroine):
	charge_time += 1
	if charge_time > MAX_CHARGE:
		heroine.super_bomb()
```

Note that we also moved `chargeTime_` out of `Heroine` and into the `DuckingState` class. This is great — that piece of data is only meaningful while in that state, and now our object model reflects that explicitly.

### [Delegate to the state](https://www.gameprogrammingpatterns.com/state.html#delegate-to-the-state)

Next, we give the `Heroine` a pointer to her current state, lose each big `switch`, and delegate to the state instead:

```gdscript
func handle_input(input):
	state.handle_input(self, input)

func _process(delta):
	state.update(self)
```

In order to “change state”, we just need to assign `state_` to point to a different `HeroineState` object. That’s the State pattern in its entirety.

This looks like the [Strategy](http://en.wikipedia.org/wiki/Strategy_pattern) and [Type Object](https://www.gameprogrammingpatterns.com/type-object.html) patterns. In all three, you have a main object that delegates to another subordinate one. The difference is _intent_.

- With Strategy, the goal is to _decouple_ the main class from some portion of its behavior.
    
- With Type Object, the goal is to make a _number_ of objects behave similarly by _sharing_ a reference to the same type object.
    
- With State, the goal is for the main object to _change_ its behavior by _changing_ the object it delegates to.
    

## [Where Are the State Objects?](https://www.gameprogrammingpatterns.com/state.html#where-are-the-state-objects)

I did gloss over one bit here. To change states, we need to assign `state_` to point to the new one, but where does that object come from? With our `enum` implementation, that was a no-brainer — `enum` values are primitives like numbers. But now our states are classes, which means we need an actual instance to point to. There are two common answers to this:

### [Static states](https://www.gameprogrammingpatterns.com/state.html#static-states)

If the state object doesn’t have any other fields, then the only data it stores is a pointer to the internal virtual method table so that its methods can be called. In that case, there’s no reason to ever have more than one instance of it. Every instance would be identical anyway.

If your state has no fields and only _one_ virtual method in it, you can simplify this pattern even more. Replace each state _class_ with a state _function_ — just a plain vanilla top-level function. Then, the `state_` field in your main class becomes a simple function pointer.

In that case, you can make a single _static_ instance. Even if you have a bunch of FSMs all going at the same time in that same state, they can all point to the same instance since it has nothing machine-specific about it.

This is the [Flyweight](https://www.gameprogrammingpatterns.com/flyweight.html) pattern.

_Where_ you put that static instance is up to you. Find a place that makes sense. For no particular reason, let’s put ours inside the base state class:

```gdscript
class_name HeroineState

static var standing = StandingState.new()
static var ducking = DuckingState.new()
static var jumping = JumpingState.new()
static var diving = DivingState.new()
```

Each of those static fields is the one instance of that state that the game uses. To make the heroine jump, the standing state would do something like:

```gdscript
if input == PRESS_B:
	heroine.state = HeroineState.jumping
	heroine.set_graphics(IMAGE_JUMP)
```

### [Instantiated states](https://www.gameprogrammingpatterns.com/state.html#instantiated-states)

Sometimes, though, this doesn’t fly. A static state won’t work for the ducking state. It has a `chargeTime_` field, and that’s specific to the heroine that happens to be ducking. This may coincidentally work in our game if there’s only one heroine, but if we try to add two-player co-op and have two heroines on screen at the same time, we’ll have problems.

In that case, we have to create a state object when we transition to it. This lets each FSM have its own instance of the state. Of course, if we’re allocating a _new_ state, that means we need to free the _current_ one. We have to be careful here, since the code that’s triggering the change is in a method in the current state. We don’t want to delete `this` out from under ourselves.

Instead, we’ll allow `handleInput()` in `HeroineState` to optionally return a new state. When it does, `Heroine` will delete the old one and swap in the new one, like so:

```gdscript
func handle_input(input):
	var new_state = state.handle_input(self, input)
	if new_state != null:
		state = new_state
```

That way, we don’t delete the previous state until we’ve returned from its method. Now, the standing state can transition to ducking by creating a new instance:

```gdscript
func handle_input(heroine, input):
	if input == PRESS_DOWN:
		# Other code...
		return DuckingState.new()
	return null
```

When I can, I prefer to use static states since they don’t burn memory and CPU cycles allocating objects each state change. For states that are more, uh, _stateful_, though, this is the way to go.

When you dynamically allocate states, you may have to worry about fragmentation. The [Object Pool](https://www.gameprogrammingpatterns.com/object-pool.html) pattern can help.

## [Enter and Exit Actions](https://www.gameprogrammingpatterns.com/state.html#enter-and-exit-actions)

The goal of the State pattern is to encapsulate all of the behavior and data for one state in a single class. We’re partway there, but we still have some loose ends.

When the heroine changes state, we also switch her sprite. Right now, that code is owned by the state she’s switching _from_. When she goes from ducking to standing, the ducking state sets her image:

```gdscript
func handle_input(heroine, input):
	if input == RELEASE_DOWN:
		heroine.set_graphics(IMAGE_STAND)
		return StandingState.new()
	return null
```

What we really want is each state to control its own graphics. We can handle that by giving the state an _entry action_:

```gdscript
func enter(heroine):
	heroine.set_graphics(IMAGE_STAND)
```

Back in `Heroine`, we modify the code for handling state changes to call that on the new state:

`void Heroine::handleInput(Input input) {   HeroineState* state = state_->handleInput(*this, input);   if (state != NULL)   {     delete state_;     state_ = state;      // Call the enter action on the new state.     state_->enter(*this);   } }`

This lets us simplify the ducking code to:

```gdscript
func handle_input(input):
	var new_state = state.handle_input(self, input)
	if new_state != null:
		state = new_state
		state.enter(self)
```

All it does is switch to standing and the standing state takes care of the graphics. Now our states really are encapsulated. One particularly nice thing about entry actions is that they run when you enter the state regardless of which state you’re coming _from_.

Most real-world state graphs have multiple transitions into the same state. For example, our heroine will also end up standing after she lands a jump or dive. That means we would end up duplicating some code everywhere that transition occurs. Entry actions give us a place to consolidate that.

We can, of course, also extend this to support an _exit action_. This is just a method we call on the state we’re _leaving_ right before we switch to the new state.

## [What’s the Catch?](https://www.gameprogrammingpatterns.com/state.html#what's-the-catch)

I’ve spent all this time selling you on FSMs, and now I’m going to pull the rug out from under you. Everything I’ve said so far is true, and FSMs are a good fit for some problems. But their greatest virtue is also their greatest flaw.

State machines help you untangle hairy code by enforcing a very constrained structure on it. All you’ve got is a fixed set of states, a single current state, and some hardcoded transitions.

A finite state machine isn’t even _Turing complete_. Automata theory describes computation using a series of abstract models, each more complex than the previous. A _Turing machine_ is one of the most expressive models.

“Turing complete” means a system (usually a programming language) is powerful enough to implement a Turing machine in it, which means all Turing complete languages are, in some ways, equally expressive. FSMs are not flexible enough to be in that club.

If you try using a state machine for something more complex like game AI, you will slam face-first into the limitations of that model. Thankfully, our forebears have found ways to dodge some of those barriers. I’ll close this chapter out by walking you through a couple of them.

## [Concurrent State Machines](https://www.gameprogrammingpatterns.com/state.html#concurrent-state-machines)

We’ve decided to give our heroine the ability to carry a gun. When she’s packing heat, she can still do everything she could before: run, jump, duck, etc. But she also needs to be able to fire her weapon while doing it.

If we want to stick to the confines of an FSM, we have to _double_ the number of states we have. For each existing state, we’ll need another one for doing the same thing while she’s armed: standing, standing with gun, jumping, jumping with gun, you get the idea.

Add a couple of more weapons and the number of states explodes combinatorially. Not only is it a huge number of states, it’s a huge amount of redundancy: the unarmed and armed states are almost identical except for the little bit of code to handle firing.

The problem is that we’ve jammed two pieces of state — what she’s _doing_ and what she’s _carrying_ — into a single machine. To model all possible combinations, we would need a state for each _pair_. The fix is obvious: have two separate state machines.

If we want to cram _n_ states for what she’s doing and _m_ states for what she’s carrying into a single machine, we need _n × m_ states. With two machines, it’s just _n + m_.

We keep our original state machine for what she’s doing and leave it alone. Then we define a separate state machine for what she’s carrying. `Heroine` will have _two_ “state” references, one for each, like:

`class Heroine {   // Other code...  private:   HeroineState* state_;   HeroineState* equipment_; };`

For illustrative purposes, we’re using the full State pattern for her equipment. In practice, since it only has two states, a Boolean flag would work too.

When the heroine delegates inputs to the states, she hands it to both of them:

`void Heroine::handleInput(Input input) {   state_->handleInput(*this, input);   equipment_->handleInput(*this, input); }`

A more full-featured system would probably have a way for one state machine to _consume_ an input so that the other doesn’t receive it. That would prevent both machines from erroneously trying to respond to the same input.

Each state machine can then respond to inputs, spawn behavior, and change its state independently of the other machine. When the two sets of states are mostly unrelated, this works well.

In practice, you’ll find a few cases where the states do interact. For example, maybe she can’t fire while jumping, or maybe she can’t do a dive attack if she’s armed. To handle that, in the code for one state, you’ll probably just do some crude `if` tests on the _other_ machine’s state to coordinate them. It’s not the most elegant solution, but it gets the job done.

## [Hierarchical State Machines](https://www.gameprogrammingpatterns.com/state.html#hierarchical-state-machines)

After fleshing out our heroine’s behavior some more, she’ll likely have a bunch of similar states. For example, she may have standing, walking, running, and sliding states. In any of those, pressing B jumps and pressing down ducks.

With a simple state machine implementation, we have to duplicate that code in each of those states. It would be better if we could implement that once and reuse it across all of the states.

If this was just object-oriented code instead of a state machine, one way to share code across those states would be using inheritance. We could define a class for an “on ground” state that handles jumping and ducking. Standing, walking, running, and sliding would then inherit from that and add their own additional behavior.

This has both good and bad implications. Inheritance is a powerful means of code reuse, but it’s also a very strong coupling between two chunks of code. It’s a big hammer, so swing it carefully.

It turns out, this is a common structure called a _hierarchical state machine_. A state can have a _superstate_ (making itself a _substate_). When an event comes in, if the substate doesn’t handle it, it rolls up the chain of superstates. In other words, it works just like overriding inherited methods.

In fact, if we’re using the State pattern to implement our FSM, we can use class inheritance to implement the hierarchy. Define a base class for the superstate:

`class OnGroundState : public HeroineState { public:   virtual void handleInput(Heroine& heroine, Input input)   {     if (input == PRESS_B)     {       // Jump...     }     else if (input == PRESS_DOWN)     {       // Duck...     }   } };`

And then each substate inherits it:

`class DuckingState : public OnGroundState { public:   virtual void handleInput(Heroine& heroine, Input input)   {     if (input == RELEASE_DOWN)     {       // Stand up...     }     else     {       // Didn't handle input, so walk up hierarchy.       OnGroundState::handleInput(heroine, input);     }   } };`

This isn’t the only way to implement the hierarchy, of course. If you aren’t using the Gang of Four’s State pattern, this won’t work. Instead, you can model the current state’s chain of superstates explicitly using a _stack_ of states instead of a single state in the main class.

The current state is the one on the top of the stack, under that is its immediate superstate, and then _that_ state’s superstate and so on. When you dish out some state-specific behavior, you start at the top of the stack and walk down until one of the states handles it. (If none do, you ignore it.)

## [Pushdown Automata](https://www.gameprogrammingpatterns.com/state.html#pushdown-automata)

There’s another common extension to finite state machines that also uses a stack of states. Confusingly, the stack represents something entirely different, and is used to solve a different problem.

The problem is that finite state machines have no concept of _history_. You know what state you _are_ in, but have no memory of what state you _were_ in. There’s no easy way to go back to a previous state.

Here’s an example: Earlier, we let our fearless heroine arm herself to the teeth. When she fires her gun, we need a new state that plays the firing animation and spawns the bullet and any visual effects. So we slap together a `FiringState` and make all of the states that she can fire from transition into that when the fire button is pressed.

Since this behavior is duplicated across several states, it may also be a good place to use a hierarchical state machine to reuse that code.

The tricky part is what state she transitions to _after_ firing. She can pop off a round while standing, running, jumping, and ducking. When the firing sequence is complete, she should transition back to what she was doing before.

If we’re sticking with a vanilla FSM, we’ve already forgotten what state she was in. To keep track of it, we’d have to define a slew of nearly identical states — firing while standing, firing while running, firing while jumping, and so on — just so that each one can have a hardcoded transition that goes back to the right state when it’s done.

What we’d really like is a way to _store_ the state she was in before firing and then _recall_ it later. Again, automata theory is here to help. The relevant data structure is called a [_pushdown automaton_](http://en.wikipedia.org/wiki/Pushdown_automaton).

Where a finite state machine has a _single_ pointer to a state, a pushdown automaton has a _stack_ of them. In an FSM, transitioning to a new state _replaces_ the previous one. A pushdown automaton lets you do that, but it also gives you two additional operations:

1. You can _push_ a new state onto the stack. The “current” state is always the one on top of the stack, so this transitions to the new state. But it leaves the previous state directly under it on the stack instead of discarding it.
    
2. You can _pop_ the topmost state off the stack. That state is discarded, and the state under it becomes the new current state.
    

![The stack for a pushdown automaton. First it just contains a Standing state. A Firing state is pushed on top, then popped back off when done.](https://www.gameprogrammingpatterns.com/images/state-pushdown.png)

This is just what we need for firing. We create a _single_ firing state. When the fire button is pressed while in any other state, we _push_ the firing state onto the stack. When the firing animation is done, we _pop_ that state off, and the pushdown automaton automatically transitions us right back to the state we were in before.

## [So How Useful Are They?](https://www.gameprogrammingpatterns.com/state.html#so-how-useful-are-they)

Even with those common extensions to state machines, they are still pretty limited. The trend these days in game AI is more toward exciting things like _[behavior trees](http://web.archive.org/web/20140402204854/http://www.altdevblogaday.com/2011/02/24/introduction-to-behavior-trees/)_ and _[planning systems](http://web.media.mit.edu/~jorkin/goap.html)_. If complex AI is what you’re interested in, all this chapter has done is whet your appetite. You’ll want to read other books to satisfy it.

This doesn’t mean finite state machines, pushdown automata, and other simple systems aren’t useful. They’re a good modeling tool for certain kinds of problems. Finite state machines are useful when:

- You have an entity whose behavior changes based on some internal state.
    
- That state can be rigidly divided into one of a relatively small number of distinct options.
    
- The entity responds to a series of inputs or events over time.
    

In games, they are most known for being used in AI, but they are also common in implementations of user input handling, navigating menu screens, parsing text, network protocols, and other asynchronous behavior.
