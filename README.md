# Specula
Lower level language (still high) between C and Rust with heavy focus on interapplication communication and concurrency.
All files must end with `.spc`

## Entry Point'
All projects must have a single `main` function.

```
fn main(args: Array[String])
{
}
```
## Contract Handshaking
Contracts are basically protocols or traits like in `rust`. They define api's that can be used in code.
The difference is that they are stateful, they describe not just what data is exchanged, but its sequence and
failure recovery policy. This is generally only defined in the server.

``` 
contract MotionControl {
    state Idle <-> CommandSent -> Acknowledged -> Completed;
    state CommandSent -> Idle;
    state Idle -> Completed;

    send MoveCommand { direction: str, speed: float } @Idle -> CommandSent;
    recv Ack { accepted: bool } @CommandSent -> Acknowledged;
    recv Status { position: vec3, battery: float } @Acknowledged -> Completed;

    fail::low_battery; 
    auto-reset after Completed;
}

listener target 0.0.0.0:5050 using MotionControl as Server {
    on MoveCommand { direction, speed } {
        if (battery < 20.0) fail low_battery;
        print("Moving", direction, "at", speed);
        moveMotors(direction, speed);
        respond Ack(true);
    } -> State (transition-based)

    after Acknowledged {
        let pos = getPosition();
        respond Status(pos, battery);
    }
    
    on fail::low_battery {
        // Do something
    }
}
```

### Contract Message Events
| Event | Description |
| ------ | ----------- |
| send   | Client is sending, this event defines what should happen in the specific state |
| recv   | Same with send, but the client is receiving |

Definition of contract message event must be in
```
send/receive [identifier] { [params] } @BeforeState -> AfterState;
```

### Contract Events
The difference on this from *Contract Message Events* is its usually an event not requiring a message sent from a client or a server. This are purely server sided.
| Event | Description |
| ----- | ----------- |
| fail::fail_name | Definition of a failure mode (always server side call) |
| auto-reset | Resets back to initial state before or after a state |
| auto-move  | Auto moves on statement |

#### auto-reset 
Automatically resets back to initial state after a state is moved. Usage of auto-reset is:
```
auto-reset after [state];

// For multiple states you can:
auto-reset after [state1] | [state2] | [state3]
```
#### auto-move
Automatically moves to a specific state. Usage of auto-move is:
```
auto-move after [state] to [target-state]

// For multiple states you can:
auto-move after [state1] | [state2] | [state3] to [target-state]
```

## Listeners
Listeners are implementations of contracts. They define implementation on actions and failure modes. The definition of a listener is listed here:
```
listener target [url]:[port] using [contract] as [identifier] {
}
```

### Listener Message Events 
Listener defines events on the contract's `Contract Message Events`. To use these use the `on` keyword:
```
// [params] doesn't need type definition as its implied (but it's optional)
on [event_name] { [params] } { 
    [statements]
}


// ex.
on MoveCommand { direction, speed } {
    if (battery < 20.0) fail low_battery;
    print("Moving", direction, "at", speed);
    moveMotors(direction, speed);
    respond Ack(true);
}
```

### Listener Events
Like *Contract Events* it doesn't require a message sent from a client to server, purely transitions from States calls this
| Event | Description |
| ----- | ----------- |
| after | Happens after a `message event` is called|
| before | Happens before a `message event` is called|
Usage of `before` and `after` is like this:
```
before [state] {}
after [state] {}
```

### Listener Fail Events 
This defines failure implementation on a contract. Unlike message events, these are always server side.
The syntax for defining fail events is similar to message events, but with parenthesis as params and with a `fail::` before the failure name. 
> Params in fail events should always be using () and not {}
> Calling a fail event will skip the remaining statements in that message event like a `throw`

```
on fail::[identifier] ( [params] ) {

}

// General call without params 
fail [identifier];

// If it contains params
fail [identifier]([param_values]);
```

#### Moving states after fail
You can move to a specific state when approaching a fail with a `->` operator.
```
// Just move to a new state
on fail::[identifier] -> [target_state];

// Move after statement
on fail::[identifier] (params) { } -> [target_state];
```

### Differences between { } and ( ) in message events and fail events
To note, message events uses { } and fail events uses ( ) as parameter containers.
The reason is that message events are similar to struct containers which are data being passed from a different client. 
Fail events are like function calls, which are generally locally called by the server.


### Using Listeners
To use listeners use `listen` on it.
```
fn main()
{
    listen on [identifier];
}

// You could also define its url and port
fn main()
{
    listen on target [url]:[port] [identifier];
}
```

## Predictive async handling
predict runs a local predictive model (like Kalman filter, neural net, or motion estimate).
it can predict a future result before it’s fully available — then continue execution speculatively, rolling back or adjusting if the real data later disagrees.
```
fn computeData(): PredictAsync[int] {

}
```

### Confidence Thresholds
Predictive functions can be dangerous on sensitive systems, adding a confidence interval might help reduce its potential danger. A confidence of 1 always awaits the function
```
pos = predict camera.getTarget().atConfidence(0.8)
await moveTo(Pos)
```
Each prediction returns a confidence score — the runtime executes the action only if confidence > threshold.

```
fn async stabilizeFlight(drone: Drone): PredictAsync[bool] {
    estimatedPitch = predict drone.gyro.pitch().atConfidence(0.8)
    await adjustMotors(estimatedPitch);

    realPitch = await drone.gyro.pitch();
    if abs(realPitch - estimatedPitch) > 2deg {
        correctMotors(realPitch);
    }
}

() => {}
```

