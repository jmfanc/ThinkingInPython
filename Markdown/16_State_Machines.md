State Machines
==============

While *State* has a way to allow the client programmer to change the
implementation, *StateMachine* imposes a structure to automatically
change the implementation from one object to the next. The current
implementation represents the state that a system is in, and the system
behaves differently from one state to the next (because it uses
*State*). Basically, this is a "state machine" using objects.

The code that moves the system from one state to the next is often a
*Template Method*, as seen in the following framework for a basic state
machine.

Each state can be `run()` to perform its behavior, and (in this
design) you can also pass it an "input" object so it can tell you what
new state to move to based on that "input". The key distinction between
this design and the next is that here, each `State` object decides
what other states it can move to, based on the "input", whereas in the
subsequent design all of the state transitions are held in a single
table. Another way to put it is that here, each `State` object has its
own little `State` table, and in the subsequent design there is a
single master state transition table for the whole system:

```python
# StateMachine/State.py
# A State has an operation, and can be moved
# into the next State given an Input:

class State:
    def run(self):
        assert 0, "run not implemented"
    def next(self, input):
        assert 0, "next not implemented"
```

This class is clearly unnecessary, but it allows us to say that
something is a `State` object in code, and provide a slightly
different error message when all the methods are not implemented. We
could have gotten basically the same effect by saying:

    class State: pass

because we would still get exceptions if `run()` or `next()` were
called for a derived type, and they hadn't been implemented.

The `StateMachine` keeps track of the current state, which is
initialized by the constructor. The `runAll()` method takes a list of
`Input` objects. This method not only moves to the next state, but it
also calls `run()` for each state object---thus you can see it's an
expansion of the idea of the `State` pattern, since `run()` does
something different depending on the state that the system is in:

```python
# StateMachine/StateMachine.py
# Takes a list of Inputs to move from State to
# State using a template method.

class StateMachine:
    def __init__(self, initialState):
        self.currentState = initialState
        self.currentState.run()
    # Template method:
    def runAll(self, inputs):
        for i in inputs:
            print(i)
            self.currentState = self.currentState.next(i)
            self.currentState.run()
```

I've also treated `runAll()` as a template method. This is typical,
but certainly not required---you could concievably want to override it,
but typically the behavior change will occur in `State`'s `run()`
instead.

At this point the basic framework for this style of *StateMachine* (where each
state decides the next states) is complete. As an example, I'll use a fancy
mousetrap that can move through several states in the process of trapping a
mouse^[No mice were harmed in the creation of this example.]. The mouse
classes and information are stored in the `mouse` package, including a class
representing all the possible moves that a mouse can make, which will be the
inputs to the state machine:

```python
# StateMachine/mouse/MouseAction.py

class MouseAction:
    def __init__(self, action):
        self.action = action
    def __str__(self): return self.action
    def __cmp__(self, other):
        return cmp(self.action, other.action)
    # Necessary when __cmp__ or __eq__ is defined
    # in order to make this class usable as a
    # dictionary key:
    def __hash__(self):
        return hash(self.action)

# Static fields; an enumeration of instances:
MouseAction.appears = MouseAction("mouse appears")
MouseAction.runsAway = MouseAction("mouse runs away")
MouseAction.enters = MouseAction("mouse enters trap")
MouseAction.escapes = MouseAction("mouse escapes")
MouseAction.trapped = MouseAction("mouse trapped")
MouseAction.removed = MouseAction("mouse removed")
```

You'll note that `__cmp__()` has been overidden to implement a
comparison between `action` values. Also, each possible move by a
mouse is enumerated as a `MouseAction` object, all of which are static
fields in `MouseAction`.

For creating test code, a sequence of mouse inputs is provided from a
text file:

```python
# StateMachine/mouse/MouseMoves.txt
mouse appears
mouse runs away
mouse appears
mouse enters trap
mouse escapes
mouse appears
mouse enters trap
mouse trapped
mouse removed
mouse appears
mouse runs away
mouse appears
mouse enters trap
mouse trapped
mouse removed
```

