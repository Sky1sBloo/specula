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
failure recovery policy

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
    node RobotUnit {
        on MoveCommand { direction, speed } -> Acknowledged {
            if (battery < 20.0) fail low_battery;
            print("Moving", direction, "at", speed);
            moveMotors(direction, speed);
            respond Ack(true);
        }

        after Ack -> Completed {
            let pos = getPosition();
            respond Status(pos, battery);
        }
        
        on fail::low_battery {
            // Do something
        }
    }
}
```
## Listeners
Listeners are implementations of contracts. They define implementation on actions and failure modes. The definition of a listener is listed here:
```
listener target [url]:[port] using [contract] as [identifier] {
}
```
To use it, create use `listen` on it.
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

fn async compute(x: int [readonly]) -> int [local, mut] {

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
