# statig

[![Current crates.io version](https://img.shields.io/crates/v/statig.svg)](https://crates.io/crates/statig)
[![Documentation](https://docs.rs/statig/badge.svg)](https://docs.rs/statig)
[![Rust version](https://img.shields.io/badge/Rust-1.65.0-blue)](https://releases.rs/docs/released/1.65.0)
[![CI](https://github.com/mdeloof/statig/actions/workflows/ci.yml/badge.svg)](https://github.com/mdeloof/statig/actions/workflows/ci.yml)


Hierarchical state machines for designing event-driven systems.

**Features**

- Hierachical state machines
- State-local storage
- Compatible with `#![no_std]`, no heap memory allocation
- (Optional) macro's for reducing boilerplate.

---

**Overview**

- [Statig in action](#statig-in-action)
- [Concepts](#concepts)
    - [States](#states)
    - [Superstates](#superstates)
    - [Actions](#actions)
    - [Context](#context)
    - [State-local storage](#state-local-storage)
- [Implementation](#implementation)
- [FAQ](#faq)
- [Credits](#credits)

---

## Statig in action

A simple blinky state machine:

```
┌─────────────────────────┐                   
│         Blinking        │◀─────────┐        
│    ┌───────────────┐    │          │        
│ ┌─▶│     LedOn     │──┐ │  ┌───────────────┐
│ │  └───────────────┘  │ │  │  NotBlinking  │
│ │  ┌───────────────┐  │ │  └───────────────┘
│ └──│     LedOff    │◀─┘ │          ▲        
│    └───────────────┘    │──────────┘        
└─────────────────────────┘                                   
```

```rust
#[derive(Default)]
pub struct Blinky;

pub enum Event {
    TimerElapsed,
    ButtonPressed
}

#[state_machine(initial = "State::led_on()")]
impl Blinky {
    #[state(superstate = "blinking")]
    fn led_on(event: &Event) -> Response<State> {
        match event {
            Event::TimerElapsed => Transition(State::led_off()),
            _ => Super
        }
    }

    #[state(superstate = "blinking")]
    fn led_off(event: &Event) -> Response<State> {
        match event {
            Event::TimerElapsed => Transition(State::led_on()),
            _ => Super
        }
    }

    #[superstate]
    fn blinking(event: &Event) -> Response<State> {
        match event {
            Event::ButtonPressed => Transition(State::not_blinking()),
            _ => Super
        }
    }

    #[state]
    fn not_blinking(event: &Event) -> Response<State> {
        match event {
            Event::ButtonPressed => Transition(State::led_on()),
            _ => Super
        }
    }
}

fn main() {
    let mut state_machine = Blinky::default().state_machine().init();

    state_machine.handle(&Event::TimerElapsed);

    state_machine.handle(&Event::ButtonPressed);
}
```

(See the [`macro/blinky`](examples/macro/blinky/src/main.rs) example for the full code with comments. Or see [`no_macro/blinky`](examples/no_macro/blinky/src/main.rs) for a version without using macro's).


---

## Concepts

### States

States are defined by writing methods inside the `impl` block and adding the `#[state]` attribute to them. By default the `event` argument will map to the event handled by the state machine.

```rust
#[state]
fn led_on(event: &Event) -> Response<State> {
    Transition(State::led_off())
}
```

Every state must return a `Response`. A `Response` can be one of three things:

- `Handled`: The event has been handled.
- `Transition`: Transition to another state.
- `Super`: Defer the event to the next superstate.

### Superstates

Superstates allow you to create a hierarchy of states. States can defer an event to their superstate by returning the `Super` response.

```rust
#[state(superstate = "blinking")]
fn led_on(event: &Event) -> Response<State> {
    match event {
        Event::TimerElapsed => Transition(State::led_off()),
        Event::ButtonPressed => Super
    }
}

#[superstate]
fn blinking(event: &Event) -> Response<State> {
    match event {
        Event::ButtonPressed => Transition(State::not_blinking()),
        _ => Super
    }
}
```

Superstates can themselves also have superstates.

### Actions

Actions run when entering or leaving states during a transition.

```rust
#[state(entry_action = "enter_led_on", exit_action = "exit_led_on")]
fn led_on(event: &Event) -> Response<State> {
    Transition(State::off())
}

#[action]
fn enter_led_on() {
    println!("Entered on");
}

#[action]
fn exit_led_on() {
    println!("Exited on");
}
```

### Context

If the type on which your state machine is implemented has any fields, you can access them inside all states, superstates or actions.

```rust
#[state]
fn led_on(&mut self, event: &Event) -> Response<State> {
    match event {
        Event::TimerElapsed => {
            self.led = false;
            Transition(State::led_off())
        }
        _ => Super
    }
}
```

Or alternatively, set `led` inside the entry action.

```rust
#[action]
fn enter_led_off(&mut self) {
    self.led = false;
}
```

### State-local storage

Sometimes you have data that only exists in a certain state. Instead of adding this data to the context and potentially having to unwrap an `Option<T>`, you can add it as an input to your state handler.

```rust
#[state]
fn led_on(counter: &mut u32, event: &Event) -> Response<State> {
    match event {
        Event::TimerElapsed => {
            *counter -= 1;
            if *counter == 0 {
                Transition(State::led_off())
            } else {
                Handled
            }
        }
        Event::ButtonPressed => Transition(State::led_on(10))
    }
}
```

`counter` is only available in the `led_on` state but can also be accessed in its superstates and actions.

---

## Implementation

A lot of the implemenation details are dealt with by the `#[state_machine]` macro, but it's always valuable to understand what's happening behind the scenes.

The goal of `statig` is to represent a hierarchical state machine. Conceptually a hierarchical state machine can be tought of as graph.

```
                          ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐             
                                    Top                       
                          └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘             
                                     │                        
                        ┌────────────┴────────────┐           
                        │                         │           
             ┌─────────────────────┐   ╔═════════════════════╗
             │      Blinking       │   ║      NotBlinking    ║
             │─────────────────────│   ╚═════════════════════╝
             │ counter: &'a usize  │                          
             └─────────────────────┘                          
                        │                                     
           ┌────────────┴────────────┐                        
           │                         │                        
╔═════════════════════╗   ╔═════════════════════╗             
║        LedOn        ║   ║        LedOff       ║             
║─────────────────────║   ║─────────────────────║             
║ counter: usize      ║   ║ counter: usize      ║             
╚═════════════════════╝   ╚═════════════════════╝                                 
```

Nodes at the edge of the graph are called leaf-states and are represented by an `enum` in `statig`. If data only exists in a particular state we can give that state ownership of the data. This is referred to as 'state-local storage'. For example `counter` only exists in the `LedOn` and `LedOff` state.

```rust
enum State {
    On { counter: usize },
    Off { counter: usize },
    Paused
}
```

States such as `blinking` are called superstates. They define shared behavior of their child states. Superstates are also represented by an enum, but instead of owning their data, they borrow it from the underlying state.

```rust
enum Superstate<'a> {
    Playing { counter: &'a usize }
}
```

The graph structure is then expressed in the `superstate` method of the `State` and `Superstate` trait.

```rust
impl statig::State<Blinky> for State {

    // Other methods omitted.

    fn superstate(&mut self) -> Option<Superstate<'_>> {
        match self {
            State::On { counter } => Some(Superstate::Playing { counter }),
            State::Off { counter } => Some(Superstate::Playing { counter }),
            State::Paused => None
        }
    }
}

impl<'a> statig::Superstate<Blinky> for Superstate<'a> {

    // Other methods omitted.

    fn superstate(&mut self) -> Option<Superstate<'_>> {
        match self {
            Superstate::Playing { .. } => None
        }
    }
}
```

When an event arrives, `statig` will first dispatch it to the current leaf state. If this state returns a `Super` response, it will then be dispatched to that state's superstate, which in turn returns its own response. Every time an event is defered to a superstate, `statig` will traverse upwards in the graph until it reaches the `Top` state. This is an implicit superstate that will consider every event as handled.

In case the returned response is a `Transition`, `statig` will perform a transition sequence by traversing the graph from the current source state to the target state by taking the shortest possible path. When this path is going upwards from the source state, every state that is passed will have its **exit action** executed. And then similarly when going downward, every state that is passed will have its **entry action** executed.

For example when transitioning from the `LedOn` state to the `NotBlinking` state the transition sequence looks like this:

1. Exit the `LedOn` state
2. Exit the `Blinking` state
3. Enter the `NotBlinking` state

For comparison, the transition from the `LedOn` state to the `LedOff` state looks like this:

1. Exit the `LedOn` state
2. Enter the `LedOff` state

We don't execute the exit or entry action of `Blinking` as this superstate is shared between the `LedOn` and `LedOff` state.

Entry and exit actions also have access to state-local storage, but note that exit actions operate on state-local storage of the source state and that entry actions operate on the state-local storage of the target state.

For example chaning the value of `counter` in the exit action of `LedOn` will have no effect on the value of `counter` in the `LedOff` state.

---

## FAQ

### **What is this `#[state_machine]` proc-macro doing to my code? 🤨**

Short answer: nothing. `#[state_machine]` simply parses the underlying `impl` block and derives some code based on its content and adds it to your source file. Your code will still be there, unchanged. In fact `#[state_machine]` could have been a derive macro, but at the moment Rust only allows derive macros to be used on enums and structs. If you'd like to see what the generated code looks like take a look at the test [with](./statig/tests/transition_macro.rs) and [without](./statig/tests/transition.rs) macros.

### What advantage does this have over using the typestate pattern?

I would say they serve a different purpose. The [typestate pattern](http://cliffle.com/blog/rust-typestate/) is very useful for designing an API as it is able to enforce the validity of operations at compile time by making each state a unique type. But `statig` is designed to model a dynamic system where events originate externally and the order of operations is determined at run time. More concretely, this means that the state machine is going to sit in a loop where events are read from a queue and submitted to the state machine using the `handle()` method. If we want to do the same with a state machine that uses the typestate pattern we'd have to use an enum to wrap all our different states and match events to operations on these states. This means extra boilerplate code for little advantage as the order of operations is unknown so it can't be checked at compile time. On the other hand `statig` gives you the ability to create a hierarchy of states which I find to be invaluable as state machines grow in complexity.

---

## Credits

The idea for this library came from reading the book [Practical UML Statecharts in C/C++](https://www.state-machine.com/doc/PSiCC2.pdf). I highly recommend it if you want to learn how to use state machines to design complex systems.
