+++
date = "2017-12-10"
author = "Marc-Antoine Ruel"
authorlink = "https://maruel.ca"
title = "Reaching 82Msps"
description = "How we cranked up GPIO performance in v2.1.0"
tags = []
notruncate = false
+++


One of the key design goal of periph.io is to be as fast as possible while
exposing a simple API. For GPIO pins, this means having low latency in the happy
path, which can be measured for output by toggling the output low and high
continuously as fast as possible and for inputs by, _well_, reading continuously.

Optimizing without data is useless, so the first step has been to write a
reproducible benchmark that could be used across platforms.

<!--more-->


# Use case

Why micro-optimize the GPIO to be as fast as possible? The use cases includes:

- [bit banging](https://en.wikipedia.org/wiki/Bit_banging), which can be used to
  emulate a protocol.
- Create a software-based best-effort logic analyzer.
- using as little CPU overhead for devices on CPU bound operation especially in
  the case of single-core platforms like the Raspberry Pi Zero and the C.H.I.P.

The question is: how fast can we go?


# Output

## Methodology

I took measurements via
[periph-smoketest](https://github.com/google/periph/tree/master/cmd/periph-smoketest),
which leverages the excellent benchmarking support in package
[testing](https://golang.org/pkg/testing/#hdr-Benchmarks) from the Go standard
library.

All builds were done with Go 1.9.2.

The output benchmark toggles the output
[gpio.High](https://periph.io/x/periph/conn/gpio#Level) then
[gpio.Low](https://periph.io/x/periph/conn/gpio#Level), for each single
iteration. So each measurement is in fact measuring two changes, which amounts
to a single clock signal.

The 3 boards tested are:

- [C.H.I.P.](/platform/chip/), both with a native GPIO and a XIO-Pn GPIO
  (CSID0/PE4/132 and XIO-P0/1013 were used)
- [Raspberry Pi 3](/platform/raspberrypi/) running Stretch Lite (GPIO12 was
  used)
- [Raspberry Pi Zero Wireless](/platform/raspberrypi/) running Jessie Lite
  (GPIO12 was used)

The command used was one of:

- `periph-smoketest sysfs-benchmark -p NN`
- `periph-smoketest bcm283x-benchmark -p NN`

## Initial Results

The initial results weren't really good, but gave a starting point.

|            | C.H.I.P. XIO-Pn | C.H.I.P. GPIO | RPi3   | RPiZW  |
|------------|-----------------|---------------|--------|--------|
| sys-fs     | 2.0Khz          | 44Khz         | 74Khz  | 21Khz  |
| CPU driver | _N/A_           | 490kHz        | 1.6MHz | 1.7MHz |


Screenshot of a slower run:

{{< figure src="/img/news_2017_gpio_perf_before.png" title="Blue is RPi3, yellow is RPiZW" >}}

## Key points

- The CPU driver were definitely lower than what could be achieved by an order
  of magnitude.
- Using sysfs kernel driver incurs a cost between 11x to 81x in term of
  performance.
- The sysfs driver is significantly slower on the RPiZW than the RPi3 for an
  unknown reason, maybe due to Debian Stretch vs Jessie. Even the CHIP is
  faster.
- The C.H.I.P. with the CPU driver is 3.5 times slower than a Raspberry Pi Zero
  Wireless even with the same clock speed (1GHz).
- The XIO-Px pins on the C.H.I.P. are very slow due to the I²C communication
  overhead. These should only be used for signals that changes infrequently,
  e.g. driving a LED.


## First optimization

The first fix was to reduce the overhead due to Go's bound checking code. This
was done by hardcoding the offsets when possible via an eleven lines commit
[a929d4be37](
https://github.com/google/periph/commit/a929d4be3796b67d5ee7359f9d7a474e05453a03).

|                     | C.H.I.P. GPIO | RPi3   | RPiZW  |
|---------------------|---------------|--------|--------|
| CPU driver (before) | 490kHz        | 1.6MHz | 1.7MHz |
| CPU driver (after)  | 540khz        | 2.2MHz | 2.1MHz |

That is a very impressive improvements with a simple change. Boundary checks in
Go cost a lot, so for performance sensitive code, it is worth doing micro
benchmarks to determine if tweaks can be done to reduce the cost.

A notorious example is the [binary
package](https://go.googlesource.com/go/+/go1.9.2/src/encoding/binary/binary.go#52).


### Diving deeper

To better understand why this happens, a good source of information is to use
the `GOSSAFUNC` environment variable. Sadly all the examples on the internet
uses `GOSSAFUNC=main` but what we want to analyze is
`periph.io/x/periph/host/bcm283x.(*Pin).Out`. After trying to guess a few
variations, I decided to add `println(name)` to [buildssa() in
ssa.go](https://go.googlesource.com/go/+/go1.9.2/src/cmd/compile/internal/gc/ssa.go#105),
built the toolchain via `./make.bash` which takes less than a minute on my
workstation and determined that what I needed is the name without the package
`(*Pin).Out`:

```
GOOS=linux GOARCH=arm GOSSAFUNC="(*Pin).Out" go build |& less
```

#### ssa.html

**Warning**: The `ssa.html` file is dumped in the package directory that
contains the function, not in the current directory! (I spent a ridiculous
amount of time to figure this out)

```
cat > main.go <<EOF
package main
import "periph.io/x/periph/conn/gpio"
import "periph.io/x/periph/host/bcm283x"
func main() { bcm283x.GPIO0.Out(gpio.Low) }
EOF
git checkout a929d4be37
GOOS=linux GOARCH=arm GOSSAFUNC="(*Pin).Out" go build
mv $GOPATH/src/periph.io/x/periph/host/bcm238x/ssa.html fast.html
git checkout HEAD~1
GOOS=linux GOARCH=arm GOSSAFUNC="(*Pin).Out" go build
mv $GOPATH/src/periph.io/x/periph/host/bcm238x/ssa.html slow.html
```

Then you can load the two pages side-by-side and look at `b7`.

_Pro-tip_: use the web browser's 'Find in page' functionality with `gpioMap` to
focus on the important parts.


#### Assembly

The function assembly code is output and we can leverage that. The assembly code
is printed at the very end of the output with an header `genssa`, which appears
twice in the output, so the second must be used as the cutoff. Also we want to
remove the file and line offset since it makes the listing noisy. So I leveraged
this new found knowledge to compare the two following snippets:

```
cat > main.go <<EOF
package main
import "periph.io/x/periph/conn/gpio"
import "periph.io/x/periph/host/bcm283x"
func main() { bcm283x.GPIO0.Out(gpio.Low) }
EOF
git checkout a929d4be37
GOOS=linux GOARCH=arm GOSSAFUNC="(*Pin).Out" go build |& sed -n -e '/genssa/,$p' | tail -n +2 | sed -n -e '/genssa/,$p' | sed -e 's/([^(]\+go\:[0-9]\+)//g' | sed -e 's/^[^\t]\+//g' > fast.txt
git checkout HEAD~1
GOOS=linux GOARCH=arm GOSSAFUNC="(*Pin).Out" go build |& sed -n -e '/genssa/,$p' | tail -n +2 | sed -n -e '/genssa/,$p' | sed -e 's/([^(]\+go\:[0-9]\+)//g' | sed -e 's/^[^\t]\+//g' > slow.txt
```

Armed with `vimdiff` we can clearly see the `CALL  runtime.panicindex` in the
slow code path that are missing in the fast one, and one less conditional
jump in the hot path.

#### Side note

Some checks are deterministically differents in the two generated code which
makes the overall delta'ing a bit more difficult. To the rescue is the
excellent [IDA Pro](https://www.hex-rays.com/products/ida/) software, with one
of [Hex-Rays
Decompiler](https://www.hex-rays.com/products/decompiler/index.shtml) or the
free (!) [BinDiff](https://www.zynamics.com/software.html). Sadly my license
lapsed many years ago and my setup failed into disrepair so deeper analysis
will be for another time. :(


## Cutting out the safety

To get further into deep performance, the safety checks in `Out()` need to be
removed. This was done by adding a new function `FastOut()` that skips all the
safety checks. In practice, `Out()` does the safety checks, then calls
`FastOut()`.

Interestingly, inlining the
[gpio.Level](https://periph.io/x/periph/conn/gpio#Level) argument, creating one
dedicated function per level, `FastOutLow()` and `FastOutHigh()`, doesn't
significantly affect the performance on the Raspberry Pi 3 but does give a 18%
performance boost on the Raspberry Pi Zero Wireless on the micro benchmark. I
suspect that the processor on the RPi3 is more superscalar than the RPiZW. In
practice, this would make the call sites more tricky when the level is not known
in advance (hard coded) so this "optimization" was left out.

If you take a look at `FastOut()` you may be tempted to hardcode the RPi bit
mask used to set the CPU register. In practice this didn't affect the
performance at all.

This was done with minimal API increase with a single new function:

|             | C.H.I.P. GPIO | RPi3    | RPiZW  |
|-------------|---------------|---------|--------|
| `Out()`     | 540kHz        | 2.2MHz  | 2.1MHz |
| `FastOut()` | 2.3MHz        | 41.3MHz | 14MHz  |

Now we're talking! This is 82.6 megasamples per second for the RPi3, and the
C.H.I.P. is now in the MHz range. In fact, this is so fast on the RPis that my
oscilloscope has an hard time getting a clear signal:

{{< figure src="/img/news_2017_gpio_perf_after.png" title="« We'll need a bigger oscilloscope »" >}}

There's also a risk that the signal integrity at this point can just not be
guaranteed with the GPIO so you will need to plan the electric properties of the
design accordingly.


## Anti-pattern

Astute readers will notice that there is no interface in the package
[gpio](https://periph.io/x/periph/conn/gpio) declaring the function `FastOut()`.
A user may be tempted to do something like this:

```go
type fastOuter interface {
  Out(l gpio.Level) error
  FastOut(l gpio.Level)
}

var p fastOuter = gpioreg.ByName("GPIO12")
p.Out(gpio.Low)
p.FastOut(gpio.High)
...
```

**Don't!** At least, don't without testing the performance implications. Local
testing lowered the performance on a RPi3 **from 41MHz down to 16.4MHz!** This
is a dramatic performance decrease, so it is very important to keep the code to
only do concrete direct calls and not use an interface.


## Risks

Keep in mind the risks of using `FastOut()`:

- you must assert that `Out()` succeeded before and that no other call was done,
  like `In()`.
- you must keep only direct concrete calls and no interfaces in the hot path to
  extract the maximum juice of the platform, otherwise you'll lose 60% of the
  performance right away.


# Input

So we optimized `Out()` and created a fast path function `FastOut()`. What about
input now?


## Methodology

Similar to output, input latency needs to be consistently measured.
The first step is to create the benchmark and run it on platforms.

```go
p := s.p
if err := p.In(gpio.PullDown, gpio.NoEdge); err != nil {
  b.Fatal(err)
}
b.ResetTimer()
for i := 0; i < b.N; i++ {
  p.Read()
}
```

|            | C.H.I.P. XIO-Pn | C.H.I.P. GPIO | RPi3   | RPiZW |
|------------|-----------------|---------------|--------|-------|
| sys-fs     | 4.0Khz          | 74Khz         | 135Khz | 37Khz |
| CPU driver | _N/A_           | 66Mhz         | 70MHz  | 41MHz |

Clearly, the input code was fairly performant! This is even more surprising with
the C.H.I.P. which outperforms the RPiZW by a wide marging.

Right?


## Fast benchmarks, and lies

But what is the data measured above was a complete lie? Remember that the SSA
optimizer can optimize across concrete calls, so if a value read is never used,
the read operation may be optimized out. To ensure this is true, we change the
micro benchmark from the version above to:

```go
p := s.p
if err := p.In(gpio.PullDown, gpio.NoEdge); err != nil {
  b.Fatal(err)
}
l := gpio.Low
b.ResetTimer()
for i := 0; i < b.N; i++ {
  l = p.Read()
}
b.StopTimer()
b.Log(l)
```

and here's the results:

|                   | C.H.I.P. GPIO | RPi3    | RPiZW   |
|-------------------|---------------|---------|---------|
| CPU driver (lie)  | 66Mhz         | 70MHz   | 41MHz   |
| CPU driver (real) | 8.9MHz        | 10.5Mhz | 12.3MHz |

Ouch! That hurts. This proves that the naive benchmark wasn't testing much at
all, and it is important to trick the SSA optimizer into not optimizing _too
much_.


## Input optimization

### The Ugly

The reading code looks like this:

```go
func (p *Pin) Read() gpio.Level {
  if gpioMemory == nil {
    return gpio.Low
  }
  return gpio.Level((gpioMemory.level[p.number/32] & (1 << uint(p.number&31))) != 0)
}
```

One would be tempted to do, similar to `FastOut()`, skip the gpioMemory == nil
to shorten it as one line:

```go
func (p *Pin) FastRead() gpio.Level {
  return gpio.Level((gpioMemory.level[p.number/32] & (1 << uint(p.number&31))) != 0)
}
```

Interestingly, this code is _slower_, and not just a bit, it's **2x slower**,
resulting a reading speed of 5.9MHz on the RPi3. So in Go, it's sometimes
important to keep these checks, to prove to the compiler that the code cannot
panic.


### The Bad

Wait a minute! `FastOut()` doesn't have the check for `gpioMemory == nil`. What
happens if it is added?

The funny thing is that it becomes ~2% faster on the RPi3 and ~2% slower on the
RPiZW, so it was not added.


### The Good

On the other hand, the same indexing hardcoding done for write can be
leveraged as done in commits [TODO](TODO) and [TODO](TODO).

|                     | C.H.I.P. GPIO | RPi3    | RPiZW   |
|---------------------|---------------|---------|---------|
| CPU driver (before) | 8.9MHz        | 10.5Mhz | 12.3MHz |
| CPU driver (after)  | 8.9MHz        | 11.3Mhz | 13.6MHz |

In this case, expanding the switch case on the C.H.I.P. has measurably no
effect, but it gives a 7%~10% performance boost on the RPi3, which is not
negligible.

 Keep in mind that the `In()` performance readings are 2x the `Out()` readings
 because `In` measures a single read, instead of two operations for `Out`.


# Conclusion

- Beware of the danger of micro benchmarks. They can give you an incorrect view
  of the reality and make you optimize for the wrong things.
- Learn to love and fear the optimizer, and its inscrutable entrails. The
  optimization process can be counter-intuitive at times.
- Reading a digital I/O is much slower than writing. It makes sense, as it takes
  time to initiate the sampling. The fact that it is 8x slower is a bit
  surprising, so take this in account.
- It's not because a platform is faster at writing that it will also be faster
  at reading, as shown above.

If you need performance at all costs, you can now do the following:

```go
p, ok := gpioreg.ByName("GPIO12").(*bcm283x.Pin)
if !ok {
  // Either fallback to slow path or bail out.
}
// Call Out() at least once, to ensure the pin is initialize correctly.
if err := p.Out(gpio.Low); err != nil {
  log.Fatal(err)
}
// Then use FastOut() to stay on the fast path:
p.FastOut(gpio.High)
```

On the other hand, reading is now optimized with no API change at all.

Happy bit banging!

*p.s.:* Did you find a factual error? An incorrect methodology? It happens! Or
just excited about this research and want to discuss it more? Please join the
#periph Slack channel (see top right of the page) to comment about this with the
author!
