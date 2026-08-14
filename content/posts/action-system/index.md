---
title: "Input buffered action system"
summary: "A system that manages actions and input buffers."
categories: ["Post","Blog",]
tags: ["TGA", "Gameplay"]
#externalUrl: ""
#showSummary: true
date: 2026-03-04
draft: false

studio: There goes the goose
role: Gameplay programmer
---

## Context

For the next-to-last game project I was in charge of implementing the player movement and actions.
During this, I tried my hand at implementing a buffered action system. 
We had a couple of actions that had to be executed without interrupting each other, 
but I also wanted them to be buffered in order to make the game handle better.
The purpose of an input buffer is that if the player presses the trigger key the moment before it's ready, 
that input should still be buffered and the action executed afterward.
This is in order to align with player intent. 
It's important to note that in most cases the buffer time 
can't be more than a fraction of a second since the situation might've changed enough 
since the input was given for the player to have changed their intention.
For example, there might now be an incoming attack towards the player, 
and they would rather block than have the attack input they triggered some time before to be used.
The human reaction time of ~150-250ms is a good starting position, 
since the intent is unlikely to have changed due to a response to something that occured in the game during that time.

## Execution

I first defined an action struct with some barebones data I knew would be needed.
Then I began working on the system for evaluating the actions, 
setting up a basic loop that read inputs and updated the actions according to
their parameters. 

Using basic flow control in the shape of if-statements and loops was a choice I made at this point. 
My reasoning for it was that it would be easy to modify and alter as I went along.

I kept progressively adding data to the actions and more complex evaluation of the actions 
as the gameplay features were refined and edge-cases were found. 
At one point towards the end of the project I did a slight overhaul in order to
solve a flow-related bug and clean up some messy portions.

## Breakdown

### The action stuct

Actions have a few things going on, but primarily it boils down to 3 major aspects. 
The callbacks that execute at different points of the action, 
the parameters for adjusting how the action should behave 
and the timers for keeping track of the current state of the action.

```cpp {classes="PlayerAction Player ActionCallback Trigger ActionState Actions ButtonMappingIndex" enum_values="OnPressed OnReleased INVALID_BUTTON Inactive Count"}

struct PlayerAction
{
const char* debugName = "";

// Callbacks
// Params: PlayerAction&, const PlayerState&
// Return: bool

ActionCallback onBegin = nullptr;
ActionCallback onCharge = nullptr;
ActionCallback onTrigger = nullptr;
ActionCallback onUpdate = nullptr;
ActionCallback onEnd = nullptr;
ActionCallback onInterrupt = nullptr;

// Parameters

array<bool, Actions::Count> isBlockedBy{false};

Trigger trigger = Trigger::OnPressed;
ButtonMappingIndex key = INVALID_BUTTON;

ActionState state = ActionState::Inactive;

// Time the action should be buffered
float bufferDuration = 0.15f;

// Time between uses of the action
float cooldown = 0.f;

// The time the action takes to finish
float duration = 0.f;

// Timers
float currentCooldown = 0.f;

float currentDuration = 0.f;

float currentChargeTime = 0.f;

float timeSinceButtonPressed = 0.f;
float timeSinceButtonReleased = 0.f;
bool buttonHeld = false;
};
```

I opted for using an array in front of a map to store what actions the action blocks,
in order to completely avoid heap allocations.
It is also faster to look up results and can be compressed to a bitset. 
Only downside is that the memory usage scales with the amount of actions available to the player, 
but that is unlikely to have a noticeable effect on any project scale.

### Evaluating the actions

The way I evaluate it is by letting it trickle down a tree of states 
based on the trigger, current state and the other run-time values. 
The callbacks are called at state changes or during states and 
are given information about the player and the action.

I am going to break down each part individually, 
partly to better describe it in detail and partly to make it more readable.

This is what the update function for the action system looks like. 
The next parts will focus on the "..." portion and omit the loop. 
I will also omit all "action." accesses, as well as const qualifiers in order to make better use of the line width.