## Capabilities
Memory management defining which actions are permitted on that data. 
```
let x: int [mut]

own - ownership cannot be transferred (Default)
move - ownership can be transferred. 
shared - ownership can be shared with a reference counter.
view - specifies the object that it cannot own data. (Pass by reference only)
mut - can be modified 
const - cannot be modified (default)
thr_loc - separate copy per thread
sync - safe concurrent access (read/write)
infer - copy capabilities from assignee (generally used for parameters)
network - can be serialized and sent across nodes
 - basically just a tag that the compiler will ensure its serializable, encoded, and sanitized
ex.
network[json], network[html], network[specific format here]

fn async compute(x: String [view]]) -> int [local, mut] {

}
```
### Incompatible capability 
Not all capabilities support each other:
1. own & move & shared 
2. thr_loc & sync

### Moving ownership of data
To specify `[move]` use the `move` keyword. Ownership cannot be transferred to variables with `[view]`

```
let a: int [move] = 5;
let b = a;  // Copy 
let b = move a;  // Explicit move
```

### Reference of data
Ownership can be referenced using the `ref` keyword.
```
let a: int = 5;
let b = a; // Copy
let b: [view] = ref a;
```
Nested reference will ultimately point to its original owner.

### Non ownership of data
Using `[view]` will ensure that the variable will never own the data.
Passing a variable with `[view]` capability will point to its owner.


### Summary of ownership
Capability | Owns Data | Can Transfer Ownership | Can be shared | Can be viewed
|---|---|---|---|---|
`[own]` | / | | | / |
`[move]` | / | / | | |
`[ref]` | / | | / | / |
`[view]` | | | / | / |


### Infer Capabilities
Variables can copy capabilities from assignees using `[infer]`

```
ex.
let a: int [mut] = 5;
let b: int [infer] = a;  // equivalent to b: int[mut]
```
**Note**
1. Copy capabilities from multiple variables will be union. Incompatible capabilities will throw an error
```
let a: int [mut] = 5;
let b: int [move] = 5;
let c: int [infer] = a + b;  // equivalent to [mut, move]
```
2. Parameters can also inherit data from caller
```
let sum: int [mut] = sum(a, b);

fn sum(a: int[infer], b: int[infer])
{
    ret a + b;
}

NOTE: The compiler will perform function overloading on each capability
```

## Variable Scoping and Ownership
Capabilities handle the lifetime and scope of the variables

## Multi threading
```
thread testThread(param: int[copy])
{
    let x: List[int] [thr_loc, mut] = [];
    for i in 0..5 {
        buffer.push(i);
        sleep(100ms);
    }
}

usage:
let t1: ThreadFunc = spawn testThread(1);
let t2 = spawn testThread(2);
await t1;
await t2;
```

## Loops
### For loop
```
for (let i = 0; i < 5; i++) {
}

let arr: Array[int];
for (let i: int in arr)
{
}
```

### Parallel For loop
Shorthand for iterations in multiple threads.
```
let files: List[File];
parallel[threads=8] for (file in files) {
}
```

### While Loop
```
while (condition) {
}

do {

} while (condition);
```

## Identifier
Similar to c

```
[a-zA-Z_][a-zA-Z0-9_]
```

## Communication through different files
### Exporting
To export data, use the `[export]` keyword on top. `[export_default]` makes the value implicitly imported if it is not specified
```
extra.spc

[export]
fn toExport() { ret 1; }

[export_default]
fn defaultFunc() { ret 1; }

```

### Importing
To import add it as a top level statement
```
import { toExport } from "extra.spc"; //  imports toExport
import Extra from "extra.spc";  // imports defaultFunc
```

## Objects
The language is not a fully focused OOP, but it supports object like behaviour using `struct`, `interface`, and `impl`.
Similar to Rust's `struct`, `traits`, `impl`, this makes the language focus more on composition rather than inheritance.
This improves compile time guarantees and avoids class hierarchies.

### Struct
An object containing only **data**
```
struct Vector2 {
    let x: int [mut];
    let y: int [mut];
}
```

### Interface
Similar to Rust's `traits`, it defines **function declarations**
```
interface Vector {
    fn add(other: self[view]): self;
    fn sub(other: self[view]): self;
}
```

### Impl
Similar to Rust's `impl`, it **defines functions** of an interface connecting structs. 
```
impl Vector using Vector2 {
    fn add(other: Vector2[view]): Vector2 {
        ret Vector2 { 
            x: this.x + other.x,
            y: this.y + other.y
        };
    }

    fn sub(other: Vector2[view]): Vector2 {
        ret Vector2 { 
            x: this.x - other.x,
            y: this.y - other.y
        };
    }
}
```

### Example Usage
```
fn main() {
    let vec1 = Vector2 { x: 5, y: 2};
    let vec2 = Vector2 { x: 3, y: 5};
    let result = vec1.add(vec2);
    print(result.x);
    print(result.y);
}
```

### Difference between "this" and "self"
| Keyword | Meaning |
| ------- | ------- |
| self | refers to a type that will be used to an `impl`. Its generally used by `interface`. So if `impl` uses `Vector2`, self will turn into `Vector2`. |
| this | refers to the current instance itself. It contains the data owned by the `struct` itself. |

