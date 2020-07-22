# Continuation-Passing Style [![Build Status](https://travis-ci.org/disruptek/cps.svg?branch=master)](https://travis-ci.org/disruptek/cps)

This project provides a macro `cps` which you can apply to a procedure to
rewrite it to use continuations for control flow.

For a description of the origins of this concept, see the included papers
and https://github.com/zevv/nimcsp.

_Windows is not supported yet; feel free to submit a pull request!_

## Why

These continuations...

- ...compose efficient and idiomatic asynchronous code
- ...are over a thousand times lighter than threads
- ...compose beautifully with Nim's ARC memory management
- ...may be based upon your own custom `ref object`
- ...may be dispatched using your own custom dispatcher
- ...may be moved between threads to parallelize execution

## Usage

A simple `selectors`-based event queue is provided, which includes some `cps`
primitives to enable asynchronous operations. You can replace this with your
own implementation if you choose.

```nim
import std/times

import cps             # .cps. macro
import cps/eventqueue  # cps_sleep(), trampoline, run(), Cont

# a procedure that starts off synchronous and becomes asynchronous
proc tock(name: string; interval: Duration): Cont {.cps.} =
  var count: int = 0
  while true:
    inc count
    # this primitive sends the continuation to the dispatcher
    cps_sleep interval
    # this is executed from the dispatcher
    echo name, " ", count

# the trampoline repeatedly invokes continuations...
trampoline tock("tick", initDuration(milliseconds = 300))
# ...until they complete or are queued in the dispatcher
trampoline tock("tock", initDuration(milliseconds = 700))

# run the dispatcher to invoke pending continuations
run()
```
...is rewritten during compilation to something like...

```nim
type
  env_16446076 = ref object of Cont
    name: string
    interval: Duration

  env_16446209 = ref object of env_16446076
    count: int

proc loop_16446121(locals_16446228: Cont): Cont =
  let interval: Duration = env_16446209(locals_16446228).interval
  let name: string = env_16446209(locals_16446228).name
  var count: int = env_16446209(locals_16446228).count
  if true:
    inc count
    cps_sleep interval
    echo name, " ", count
    return env_16446209(fn: loop_16446121, count: count, name: name, interval: interval).Cont

proc tock(name: string; interval: Duration): Cont =
  var count: int = 0
  return env_16446209(fn: loop_16446121, count: count, name: name, interval: interval).Cont

trampoline tock("tick", initDuration(milliseconds = 300))
trampoline tock("tock", initDuration(milliseconds = 700))

run()
```

## Documentation
See [the documentation for the cps module](https://disruptek.github.io/cps/cps.html) as generated directly from the source.

## License
MIT