```cpp {classes="StateInfo"}
void Update(StateInfo& aState, float aDeltaTime)
{
  for (size_t i = 0; i < Actions::Count; i++)
  {
    auto& action     = aState.player.actions[i];
    auto actionType  = static_cast<Actions>(i);
    char* actionName = action.debugName;
    
    ...
  }
}
```
Note that the actions can be prioritized by sorting the Actions enum, 
changing the order in which they are evaluated. 
It's not very dynamic, and if we used scripting for gameplay it wouldn't be an option,
but it worked well for this project.

First off, I check if any action has been reset. 
This is done at the start of the loop primarily because it's easier to 
keep track of the resets when they all occur in one place.

```cpp {enum_values="Ended Interrupted Blocked"}
if (state == ActionState::Ended)
{
  if (onEnd != nullptr)
  {
    onEnd(action,aState);
  }

  buttonHeld = false;
  timeSinceButtonPressed = 0;
  timeSinceButtonReleased = 0;
  currentCooldown = cooldown;
  currentDuration = duration;
  state = ActionState::Inactive;
}

if (state == ActionState::Interrupted)
{
  if (onInterrupt != nullptr)
  {
    onInterrupt(action, aState);
  }

  buttonHeld = false;
  timeSinceButtonPressed = 0;
  timeSinceButtonReleased = 0;
  currentCooldown = cooldown;
  currentDuration = duration;
  state = ActionState::Inactive;
}

if (state == ActionState::Blocked)
{
  state = ActionState::Inactive;
}
```

The timers that are running continuously are updated. In hindsight, 
since we use double-buffering which delays the input by one frame,
those timers should be updated after the input is processed.

```cpp
currentCooldown -= aDeltaTime;
timeSinceButtonPressed += aDeltaTime;
timeSinceButtonReleased += aDeltaTime;
```

Now input is evaluated. I use separate timers for pressed and release since they can both trigger the action independently.
I also use them to check if the button should be considered held or not, 
in order to be able to "release" the button for chaining actions that require held input.
This has to be done before the block/interrupt stage so that it always ends up registering in the input buffer.

```cpp
// Input
{
  if (InputMapper::IsButtonPressed(key))
  {
    LOG_DEBUG("Action: %s pressed", actionName);
    timeSinceButtonPressed = 0;
  }
  
  if (InputMapper::IsButtonReleased(key))
  {
    LOG_DEBUG("Action: %s released", actionName);
    timeSinceButtonReleased = 0;
  }
  buttonHeld = timeSinceButtonPressed < timeSinceButtonReleased;
}


bool actionPressed   = timeSinceButtonPressed  < bufferDuration;
bool actionReleased  = timeSinceButtonReleased < bufferDuration;
bool pressActivate   = actionPressed  && trigger == Trigger::OnPressed;
bool releaseActivate = actionReleased && trigger == Trigger::OnReleased;
bool wantToActivate  = pressActivate || releaseActivate;

if (currentCooldown > 0.f)
{
  continue;
}
```

Then the blockers are evaluated.
At this stage it’s also possible for an ongoing 
action to be interrupted, or animation-canceled, by an action with higher priority.

```cpp {enum_values="Ongoing"}
bool blocked = false;
const char* blockerName = nullptr;

for (size_t i = 0; i < Actions::Count; i++)
{
  if (isBlockedBy[i])
  {
    auto& blockingAction = aState.player.actions[i];
    
    if (blockingstate == ActionState::Ongoing || blockingstate == ActionState::Charging)
    {
      blockerName = blockingAction.name;
      blocked = true;
      break;
    }
  }
}

if (blocked)
{
  if (state == ActionState::Ongoing)
  {					
    LOG_DEBUG("Action: %s was interrupted by %s", actionName, blockerName);
    state = ActionState::Interrupted;
  }
  else
  {
    if (wantToActivate)
    {
      LOG_DEBUG("Action: %s was blocked by %s", actionName, blockerName);
    }
    state = ActionState::Blocked;
  }
    
  continue;
}
```

If the action passes, I check if the registered input corresponds to activating 
the action and resolving how the action should be activated based on the trigger. 
The simplest is triggering it immediately when the associated button is pressed.

