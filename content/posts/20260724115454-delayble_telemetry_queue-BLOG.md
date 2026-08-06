+++
title = "Delayable Telemetry Queue | Design Notes - indy-tui"
author = ["Cole"]
lastmod = 2026-08-06T08:23:40-04:00
tags = ["design-notes", "indy-tui", "cpp"]
draft = false
katex = false
+++

As the core impetus for my most recent _(and ongoing)_ project, I think a break down of my design and implementation of
the delay queue used in [#indy-tui](/tags/indy-tui) is perfectly befitting of a first post.

I'll start with a little background on the project, follwed by a
little more on the problem I was trying to solve and why I'm/ taking a crack at it. Then, I'll touch on the two main
approaches I considered, what primary goals I was trying to meet, and ultimately which of the approaches stuck. Finally,
I'll go a little more technically in-depth on the implementation I have today.

If you are a motorsports fan, the non-technical part(s) of this post should be right up your
alley. If you aren't, fear not -- the aforementioned non-technical part(s) should be quite
limited. With that said, [why not just... become one](https://www.youtube.com/watch?v=Ubt9vnioGyA)? [I mean it's pretty cool](https://www.youtube.com/watch?v=uKEW7cCxxjk)[^fn:1].

This is _a_ solution, but certainly not _the_ solution -- my hope is to illustrate why I picked this one.


## Background {#background}


### The problem {#the-problem}

Compared to "stick and ball" sports like hockey, football, or soccer, broadcasting motorsports
presents a unique challenge: the broadcast can only ever show a very small fraction of what's happening at any given
time. Aside from the primary challenge that it's just not possible to show all 25+ cars spanning miles on screen at
once, each of those cars has a team plotting strategy _(sometimes)_ on the radio throughout the duration of the
event. Needless to say, if your favorite driver is not in the top X or is clearly charging through the field, it's not a
given they'll get much coverage (if any).

To their credit, various broadcasts have made progress trying to tackle this problem. The most important of these
spectator assist devices is inarguably the on screen leaderboard -- a live graphic which displays the state of the race
(flag status, laps completed/laps to go) and, at minimum, the current position of each driver in the field. When used
more effectively, the broadcaster will include more live information on the leaderboard, data points like each driver's
gap to the leader, the number of laps since they last pitted, or the tire compound they're currently equipped with.

For the more observant viewer, the leaderboard goes a long way in filling the gaps inherent to motorsports
coverage. But, for the more impatiently observant viewer[^fn:2], this presents a new pain point
in the broadcast's production: _rarely_ is the leaderboard showing the information you would like to know at any given
time. Thankfully, today most premier motorsports series provide a much more thorough live leaderboard separate from the
broadcast, often by app, which displays most of the information at once. With this alternative, said impatiently
observant viewers[^fn:2] should be satisfied -- they have all of the information when they want it, even during
commercials. So... they are satisfied right? Right?

Have you ever had an upstairs neighbor who somehow has a slightly faster stream of the football game that you both are
watching? Watched a pass thrown and had all suspense robbed from you as said neighbor is celebrating the touchdown when,
for you, the ball is still in the air? Unfortunately, unless you are lucky enough to somehow[^fn:3]
have a broadcast with next to zero delay, this upstairs neighbor and these online leaderboard options are one in the
same -- the online leaderboards have a very neglible delay, while broadcasts do not. Using what the series gives you,
your two options are to either just take the on screen leaderboard and accept the lack of spoilers over getting the
information you want, or to use their online leaderboard and get the information desired but spoilers along with it.

All that's really needed here is a way to delay the leaderboard information _(from here on "telemetry")_ by a configurable
amount. While I don't know of any major series which offers this directly, there is third party tooling which does[^fn:4]. Although the options I've tried out do work, I've
had a few main gripes:

1.  **The delay is configured in milliseconds**. When I'm trying to measure a delay while watching a race, I
    personally don't want to bother doing the math of adding the right number of zeros to the rough number I've
    observed. At most, I want control down to half-seconds[^fn:5].
2.  **When accruing a delay, the interface stalls and doesn't communicate whether it is broken, there is no telemetry
    coming in, or it is simply building up a delay**. I would like a small portion of the interface to show the status of
    the delay, for example how much of a delay is currently built up.
3.  **The delay overshoots or undershoots over long spans**. I have observed times where after adjusting the delay back and
    forth, it is not uncommon to wind up with the observed delay markedly longer than the configured delay.
4.  **The tools are not open source**. Ultimately, this is the most consequential gripe -- if I am the only person in the
    world that has the above issues, I will happily fork the project and make it most comfortable for myself. Without
    that ability, I am left to either suck it up or try my hand at a solution.

So, try my hand I did.


### Goals {#goals}

1.  **The delay is consistent**.
    -   There should be strong guards against undershoot and overshoot.
2.  **Exact delay is more important than continuity of telemetry**.
    -   There will be stutters in the telemetry we receive. I care that the duration between front and
        back of the queue matches the desired delay much more than I can that the telemetry shown is
        continuous _(w/o skipping)_
3.  **The delay is configured in second increments**.
    -   If implemented well, this is easily updated later to be half-second _(or smaller)_ increments.
4.  **The queue is thread-safe**.
    -   When used in the project, we'll want to have what is basically a producer-consumer setup.
5.  **Delay status is retrievable**.
    -   We'll want to show on the interface whether the delay is satisfied, and if not how much of a delay is accrued.
6.  **The delay can be zero**.
    -   If the user wants a "true" live feed, we don't want to have a second queue to accommodate it.


## Two approaches {#two-approaches}

When first tackling this problem, two main solutions jumped out at me -- a dynamic queue, and a fixed-size queue.


### Dynamic queue {#dynamic-queue}

The first, and what I suspect most other tools attempting this use, is a straightforward thread-safe queue where the
"producer" thread enqueues receieved telemetry paired with a timestamp, and the "consumer" thread dequeues telemetry
frames only if the duration between now and that frame's timestamp satisfy the delay.

This solution satisfies goals three, four, five, and partially satisfies goal one as undershooting would be
impossible. This solution also benefits from a fairly straightforward implementation, likely requiring minimal work on
top of the C++ standard library.

A byproduct of this implementation is that as long as we receive a given telemetry frame, the user will
"see"[^fn:6] every
frame. Note that I described this as a byproduct rather than a benefit -- this actually violates goal two. It is
unavoidable that the telemetry we receive will be at an inconsistent rate, so by ensuring this continuity we run the
risk of a delay overshoot _(thus also violating the other half of goal one)_. With this solution, we end up implementing a
"best case" delay; if we receive telemetry faster than we can process and display it, we will quickly build up a surplus
of unprocessed telemetry frames, resulting in a delay overshoot.

If the end goal were a data collection tool for later analysis of race telemetry _(which wouldn't require delay
functionality)_ this continuity would likely be paramount, and this solution would certainly win out. But, that's not our
application, so we need another plan.


### Fixed-size queue {#fixed-size-queue}

This fixed-sized queue leverages two truths in this specific application: TUI users don't expect the utmost temporal
resolution for data presentation, and only so much data is actually understandable to humans at once. That is to
say, in this application, there is quickly a limit to how fast the interface's refresh rate actually needs to be. Thus,
we can fix a refresh rate, for now at 10 Hz.

Given a fixed refresh rate and a configured delay, we can fix the size of the queue in total delay
frames required as the refresh rate multiplied by the desired delay. Then, we allow dequeue only
when all frames are filled, and we allow enqueue only when not all frames are filled. Alongside
that, the number of frames between the front and back gives the total delay currently accrued.

As long as we know that we can always process and display an individual frame faster than the
refresh[^fn:7], we know that the delay is consistent as per
goal one. Similarly, by only allowing enqueue when the delay is not satisfied, we know that at any
given time the frame at the back of the queue is a snapshot in time exactly as long ago as the delay
is configured, regardless of whether there were received frames between the last "snapshot" and the
most recent one, satisfying goal two. The "fixed" queue size is using the configured delay, so
regardless of the current choice we can recalculate it as the configuration changes, satisfying goal
three. As you will see, the implementation is thread safe, and as noted above we can easily
calculate the current status of the delay, knocking out goals four and five.

This leaves us with goal six, and there's an option here to feed two birds with one scone[^fn:8]. See, one problem with this proposed solution is a
sort of "hitching" problem -- there's this back-and-forth dance where the queue is full, the
telemetry frame at the front is dequeued and shown, and before a new frame is received the interface
tries and now fails to dequeue the next frame. To get around this, we can simply add some "slop" to
the end of the queue, some relatively small number of frames with which we permit the queue to _just
barely_ over/undershoot. This way, the interface has a handful of delay-satisfactory frames to chew
on while waiting for new telemetry to come in, and the telemetry receiver thread has some padding to
flex in case a burst of frames come in.

Now we feed the second bird: when we want to go "live", we drop the queue down to just the "slop"
frames. With a refresh rate of 10 Hz _(that's 10 telemetry frames every second)_ it would take ten
slop frames to reach a delay of one second. So, as long as we keep the slop frames to at most around
five, we can keep the "live" delay down to less than half a second. This does reintroduce the
hitching problem, but with a truly live feed the hitching is unavoidable as again the telemetry is
coming in at a varying rate. As far as I have observed in testing, this solution is perfectly
satisfactory for a live display.

That's all six goals -- on to implementation.


## Implementation {#implementation}

_Note: This will reflect what is live in the repo at the time of writing, but I do not guarantee that it will remain
that way._


### The `DelayInfo` dataclass {#the-delayinfo-dataclass}

Whenever either `delay_s` _(delay in seconds)_ or `refresh_hz` _(refresh rate in hz)_, two calculated parameters change:

1.  `total_frames` : The calculated number of frames in the queue
2.  `frame_period` : The amount of time represented by one frame

Given how tightly bound these four parameters are, and how frequently they're used together throughout the
implementation, it's important for thread safety that we prohibit the use of one whilst any other is changing. This is
all `DelayInfo` exists to do.

Omitting some not-so-relevant details, we lay out the class as:

```cpp
class DelayInfo {
  static constexpr size_t DELAY_SLOP = 4;

private:
  size_t _delay_s = 0;
  size_t _refresh_hz = 10;

  std::shared_mutex _mtx;

  void recalculate_params();

public:
  /** @brief Instantiate the delayinfo with the configured parameters. */
  DelayInfo(size_t delay_s, size_t refresh_hz)
    : _delay_s(delay_s), _refresh_hz(refresh_hz) {
    recalculate_params();
  }

  void set_delay_s(size_t const);
  void set_refresh_hz(size_t const);

  [[nodiscard]] size_t get_delay_s();
  [[nodiscard]] size_t get_refresh_hz();
  [[nodiscard]] size_t get_delay_frames();
  [[nodiscard]] Time::Duration::DblMilliSec get_frame_period();
};
```

The most important detail here is the `shared_mutex`. Since `recalculate_params` will need to change
multiple paramaters, we need to guard against any getters/setters accessing any parameters during
recalculation. But, since the getters don't need to change any state and thus don't need to block
other getters, we simply take a `unique_lock` of `_mtx` in the setters and `recalculate_params`, and take
a `shared_lock` in the getters. Given how relatively infrequently the delay configuration should be
changing, `_mtx` should rarely be contested.

We'll skip going over the implementation of the getters and setters, as `recalculate_params` should
give a good idea:

```cpp
void recalculate() {
  // Called by any setters
  std::unique_lock lock(_mtx);

  _total_frames = (_delay_s * _refresh_hz) + DELAY_SLOP;
  _frame_period = Time::Duration::DblMilliSec(
		    1000.0 / static_cast<double>(_refresh_hz));
}
```

Here, we:

-   Take the `unique_lock` as mentioned before
-   Calculate the number of frames needed as the delay multiplied by the refresh rate, adding the slop
    frames
-   Calculate the frame period _(total amount of time represented by one frame, in milliseconds)_ as the
    reciprocal of the refresh rate


### `TelemetryQueue` {#telemetryqueue}

Rather than just walk through the class definition, we'll approach it from a more bottom-up
direction. To start with, we'll go over some background on the queue and then the most important two
functions of this _(or any)_ queue: enqueueing and dequeueing. Following that, we'll close out by
covering the queue reconfiguration process.


#### Queue background {#queue-background}

Since the number of the frames in the queue at any time is essentially fixed-size, I opted to reach
for a single-ended [circular queue](https://en.wikipedia.org/wiki/Circular_buffer). With this approach, we simply track monotonically increasing
enqueue and dequeue indices, adjusting them only when we take in or give out a telemetry
frame. Then, to access the queue, the actual position is calculated as the index reduced modulo the
queue size.

As highlighted earlier, a primary goal was to be able to split the "producer" and "consumer" logic
between different threads; one to receive telemetry, the other to process and display it. With this
design, we just need two small additions to get there:

-   A mutex on each individual frame, so that we don't try to access the same frame in both threads
    _(possible if the queue is either empty or full)_
-   A mutex on the entire queue, so that we can lock all frames during resize _(more on this shortly)_

One final piece to note is my error handling strategy. Whenever possible, I prefer to reach for
"errors as values" as opposed to exceptions, and I do this for two primary reasons:

1.  Forcing the caller to acknowledge the existence of and do something with errors makes code more
    readable at the callsite _(in my opinion)_.
2.  Requiring erroneous paths to return meaningful information forces me to more carefully consider
    what should be an error and often leads me to write more thorough code with stronger guarantees.

This approach is what I chose for most of the error handling in this project. Here, I have an error
type `TelemetryQueue::Err` which has various states representing errors we might expect to see. Here,
the two most relevant states are `Err::TOO_RECENT` and `Err::DELAY_FULL`, noting respectively that there
is currently not enough accrued delay to dequeue a frame or that there is currently exactly enough
accrued delay and to enqueue a new frame would cause the delay to overshoot.

With that, we'll now walk through the implementation.


#### Dequeue {#dequeue}

To start with, we see the aforementioned errors-as-values approach. Here, we leverage C++23's
wonderful [std::expected](https://en.cppreference.com/cpp/utility/expected), where we note that we'll return either a `unique_ptr` representing a
telemetry frame, or an `Err` representing the error encountered. Also, since we are handling errors
using `std::expected`, we will not throw any exceptions and anything we call will not either. Thus, we
can mark the method as `noexcept`.

```cpp
std::expected<std::unique_ptr<TelemetryFrame>, TelemetryQueue::Err>
TelemetryQueue::dequeue() noexcept {
```

First, we double check that the dequeue index is not ahead of the enqueue index, as this is
considered "impossible" -- if this has happened, we are in an unrecoverrable state. As such, we
check this with an assertion[^fn:9]:

```cpp
assert(_deq_idx <= _enq_idx);
```

Next we check that the delay is satisfied. The call to `delay_info`'s `get_min_delay_frames()` just
retrieves the minimum number of frames between the enqueue and dequeue indices such that the delay
is satisfied, which is just the total number of frames before the addition of the slop frames. If
the delay is satisified, we continue on, and if not, we log a debug message and return the
corresponding error state

```cpp
if ((_enq_idx - _deq_idx) >= _delay_info.get_min_delay_frames()) {
  // ... what follows goes here ...
} else {
  Tools::Log::Debug("Dequeue skipped, delay is not ready", "TELEM-QUEUE");
  return std::unexpected(Err{Err::TOO_RECENT});
}
```

Continuing assuming the delay is satisfied, we get a `shared_lock` on the full queue and then again
check against an "impossible" state to ensure the queue is not empty. The mutex on the full queue
will only be contested if we try to resize the queue, a relatively rare occurence. Every step
following assumes that the queue has not been resized, so we acquire the `shared_lock` _(using a [RAII](https://en.cppreference.com/cpp/language/raii)
wrapper, so `.lock()` is called at construction and `.unlock()` at destruction)_ at the start and do not
release until the `true` body exits. Since this is a `shared_lock`, both enqueue and dequeue can acquire
it simultaneously.

```cpp
std::shared_lock full_lock(_frame_mtx);
assert(!_frames.empty());
```

Next we index the full queue at the dequeue index reduced modulo the queue size, and acquire a lock
on its mutex _(this is again a RAII wrapper, just not shared lock this time)_.

```cpp
auto &cur_frame = _frames.at(_deq_idx % _frames.size());

std::scoped_lock single_lock(cur_frame.mtx);
```

Now we check if the frame retrieved is actually present. Given that we return frames as a
`std::unique_ptr<TelemetryFrame>`, when we give them to the caller we are moving[^fn:10] them out of the queue. Then,
since `std::unique_ptr`'s move constructor sets the moved-from ptr to `nullptr`, a frame which is not
yet populated will be `nullptr`. As such, if the frame we retrieve is `nullptr`, it means that we have
somehow advanced to a frame which is not ready, so the delay is not yet satisied. This should
generally not happen, but it is good practice to check pointers regardless.

The use of `std::unique_ptr` for handing around the frames is a vestige of prior unrelated troubleshooting
regarding the `TelemetryFrame`'s move constructor. At this point it's not necessary, and is probably a
target for future refactor/cleanup. With that said, the indirection cost of a pointer when accessing
the frame is negligible here, so to change it would be only for cleanliness _(still a very worthy cause)_.

```cpp
if (cur_frame.frame == nullptr)
  return std::unexpected(Err{Err::TOO_RECENT});
```

Finally, we know that we have a valid frame. As such, we'll step the dequeue index forward one frame
and then return the frame we retrieved before.

```cpp
_deq_idx++;

return std::move(cur_frame.frame);
```

Now for the full dequeue logic:

```cpp
std::expected<std::unique_ptr<TelemetryFrame>, TelemetryQueue::Err>
TelemetryQueue::dequeue() noexcept {
  assert(_deq_idx <= _enq_idx);
  if ((_enq_idx - _deq_idx) >= _delay_info.get_min_delay_frames()) {
    std::shared_lock full_lock(_frame_mtx);

    assert(!_frames.empty());

    auto &cur_frame = _frames.at(_deq_idx % _frames.size());

    std::scoped_lock single_lock(cur_frame.mtx);

    if (cur_frame.frame == nullptr)
      return std::unexpected(Err{Err::TOO_RECENT});

    _deq_idx++;

    return std::move(cur_frame.frame);
  } else {
    Tools::Log::Debug("Dequeue skipped, delay is not ready", "TELEM-QUEUE");
    return std::unexpected(Err{Err::TOO_RECENT});
  }
}
```


#### Enqueue {#enqueue}

Since the enqueue logic mirrors that of dequeue, we'll start with the full method and dissect where
they differ.

```cpp
std::expected<void, TelemetryQueue::Err>
TelemetryQueue::enqueue(std::string_view const payload) noexcept {
  assert(!payload.empty());
  assert(_deq_idx <= _enq_idx);

  if ((_enq_idx - _deq_idx) < _delay_info.get_delay_frames()) {
    std::shared_lock full_lock(_frame_mtx);

    assert(!_frames.empty());
    auto &cur_frame = _frames.at(_enq_idx % _frames.size());
    std::scoped_lock single_lock(cur_frame.mtx);

    cur_frame.frame =
        std::make_unique<TelemetryFrame>(Tools::b64_decode(payload));

    _enq_idx++;

    return cur_frame.frame->is_valid()
               ? std::expected<void, TelemetryQueue::Err>{}
               : std::unexpected(Err{Err::FRAME_INVALID});
  } else {
    Tools::Log::Debug("Enqueue skipped, delay is full", "TELEM-QUEUE");
    return std::unexpected(Err{Err::DELAY_FULL});
  }
}
```

Firstly, here we take in a [string_view](https://en.cppreference.com/cpp/string/basic_string_view) of the received base64-encoded telemetry payload. We only need
to read the string we take in, and that string's lifetime will extend beyond this called function,
so we use a `string_view` and save on an unnecessary copy. We'll also double check that payload is
non-empty with an assertion, as this should not be possible.

The second difference is in the delay satisfaction check -- this is the negation of the one in
`dequeue`, thus we only permit enqueueing a new frame if there are fewer frames than the delay
requires between the front and back.

Next, rather than retrieve a frame we must create one. We do this by first getting a reference
to the current slot in the queue, and then construct a frame by passing in the base64-decoded payload.

The final difference is in the return: if we succeed, we don't need to return anything, and if we
don't we need to tell the caller why. Hence, once we've built the frame and set the current slot in
the queue accordingly, we need to check if the frame is in a valid state. If it is, we return
nothing, and if not, we return the previously mentioned `Err` with the corresponding state.


#### Queue reconfiguration {#queue-reconfiguration}

Here is where we'll handle rebuilding the delay upon delay reconfiguration. If the delay is
decreased, we need to shrink the queue size, and likewise grow it if the delay is increased.

It would be a very frustrating user experience if the full delay had to be re-accrued upon every
delay change, so we need to preserve whatever amount of the delay is relevant when rebuilding
the delay -- we will come back to this shortly.

As mentioned before, delay reconfiguration is the only time the full queue's mutex needs to block
enqueue/dequeue _(and enqueue/dequeue needs to block reconfiguration)_, so we'll start by taking a
`unique_lock` on the full queue mutex and a `scoped_lock` on the `DelayInfo`'s mutex.

```cpp
void TelemetryQueue::rebuild_delay() noexcept {
  std::unique_lock frame_lock(_frame_mtx);

  std::scoped_lock lock(_delay_info_mtx);
```

Next we calculate the new queue size, and then how many frames forward we need to skip.
If the delay is increasing, the frame at the front of the queue _(next to be dequeued)_ is now too
recent, and thus no skip is necessary. If the delay is decreasing, the front of the queue is
now too old, requiring us to skip some number of frames ahead.

```cpp
size_t prev_delay_frames = _frames.size();
size_t new_delay_frames =
    _delay_info.get_delay_frames();

size_t skip_frames = (new_delay_frames < prev_delay_frames)
                         ? ((prev_delay_frames - new_delay_frames) - 1)
                         : 0;
```

Since C++'s `std::mutex` is neither copyable nor movable, and the queue contains a wrapped version of
`TelemetryFrame` adding a mutex, the `std::vector` which underlies the queue is also neither copyable
nor movable. Hence, we cannot simply extend the existing vector -- we instead create a new one and
move preserved frames over.

So, first we create the new vector with the new length:

```cpp
std::vector<LockedFrame> new_frames(new_delay_frames);
```

Then, if this is the initial configuration of the queue, there are no frames to preserve, and we can
just set our vector and return early:

```cpp
if (_enq_idx == 0) {
  _frames = std::move(new_frames);
  return;
}
```

Next we move the dequeue index forward by the number of skip frames calculated before. When the
delay is rapidly increased and decreased, it is sometimes possible to end up with the skip frames
pushing the dequeue index ahead of the enqueue index. Since the dequeue index should always be less
than or equal to the enqueue index, we'll set it to the minimum of the two:

```cpp
_deq_idx += skip_frames;
_deq_idx.store(std::min(_deq_idx, _enq_idx));
```

Now we'll copy over the frames we need to preserve, if any. Starting at the dequeue index, we'll
simply move each frame over until we reach the enqueue index:

```cpp
if (prev_delay_frames > 0) {
  for (size_t copy_idx = _deq_idx; copy_idx < _enq_idx; ++copy_idx) {
    new_frames.at(copy_idx % new_delay_frames).frame =
        std::move(_frames.at(copy_idx % prev_delay_frames).frame);
  }
}
```

Finally, we move the new vector into `_frames` and are done:

```cpp
_frames = std::move(new_frames);
```

In total, for queue reconfiguration we have:

```cpp
void TelemetryQueue::rebuild_delay() noexcept {
  std::unique_lock frame_lock(_frame_mtx);

  std::scoped_lock lock(_delay_info_mtx);

  size_t prev_delay_frames = _frames.size();
  size_t new_delay_frames =
      _delay_info.get_delay_frames().value_or(DelayInfo::DELAY_SLOP);

  size_t skip_frames = (new_delay_frames < prev_delay_frames)
                           ? ((prev_delay_frames - new_delay_frames) - 1)
                           : 0;

  std::vector<LockedFrame> new_frames(new_delay_frames);

  if (_enq_idx == 0) {
    _frames = std::move(new_frames);
    return;
  }

  _deq_idx += skip_frames;
  _deq_idx.store(std::min(_deq_idx, _enq_idx));

  if (prev_delay_frames > 0) {
    for (size_t copy_idx = _deq_idx; copy_idx < _enq_idx; ++copy_idx) {
      new_frames.at(copy_idx % new_delay_frames).frame =
          std::move(_frames.at(copy_idx % prev_delay_frames).frame);
    }
  }

  _frames = std::move(new_frames);
}
```


## Conclusion {#conclusion}

There is much, much more to talk about regarding this project; implementing a delay was merely the
main motivating factor in me starting it. The design I originally pictured and what is live on
[GitHub](https://github.com/schelcc/indy-tui) certainly differ, but not by _that_ much. Even so, there were countless different iterations at
what you see here, all of which will forever remain as commits quickly undone.

I started with something resembling what you see here, albeit significantly less "developed" and
explainable. With many tweaks and corrections, I was eventually able to get that initial design to
_(mostly)_ work. But, as I'm sure many can relate to, I convinced myself that wasn't the "right" way
to tackle this problem. Somewhere along the way I had convinced myself that I was "cheating", that
this was a hacky solution -- I had myself convinced that if I wanted to do it "right" I needed to
implement the aforementioned [dynamic queue](#dynamic-queue). I sunk a not insignificant amount of time into the
dynamic solution, got it to the point where it worked with my crude prototype of a display, and
excitedly got to testing it.

It did not work.

The delay was inconsistent, it wasn't always clear whether the delay was even building
correctly. Every time I'd try to use it, I would find a different way to trick the delay into
getting out of sync. To be sure, a fair amount of these problems originated between the keyboard and
chair, but some of the failure modes I encountered were remarkably reminiscent of the ways I've seen
other similar tools fail.

Whether any subset of the issues I noted using the dynamic solution were inherent to that solution or
not, I had to make a call: stick with what I had convinced myself to be the "right" solution which
I could not seem to make work, or go back to what I had come up with on my first pass. Put more
bluntly, I needed to decide whether to get in line with what I had convinced myself "everyone else"
was doing, or to trust my design intuition.

As now explained, I went with the latter. After much consideration, I'd decided that I could
more effectively defend the solution seen here than the dynamic solution I had laid out. I
went back and once again implemented yet another telemetry queue, now taking a little more time to
make sure every decision was grounded in a set of end goals. What resulted was definitely rough
around the edges and took some effort to smooth it out -- that's unavoidable. But, with that effort,
I eventually got to a point where I finally _couldn't_ trick the delay into failing. Surely, there are
latent issues which will show themselves in due time, but what I have now is something that meets
the goals I laid out, built following a design process I executed.

Once again: this is _a_ solution, certainly not _the_ solution. Hopefully, it's now clear why I've picked
this one.

[^fn:1]: yeah this was
    lowkey track limits but it's still awesome
[^fn:2]: it's me I am said viewer
[^fn:3]: this is a really cool
    explanation of this delay problem and why it's not going anywhere <https://www.youtube.com/watch?v=CgcXli8NxHw>
[^fn:4]: [multiviewer](https://multiviewer.app) is an excellent tool and works across many series
[^fn:5]: Also, your observed broadcast delay fluctuates over time --
    if you want calibrate your delay to less than 0.5s/500ms you're likely going to have to spend a lot of time
    monitoring and readjusting it. I personally would like to just set my telemetry delay to be _just_ longer than my
    broadcast delay and accept that what I see on my leaderboard is just barely behind what I see on TV.
[^fn:6]: telemetry is sent rather quickly, with each frame representing at most slightly less than a second
[^fn:7]: and, we can -- we pick the refresh rate
[^fn:8]: this
    has, for better or worse, worked its way into my lexicon:
    <https://x.com/peta/status/1070066047414345729>
[^fn:9]: I cannot wait to rewrite this and make this a
    [pre and post condition](https://en.cppreference.com/cpp/language/contracts) once C++26 is mature
[^fn:10]: here is a decent
    explainer on those new to C++'s move semantics:
    <https://stackoverflow.com/questions/3106110/what-is-move-semantics>