With these tools in place, it's now possible to create the first version
of the mousetrap program. Each `State` subclass defines its `run()`
behavior, and also establishes its next state with an `if-else`
clause:

```python
# StateMachine/mousetrap1/MouseTrapTest.py
# State Machine pattern using 'if' statements
# to determine the next state.
import string, sys
sys.path += ['../stateMachine', '../mouse']
from State import State
from StateMachine import StateMachine
from MouseAction import MouseAction
# A different subclass for each state:

class Waiting(State):
    def run(self):
        print("Waiting: Broadcasting cheese smell")

    def next(self, input):
        if input == MouseAction.appears:
            return MouseTrap.luring
        return MouseTrap.waiting

class Luring(State):
    def run(self):
        print("Luring: Presenting Cheese, door open")

    def next(self, input):
        if input == MouseAction.runsAway:
            return MouseTrap.waiting
        if input == MouseAction.enters:
            return MouseTrap.trapping
        return MouseTrap.luring

class Trapping(State):
    def run(self):
        print("Trapping: Closing door")

    def next(self, input):
        if input == MouseAction.escapes:
            return MouseTrap.waiting
        if input == MouseAction.trapped:
            return MouseTrap.holding
        return MouseTrap.trapping

class Holding(State):
    def run(self):
        print("Holding: Mouse caught")

    def next(self, input):
        if input == MouseAction.removed:
            return MouseTrap.waiting
        return MouseTrap.holding

class MouseTrap(StateMachine):
    def __init__(self):
        # Initial state
        StateMachine.__init__(self, MouseTrap.waiting)

# Static variable initialization:
MouseTrap.waiting = Waiting()
MouseTrap.luring = Luring()
MouseTrap.trapping = Trapping()
MouseTrap.holding = Holding()

moves = map(string.strip,
  open("../mouse/MouseMoves.txt").readlines())
MouseTrap().runAll(map(MouseAction, moves))
```

The `StateMachine` class simply defines all the possible states as
static objects, and also sets up the initial state. The `UnitTest`
creates a `MouseTrap` and then tests it with all the inputs from a
`MouseMoveList`.

While the use of `if` statements inside the `next()` methods is
perfectly reasonable, managing a large number of these could become
difficult. Another approach is to create tables inside each `State`
object defining the various next states based on the input.

Initially, this seems like it ought to be quite simple. You should be
able to define a static table in each `State` subclass that defines
the transitions in terms of the other `State` objects. However, it
turns out that this approach generates cyclic initialization
dependencies. To solve the problem, I've had to delay the initialization
of the tables until the first time that the `next()` method is called
for a particular `State` object. Initially, the `next()` methods
can appear a little strange because of this.