```cpp {enum_values="Charging"}
// To begin charging
if (state != ActionState::Ongoing)
{
  if (state < ActionState::Charging && buttonHeld)
  {
    LOG_DEBUG("Action: %s has begun", actionName);

    currentChargeTime = 0.f;

    if (onBegin != nullptr)
    {
      onBegin(action,aState);
    }

    state = ActionState::Charging;
  }
  
  if (trigger == Trigger::OnPressed && actionPressed)
  {
    bool success = onTrigger == nullptr;
    
    if (onTrigger != nullptr)
    {
      success |= onTrigger(action, aState);
      
      if (!success)
      {
        LOG_DEBUG("Action: %s failed", actionName);
      }
    }
      
    if (success)
    {
      LOG_DEBUG("Action: %s triggered", actionName);
      state = ActionState::Ongoing;
      currentDuration = duration;
    }
  }
  
  if (trigger == Trigger::OnReleased && actionReleased)
  {
    bool success = onTrigger == nullptr;
    
    if (onTrigger != nullptr)
    {
      success |= onTrigger(action, aState);
      
      if (!success)
      {
        LOG_DEBUG("Action: %s failed", actionName);
      }
    }
        
    if (success)
    {
      LOG_DEBUG("Action: %s triggered", actionName);
      state = ActionState::Ongoing;
      currentDuration = duration;
    }
  }
}
```

In any other case, it goes into a “charge” state, which just indicates that the button is being held. 
The “on charge” callback executes at this point and can use this info to decide if they should trigger or not.

```cpp {enum_values="OnHeld"}

// During charging
if (state == ActionState::Charging)
{	
  bool attemptTrigger = false;
  bool isExpired = false;
  
  switch (trigger)
  {
    case Trigger::OnPressed:
    {
      attemptTrigger = 
        timeSinceButtonPressed < bufferDuration || buttonHeld;
      isExpired = !attemptTrigger;
      break;
    }
    case Trigger::OnReleased:
    {
      attemptTrigger = 
        timeSinceButtonReleased < bufferDuration;
        
      if (!buttonHeld)
      {
        isExpired = !attemptTrigger;
      }
      break;
    }
    case Trigger::OnHeld:
    {
      attemptTrigger = buttonHeld;
      isExpired = !attemptTrigger;
      break;
    }
  }
    
}
```
When released, the trigger callback is run if the corresponding trigger type is set.
```cpp

  if (attemptTrigger)
  {
    bool success = onTrigger == nullptr;
    
    if (onTrigger != nullptr)
    {
      success |= onTrigger(action, aState);
      
      if (!success)
      {
        LOG_DEBUG("Action: %s failed", actionName);
      }
    }
    
    if (success)
    {
      LOG_DEBUG("Action: %s triggered", actionName);
      state = ActionState::Ongoing;
      currentDuration = duration;
    }
  }
  else if (isExpired)
  {
    LOG_DEBUG("Action: %s expired", actionName);
    state = ActionState::Ended;
  }
  else
  {
    LOG_DEBUG("Action: %s is charging", actionName);
    currentChargeTime += aDeltaTime;
    
    if (onCharge != nullptr)
    {
      onCharge(action, aState);
    }
  }
```

If the action triggers, the state is set to ongoing and the update callback is run for the duration.

```cpp
// While the action is happening
if (state == ActionState::Ongoing)
{
  LOG_DEBUG("Action: %s is ongoing", actionName);
  
  if (onUpdate != nullptr)
  {
    onUpdate(action, aState);
  }
  
  if (currentDuration <= 0.f)
  {
    LOG_DEBUG("Action: %s ended", actionName);
    state = ActionState::Ended;
  }
  else
  {
    currentDuration -= aDeltaTime;
  }
}
```

Each state is evaluated at once, meaning there's no delay for triggering the callbacks since multiple of them can happen in the same frame.

Info/data
Stages
Callbacks


## Evaluation

If I were to remake it, I would change a lot of things. 

optimize callbacks (check which ones are really needed etc)
make callbacks return an enum for clarity and extended flow control

Add multiple keys to the actions for combination triggers (such as ctrl+s).

add one synch frame before an action interrupting so that the playing action has time to stop first.

add a way to consume inputs, potentially by copying them from the input manager and then pulling them from the internal buffer which can be cleared.

the start and charging evaluations are a bit messy and convoluted, 
so I would like to find a good way to refactor the flow to something easier to maintain and expand on.