The `StateT` class is an implementation of `State` (so that the same
`StateMachine` class can be used from the previous example) that adds
a `Map` and a method to initialize the map from a two-dimensional
array. The `next()` method has a base-class implementation which must
be called from the overridden derived class `next()` methods after
they test for a `null Map` (and initialize it if it's `null`):

```python
# StateMachine/mousetrap2/MouseTrap2Test.py
# A better mousetrap using tables
import string, sys
sys.path += ['../stateMachine', '../mouse']
from State import State
from StateMachine import StateMachine
from MouseAction import MouseAction

class StateT(State):
    def __init__(self):
        self.transitions = None
    def next(self, input):
        if self.transitions.has_key(input):
            return self.transitions[input]
        else:
            raise "Input not supported for current state"

class Waiting(StateT):
    def run(self):
        print("Waiting: Broadcasting cheese smell")
    def next(self, input):
        # Lazy initialization:
        if not self.transitions:
            self.transitions = {
              MouseAction.appears : MouseTrap.luring
            }
        return StateT.next(self, input)

class Luring(StateT):
    def run(self):
        print("Luring: Presenting Cheese, door open")
    def next(self, input):
        # Lazy initialization:
        if not self.transitions:
            self.transitions = {
              MouseAction.enters : MouseTrap.trapping,
              MouseAction.runsAway : MouseTrap.waiting
            }
        return StateT.next(self, input)

class Trapping(StateT):
    def run(self):
        print("Trapping: Closing door")
    def next(self, input):
        # Lazy initialization:
        if not self.transitions:
            self.transitions = {
              MouseAction.escapes : MouseTrap.waiting,
              MouseAction.trapped : MouseTrap.holding
            }
        return StateT.next(self, input)

class Holding(StateT):
    def run(self):
        print("Holding: Mouse caught")
    def next(self, input):
        # Lazy initialization:
        if not self.transitions:
            self.transitions = {
              MouseAction.removed : MouseTrap.waiting
            }
        return StateT.next(self, input)

class MouseTrap(StateMachine):
    def __init__(self):
        # Initial state
        StateMachine.__init__(self, MouseTrap.waiting)

# Static variable initialization:
MouseTrap.waiting = Waiting()
MouseTrap.luring = Luring()
MouseTrap.trapping = Trapping()
MouseTrap.holding = Holding()

moves = map(string.strip,
  open("../mouse/MouseMoves.txt").readlines())
mouseMoves = map(MouseAction, moves)
MouseTrap().runAll(mouseMoves)
```

The rest of the code is identical---the difference is in the `next()`
methods and the `StateT` class.

If you have to create and maintain a lot of `State` classes, this
approach is an improvement, since it's easier to quickly read and
understand the state transitions from looking at the table.

Table-Driven State Machine
--------------------------

The advantage of the previous design is that all the information about a
state, including the state transition information, is located within the
state class itself. This is generally a good design principle.

However, in a pure state machine, the machine can be completely
represented by a single state-transition table. This has the advantage
of locating all the information about the state machine in a single
place, which means that you can more easily create and maintain the
table based on a classic state-transition diagram.

The classic state-transition diagram uses a circle to represent each
state, and lines from the state pointing to all states that state can
transition into. Each transition line is annotated with conditions for
transition and an action during transition. Here's what it looks like:

(Simple State Machine Diagram)

Goals:

-   Direct translation of state diagram
-   Vector of change: the state diagram representation
-   Reasonable implementation
-   No excess of states (you could represent every single change with a
    new state)
-   Simplicity and flexibility

Observations:

-   States are trivial---no information or functions/data, just an
    identity
-   Not like the State pattern!
-   The machine governs the move from state to state
-   Similar to flyweight
-   Each state may move to many others
-   Condition & action functions must also be external to states
-   Centralize description in a single table containing all variations,
    for ease of configuration

Example:

-   State Machine & Table-Driven Code
-   Implements a vending machine
-   Uses several other patterns
-   Separates common state-machine code from specific application (like
    template method)
-   Each input causes a seek for appropriate solution (like chain of
    responsibility)
-   Tests and transitions are encapsulated in function objects (objects
    that hold functions)
-   Java constraint: methods are not first-class objects

![image description](_images/stateMachine)

### The State Class

The `State` class is distinctly different from before, since it is
really just a placeholder with a name. Thus it is not inherited from
previous `State` classes:

```python
# StateMachine/stateMachine2/State.py

class State:
    def __init__(self, name): self.name = name
    def __str__(self): return self.name
```

### Conditions for Transition

In the state transition diagram, an input is tested to see if it meets
the condition necessary to transfer to the state under question. As
before, the `Input` is just a tagging interface:

```python
# StateMachine/stateMachine2/Input.py
# Inputs to a state machine

class Input: pass
```

The `Condition` evaluates the `Input` to decide whether this row in
the table is the correct transition:

```python
# StateMachine/stateMachine2/Condition.py
# Condition function object for state machine

class Condition:
    boolean condition(input) :
        assert 0, "condition() not implemented"
```

### Transition Actions

If the `Condition` returns `true`, then the transition to a new
state is made, and as that transition is made some kind of action occurs
(in the previous state machine design, this was the `run()` method):

```python
# StateMachine/stateMachine2/Transition.py
# Transition function object for state machine

class Transition:
    def transition(self, input):
        assert 0, "transition() not implemented"
```

### The Table

With these classes in place, we can set up a 3-dimensional table where
each row completely describes a state. The first element in the row is
the current state, and the rest of the elements are each a row
indicating what the *type* of the input can be, the condition that must
be satisfied in order for this state change to be the correct one, the
action that happens during transition, and the new state to move into.
Note that the `Input` object is not just used for its type, it is also
a *Messenger* object that carries information to the `Condition` and
`Transition` objects:

    {(CurrentState, InputA) : (ConditionA, TransitionA, NextA),
     (CurrentState, InputB) : (ConditionB, TransitionB, NextB),
     (CurrentState, InputC) : (ConditionC, TransitionC, NextC),
     ...
    }

### The Basic Machine

Here's the basic machine, (code only roughly converted):

```python
# StateMachine/stateMachine2/StateMachine.py
# A table-driven state machine

class StateMachine:
    def __init__(self, initialState, tranTable):
        self.state = initialState
        self.transitionTable = tranTable

    def nextState(self, input):

        Iterator it=((List)map.get(state)).iterator()
        while(it.hasNext()):
            Object[] tran = (Object[])it.next()
            if(input == tran[0] ||
               input.getClass() == tran[0]):
                if(tran[1] != null):
                    Condition c = (Condition)tran[1]
                    if(!c.condition(input))
                        continue # Failed test

                if(tran[2] != null)
                    ((Transition)tran[2]).transition(input)
                state = (State)tran[3]
                return


        throw RuntimeException(
          "Input not supported for current state")
```

### Simple Vending Machine

Here's the simple vending machine, (code only roughly converted):

```python
# StateMachine/vendingmachine/VendingMachine.py
# Demonstrates use of StateMachine.py
import sys
sys.path += ['../stateMachine2']
import StateMachine

class State:
    def __init__(self, name): self.name = name
    def __str__(self): return self.name

State.quiescent = State("Quiesecent")
State.collecting = State("Collecting")
State.selecting = State("Selecting")
State.unavailable = State("Unavailable")
State.wantMore = State("Want More?")
State.noChange = State("Use Exact Change Only")
State.makesChange = State("Machine makes change")

class HasChange:
    def __init__(self, name): self.name = name
    def __str__(self): return self.name

HasChange.yes = HasChange("Has change")
HasChange.no = HasChange("Cannot make change")

class ChangeAvailable(StateMachine):
    def __init__(self):
        StateMachine.__init__(State.makesChange, {
          # Current state, input
          (State.makesChange, HasChange.no) :
            # test, transition, next state:
            (null, null, State.noChange),
          (State.noChange, HasChange.yes) :
            (null, null, State.noChange)
        })

class Money:
    def __init__(self, name, value):
        self.name = name
        self.value = value
    def __str__(self): return self.name
    def getValue(self): return self.value

Money.quarter = Money("Quarter", 25)
Money.dollar = Money("Dollar", 100)

class Quit:
    def __str__(self): return "Quit"

Quit.quit = Quit()

class Digit:
    def __init__(self, name, value):
        self.name = name
        self.value = value
    def __str__(self): return self.name
    def getValue(self): return self.value

class FirstDigit(Digit): pass
FirstDigit.A = FirstDigit("A", 0)
FirstDigit.B = FirstDigit("B", 1)
FirstDigit.C = FirstDigit("C", 2)
FirstDigit.D = FirstDigit("D", 3)

class SecondDigit(Digit): pass
SecondDigit.one = SecondDigit("one", 0)
SecondDigit.two = SecondDigit("two", 1)
SecondDigit.three = SecondDigit("three", 2)
SecondDigit.four = SecondDigit("four", 3)

class ItemSlot:
    id = 0
    def __init__(self, price, quantity):
        self.price = price
        self.quantity = quantity
    def __str__(self): return `ItemSlot.id`
    def getPrice(self): return self.price
    def getQuantity(self): return self.quantity
    def decrQuantity(self): self.quantity -= 1

class VendingMachine(StateMachine):
    changeAvailable = ChangeAvailable()
    amount = 0
    FirstDigit first = null
    ItemSlot[][] items = ItemSlot[4][4]

    # Conditions:
    def notEnough(self, input):
        i1 = first.getValue()
        i2 = input.getValue()
        return items[i1][i2].getPrice() > amount

    def itemAvailable(self, input):
        i1 = first.getValue()
        i2 = input.getValue()
        return items[i1][i2].getQuantity() > 0

    def itemNotAvailable(self, input):
        return !itemAvailable.condition(input)
        #i1 = first.getValue()
        #i2 = input.getValue()
        #return items[i1][i2].getQuantity() == 0

    # Transitions:
    def clearSelection(self, input):
        i1 = first.getValue()
        i2 = input.getValue()
        ItemSlot is = items[i1][i2]
        print (
          "Clearing selection: item " + is +
          " costs " + is.getPrice() +
          " and has quantity " + is.getQuantity())
        first = null

    def dispense(self, input):
        i1 = first.getValue()
        i2 = input.getValue()
        ItemSlot is = items[i1][i2]
        print(("Dispensing item " +
          is + " costs " + is.getPrice() +
          " and has quantity " + is.getQuantity()))
        items[i1][i2].decrQuantity()
        print ("Quantity " +
          is.getQuantity())
        amount -= is.getPrice()
        print("Amount remaining " +
          amount)

    def showTotal(self, input):
        amount += ((Money)input).getValue()
        print("Total amount = " + amount)

    def returnChange(self, input):
        print("Returning " + amount)
        amount = 0

    def showDigit(self, input):
        first = (FirstDigit)input
        print("First Digit= "+ first)

    def __init__(self):
        StateMachine.__init__(self, State.quiescent)
        for(int i = 0 i < items.length i++)
            for(int j = 0 j < items[i].length j++)
                items[i][j] = ItemSlot((j+1)*25, 5)
        items[3][0] = ItemSlot(25, 0)
        """
        buildTable(Object[][][]{
         State.quiescent, # Current state
            # Input, test, transition, next state:
            Money.class, null,
            showTotal, State.collecting,
         State.collecting, # Current state
            # Input, test, transition, next state:
            Quit.quit, null,
            returnChange, State.quiescent,
            Money.class, null,
            showTotal, State.collecting,
            FirstDigit.class, null,
            showDigit, State.selecting,
         State.selecting, # Current state
            # Input, test, transition, next state:
            Quit.quit, null,
            returnChange, State.quiescent,
            SecondDigit.class, notEnough,
            clearSelection, State.collecting,
            SecondDigit.class, itemNotAvailable,
            clearSelection, State.unavailable,
            SecondDigit.class, itemAvailable,
            dispense, State.wantMore,
         State.unavailable, # Current state
            # Input, test, transition, next state:
            Quit.quit, null,
            returnChange, State.quiescent,
            FirstDigit.class, null,
            showDigit, State.selecting,
         State.wantMore, # Current state
            # Input, test, transition, next state:
            Quit.quit, null,
            returnChange, State.quiescent,
            FirstDigit.class, null,
            showDigit, State.selecting,
        )
        """
```

### Testing the Machine

Here's a test of the machine, (code only roughly converted):

```python
# StateMachine/vendingmachine/VendingMachineTest.py
# Demonstrates use of StateMachine.py

vm = VendingMachine()
for input in [
    Money.quarter,
    Money.quarter,
    Money.dollar,
    FirstDigit.A,
    SecondDigit.two,
    FirstDigit.A,
    SecondDigit.two,
    FirstDigit.C,
    SecondDigit.three,
    FirstDigit.D,
    SecondDigit.one,
    Quit.quit]:
    vm.nextState(input)
```

Tools
-----

Another approach, as your state machine gets bigger, is to use an
automation tool whereby you configure a table and let the tool generate
the state machine code for you. This can be created yourself using a
language like Python, but there are also free, open-source tools such as
*Libero*, at <http://www.imatix.com>.

Exercises
---------

1.  Create an example of the "virtual proxy."
2.  Create an example of the "Smart reference" proxy where you keep
    count of the number of method calls to a particular object.
3.  Create a program similar to certain DBMS systems that only allow a
    certain number of connections at any time. To implement this, use a
    singleton-like system that controls the number of "connection"
    objects that it creates. When a user is finished with a connection,
    the system must be informed so that it can check that connection
    back in to be reused. To guarantee this, provide a proxy object
    instead of a reference to the actual connection, and design the
    proxy so that it will cause the connection to be released back to
    the system.
4.  Using the *State*, make a class called `UnpredictablePerson` which
    changes the kind of response to its `hello()` method depending on
    what kind of `Mood` it's in. Add an additional kind of `Mood`
    called `Prozac`.
5.  Create a simple copy-on write implementation.
6.  Apply `TransitionTable.py` to the "Washer" problem.
7.  Create a *StateMachine* system whereby the current state along with
    input information determines the next state that the system will be
    in. To do this, each state must store a reference back to the proxy
    object (the state controller) so that it can request the state
    change. Use a `HashMap` to create a table of states, where the key
    is a `String` naming the new state and the value is the new state
    object. Inside each state subclass override a method `nextState(
    )` that has its own state-transition table. The input to
    `nextState()` should be a single word that comes from a text file
    containing one word per line.
8.  Modify the previous exercise so that the state machine can be
    configured by creating/modifying a single multi-dimensional array.
9.  Modify the "mood" exercise from the previous session so that it
    becomes a state machine using StateMachine.py
10. Create an elevator state machine system using StateMachine.py
11. Create a heating/air-conditioning system using StateMachine.py
12. A *generator* is an object that produces other objects, just like a
    factory, except that the generator function doesn't require any
    arguments. Create a `MouseMoveGenerator` which produces correct
    `MouseMove` actions as outputs each time the generator function is
    called (that is, the mouse must move in the proper sequence, thus
    the possible moves are based on the previous move---it's another
    state machine). Add a method to produce an iterator, but this method
    should take an `int` argument that specifies the number of moves
    to produce before `hasNext()` returns `false`.

Both *Proxy* and *State* provide a surrogate class that you use in your
code; the real class that does the work is hidden behind this surrogate
class. When you call a method in the surrogate, it simply turns around
and calls the method in the implementing class. These two patterns are
so similar that the *Proxy* is simply a special case of *State*. One is
tempted to just lump the two together into a pattern called *Surrogate*,
but the term "proxy" has a long-standing and specialized meaning, which
probably explains the reason for the two different patterns.

The basic idea is simple: from a base class, the surrogate is derived
along with the class or classes that provide the actual implementation:

![image description](_images/surrogate)

When a surrogate object is created, it is given an implementation to
which to send all of the method calls.

Structurally, the difference between *Proxy* and *State* is simple: a
*Proxy* has only one implementation, while *State* has more than one.
The application of the patterns is considered (in *Design Patterns*) to
be distinct: *Proxy* is used to control access to its implementation,
while *State* allows you to change the implementation dynamically.
However, if you expand your notion of "controlling access to
implementation" then the two fit neatly together.

Proxy
-----

If we implement *Proxy* by following the above diagram, it looks like
this:

```python
# Fronting/ProxyDemo.py
# Simple demonstration of the Proxy pattern.

class Implementation:
    def f(self):
        print("Implementation.f()")
    def g(self):
        print("Implementation.g()")
    def h(self):
        print("Implementation.h()")

class Proxy:
    def __init__(self):
        self.__implementation = Implementation()
    # Pass method calls to the implementation:
    def f(self): self.__implementation.f()
    def g(self): self.__implementation.g()
    def h(self): self.__implementation.h()

p = Proxy()
p.f(); p.g(); p.h()
```

It isn't necessary that `Implementation` have the same interface as
`Proxy`; as long as `Proxy` is somehow "speaking for" the class that
it is referring method calls to then the basic idea is satisfied (note
that this statement is at odds with the definition for Proxy in GoF).
However, it is convenient to have a common interface so that
`Implementation` is forced to fulfill all the methods that `Proxy`
needs to call.

Of course, in Python we have a delegation mechanism built in, so it
makes the `Proxy` even simpler to implement:

```python
# Fronting/ProxyDemo2.py
# Simple demonstration of the Proxy pattern.

class Implementation2:
    def f(self):
        print("Implementation.f()")
    def g(self):
        print("Implementation.g()")
    def h(self):
        print("Implementation.h()")

class Proxy2:
    def __init__(self):
        self.__implementation = Implementation2()
    def __getattr__(self, name):
        return getattr(self.__implementation, name)

p = Proxy2()
p.f(); p.g(); p.h();
```

The beauty of using `__getattr__()` is that `Proxy2` is
completely generic, and not tied to any particular implementation (in
Java, a rather complicated "dynamic proxy" has been invented to
accomplish this same thing).

State
-----

The *State* pattern adds more implementations to *Proxy*, along with a
way to switch from one implementation to another during the lifetime of
the surrogate:

```python
# Fronting/StateDemo.py
# Simple demonstration of the State pattern.

class State_d:
    def __init__(self, imp):
        self.__implementation = imp
    def changeImp(self, newImp):
        self.__implementation = newImp
    # Delegate calls to the implementation:
    def __getattr__(self, name):
        return getattr(self.__implementation, name)

class Implementation1:
    def f(self):
        print("Fiddle de dum, Fiddle de dee,")
    def g(self):
        print("Eric the half a bee.")
    def h(self):
        print("Ho ho ho, tee hee hee,")

class Implementation2:
    def f(self):
        print("We're Knights of the Round Table.")
    def g(self):
        print("We dance whene'er we're able.")
    def h(self):
        print("We do routines and chorus scenes")

def run(b):
    b.f()
    b.g()
    b.h()
    b.g()

b = State_d(Implementation1())
run(b)
b.changeImp(Implementation2())
run(b)
```

You can see that the first implementation is used for a bit, then the
second implementation is swapped in and that is used.

The difference between *Proxy* and *State* is in the problems that are
solved. The common uses for *Proxy* as described in *Design Patterns*
are:

1.  `Remote proxy`. This proxies for an object in a different address
    space. A remote proxy is created for you automatically by the RMI
    compiler `rmic` as it creates stubs and skeletons.
2.  `Virtual proxy`. This provides "lazy initialization" to create
    expensive objects on demand.
3.  `Protection proxy`. Used when you don't want the client programmer
    to have full access to the proxied object.
4.  `Smart reference`. To add additional actions when the proxied
    object is accessed. For example, or to keep track of the number of
    references that are held for a particular object, in order to
    implement the *copy-on-write* idiom and prevent object aliasing. A
    simpler example is keeping track of the number of calls to a
    particular method.

You could look at a Python reference as a kind of protection proxy,
since it controls access to the actual object on the heap (and ensures,
for example, that you don't use a `null` reference).

{{ Rewrite this: In *Design Patterns*, *Proxy* and *State* are not
seen as related to each other because the two are given (what I consider
arbitrarily) different structures. *State*, in particular, uses a
separate implementation hierarchy but this seems to me to be unnecessary
unless you have decided that the implementation is not under your
control (certainly a possibility, but if you own all the code there
seems to be no reason not to benefit from the elegance and helpfulness
of the single base class). In addition, *Proxy* need not use the same
base class for its implementation, as long as the proxy object is
controlling access to the object it "fronting" for. Regardless of the
specifics, in both *Proxy* and *State* a surrogate is passing method
calls through to an implementation object.}}\]
