+++
title = "C++ Thread Safety, Reinventing The Wheel, and Making My Life Easier"
author = ["Cole"]
lastmod = 2026-09-23T18:33:53-04:00
tags = ["cpp", "misc-design-notes"]
draft = false
katex = false
socialShare = true
+++

Alternative title: [Carcinization](https://en.wikipedia.org/wiki/Carcinisation)? In my C++ codebase? It's more likely than you think.

Whilst the job search continues, I thought I'd get out a quicker writeup about a lifesaver of a tool
I've recently implemented in my ongoing project `indy-tui`[^fn:1] and how I ~~stumbled~~ iterated my way to it.

<!--more-->


## Background {#background}

By its nature, `indy-tui` requires keeping several threads up and running -- when managing an
interface, user input, and a live websocket session, one's hand is slightly forced. Likewise,
some amount of shared data across threads is largely unavoidable. Hence, it's time for locking.

This project has been my first "real" go at an actually multithreaded application; a desire to gain
some experience with multithreading, both generally and with C++'s Standard Library, was a strong
factor in my motivation to take it on.[^fn:2] As such, the git history demonstrates me walking across the
learning curve -- first a lot of `std::mutex` and `std::scoped_lock`, followed by my own ill-fated RAII
wrapper to add logging to lock acquisition[^fn:3], a handful of `std::atomic` misunderstandings, only
then finally landing on `std::shared_mutex`.

Thankfully, almost every instance of a need for shared data in this project[^fn:4] is a "one writer, many readers" situation,
so `std::shared_mutex` is really the best option. Until recently, I was satisfied giving each of my
classes a shared mutex _(or multiple to give load bearing components their own)_ and keeping access to
them and their related data tightly behind private getters and setters. This was largely _fine_, it
didn't feel the best but it seemed simple.

It seemed simple.


### I like dataclasses {#i-like-dataclasses}

For better or worse, I quite like dataclasses. I like more semantic code what can I say.

For a UI rework inching towards some adaptation[^fn:5] of MVC, I needed to pull out
the UI logic I had put inside the leaderboard. That work was in there to begin with because the
leaderboard needed to access the vector of drivers, as well as a collection of event information
_(track name, laps to go, current flag, etc.)_, and with these being behind several shared mutexes the
UI logic needed the ability to get locks. Why have one responsibility when you could have more?

I like dataclasses, so the obvious solution to me was to pull the event information out into some
`EventInfo` struct, with fields for each entry, and then have the leaderboard hold around an instance
of it:

```cpp
struct EventInfo {
  // We aren't guaranteed to get the data for these fields all at the
  // same time, so we populate them as we get them, hence the
  // optionals
  std::optional<std::string> event_name;
  std::optional<std::string> track_name;
  std::optional<int> laps_to_go;
  // ...
};
```

Fine enough, but this is shared data -- gotta have those mutexes. But, what gets a mutex? And who is
responsible for it? Some of these fields are updated much more frequently than others, and most of
them are read each render loop; if the full struct gets one mutex, we could end up with a lot of
contention. Conversely, if we want to give each field its own, that's both a lot of annoying
boilerplate _and_ making the usage more annoying as I'd have to litter the codebase with
mindfully-scoped lock wrappers _(or do manual locking and risk forgetting an unlock)_ -- this would
look something like:

```cpp
struct EventInfo {
  std::shared_mutex event_name_mtx;
  std::optional<std::string> event_name;

  std::shared_mutex track_name_mtx;
  std::optional<std::string> track_name;

  // ...
};

/* ... */
// Use somewhere in the codebase, say to check if we have the event name yet

// Open a tighter scope so that the lock doesn't live longer than it needs to
{
  std::shared_lock event_name_lock(event_info.event_name_mtx);
  if (event_name.has_value()) {
    // ...
    // event_name_lock is alive for all of whatever's in here
  } else {
    // ...
    // and all of whatever's in here too
  }
}
```

That's not very fun. It's even less fun when you need multiple locks at the same time.

This was the point where I had justified a tangent of trying to find a remotely ergonomic design. In
an ideal world, I wanted a type _(or macro_)[^fn:6] which could wrap any type and:

1.  Add a shared mutex
2.  Handle getting and setting in a more ergonmic way than manually acquiring
    `std::shared_lock` and/or `std::unique_lock`
3.  If at all possible, do (2) in such a way that trying to modify the underlying object when taking a
    shared lock becomes a compilation error, as well as attempting to get a unique lock on a
    const reference to the wrapped object
4.  When necessary, I can still hold a lock on the object for an arbitrary amount of time _(e.g. need
    to work on several objects and need all locked to do so)_


## The triangular wheel {#the-triangular-wheel}

First is the solution that truly missed the mark.

For whatever reason I was quite stuck in OOP brain when working on this[^fn:7], so immediately what came to mind was a
wrapper which simply defined a getter and setter, taking a shared lock and unique lock respectively:

```cpp
// Omitting constructors and whatnot
template <typename T>
struct LockWrapper {
  T get() const {
    std::shared_lock lock(mtx);
    return obj;
  }

  void set(T t) {
    std::unique_lock lock(mtx);
    // In reality you'd want to take the input as a forwarding
    // reference and std::forward it here but there are more pressing
    // matters
    obj = t;
  }

private:
  T obj;

  // By qualifying this as mutable we can take a shared_lock in a
  // const-qualified method
  mutable std::shared_mutex mtx;
};
```

A couple problems here.

A glaring issue is the signature of `get()` -- we incur a copy every single time we call `get()`! This
is quite bad. Even worse, if `T` is not copyable it's fully incompatible with this wrapper.

Another issue is what happens when you want to first assign to the wrapped value and then use it?
You might say to return a reference to the wrapper in set:

```cpp
LockWrapper& set(T t) {
  std::unique_lock lock(mtx);
  obj = t;
  return *this;
}
```

And then you might use the two methods together:

```cpp
// Let some_field be a LockWrapper of some default constructible and
// assignable type T
some_field.set(T()).get();
```

This "works", but there is a suspiciously elephant-shaped object in the corner that is begging to be
acknowledged: we release the unique_lock and then some time later take the shared_lock. In other
words, _we are not guaranteed that `some_field` has not changed again between `set()` and `get()`_.[^fn:8]

Regardless of if I'm wrong and there is somehow a guarantee that the two calls will always directly
follow each other, I still threw this solution out -- the forced copying alone is a nonstarter, and
this doesn't allow me to actually hold onto the lock for longer if needed.


## The hexagonal wheel {#the-hexagonal-wheel}

After the above failure, another component of the "right" solution became clear: the lock itself has
to be handed off alongside the data. To take advantage of the whole RAII wrapper thing, the lock
needed to be given to the caller and stored in some way. That is, the lock must be an [lvalue](https://en.cppreference.com/cpp/language/value_category) at the
callsite. Also, we should be sure to _not_ copy literally everything -- we should be returning
references to the wrapped object.

Fortunately, someone working on the standards saw this coming -- `std::shared_lock` and
`std::unique_lock` are movable. Hence, we just need to adjust the idea behind the getter and setter
before to instead get locks. In comes the trusty `std::pair` and [structured bindings](https://en.cppreference.com/cpp/language/structured_binding)[^fn:9]:

```cpp
// shared and unique lock are templated on the mutex type -- they were
// inferred before, but won't be now, so we'll just add a defaulted
// template parameter to fill them in
template <typename T, typename Mut = std::shared_mutex>
struct LockWrapper {
  // Again omitting constructors

  std::pair<std::shared_lock<Mut>, T const&> get_const() const {
    return {std::shared_lock<Mut>(mtx), obj};
  }

  std::pair<std::unique_lock<Mut>, T&> get_mut() {
    return {std::unique_lock<Mut>(mtx), obj};
  }

private:
  T obj;
  Mut mtx;
};
```

Now, using structured bindings we can easily get both the lock and the object:

```cpp
LockedWrapper<std::string> s = "foo";

// We can append to the string
{
  auto [lock, ref_s] = s.get_mut();
  ref_s.append("bar");
}

// And we can print the string
{
  auto [lock, ref_s] = s.get_const();
  std::println("{}", ref_s);
}
```

This feels _much_ better than before -- we aren't copying and we can hold the lock as long as we see
fit. But, this has a bit of a semantic downside: if we want to use the wrapped object directly in an
expression _(not break it out into a statement beforehand, and add a scope to shorten the
lifetime)_ we have to call the annoying `.second` member of the pair:

```cpp
LockedWrapper<std::string> s = "foo";

s.get_mut().second.append("bar");

std::println("{}", ref_s.get_const().second);
```

This isn't the end of the world, but it is a bit worse than I would've liked.


## I dream of better (already existing) wheels {#i-dream-of-better--already-existing--wheels}

At this point what I had in my head as the dream solution almost mirrored Python's decorators: I
wanted to be able to somehow "wrap" an entire type to lock before and after like you could with a
decorated function, but have that type behave almost exactly like the wrapped type -- I wanted the
lock to be invisible.

Now is when I reached out to a friend and much better programmer[^fn:10] to see what they thought; whether this was even a
decent pattern, or if I was simply losing it. Turns out it was a bit of both -- this was
[carcinization](https://en.wikipedia.org/wiki/Carcinisation) manifest.


## The... crab wheel? {#the-dot-dot-dot-crab-wheel}

So, turns out, I was just inching ever closer to exactly how Rust handles this problem. Whoops.

In Rust's `std::sync` they provide `MutexGuard<T>` -- a RAII wrapper which locks an object protected by
a mutex, but acts just as `&T` when dereferenced, and releases the lock when falling out of
scope. This is exactly what I've wanted all along, so it's time for the final wheel.

To mimic this, we need two new types:

1.  One which wraps a (possibly const) reference to the wrapped type with a lock, but largely acts
    like the wrapped type
2.  One which wraps a type with a shared mutex and spits out (1)

We'll call these `LockPair` and `Locked` respectively.


### `LockPair<T>` {#lockpair-t}

Translating (1) to C++ land for `LockPair<T>`, we need to define `operator*()` and `operator->()` for some
templated struct which has the templated type and a lock. When an instance of this struct is
dereferenced, we want to return a _(possibly const)_ reference to the wrapped object -- this is our
`operator*()`. The specifics of `operator->()` are a bit of a tangent[^fn:11], but here we can get away
with just returning a _(possibly const)_ pointer[^fn:12] to the wrapped
object.

The other half of the `LockPair<T>` is the lock itself. With `std::unique_lock` and `std::shared_lock`,
the RAII functionality is such that when the object that _currently_ owns the lock goes out of scope,
the lock is released. Thus, we have to be sure to own the lock for this to work, and at construction
we'll want to move it in. A neat result of the RAII usage is that we don't actually have to interact
with the lock _at all_, we just need to hold onto it. Regardless of whether we have a unique or shared
lock, any information we need to generate the appropriate methods is contained in the templated type
-- we can just shove the lock in a variant of the two lock types and move on.[^fn:13]

The final constraint on `LockPair<T>` to note is that we template around the wrapped type, not a
reference to the wrapped type, so we'll just add a requirement on the template that `T` is not a
reference.

This gives us `LockPair<T>`:

```cpp
template <typename T, typename ...Ts>
concept IsOneOf = (std::is_same_v<T, Ts> || ...);
// This is just a requirement that the input templated type is present
// in a list of types. If you're not familiar with C++'s concepts,
// just read "IsOneOf<...> auto" below as "any type in this given list
// of types"

template <typename T, typename Mut = std::shared_mutex>
  requires(!std::is_reference_v<T>)
struct LockPair {
private:
  std::variant<std::shared_lock<Mut>, std::unique_lock<Mut>> lock;
  std::reference_wrapper<T> val;

public:
  // Technically we do end up taking the lock as a forwarded reference
  // here, but the locks are only movable so we don't need to worry
  // about copying.
  template <typename S = std::remove_cv_t<T>>
  LockPair(IsOneOf<std::shared_lock<Mut>, std::unique_lock<Mut>> auto &&l,
           S &&s)
      : lock(std::move(l)), val(std::reference_wrapper<T>(std::forward<S>(s))) {
  }

  // Return a (possibly const) pointer to T
  T *operator->() { return &(val.get()); }

  // Return a (possibly const) ref to T
  T &operator*() { return val.get(); }
};
```


### `Locked<T>` {#locked-t}

`Locked<T>` has one job: control access to the contained object. Where `LockPair<T>` acts as a way to
get to the object, `Locked<T>` acts as a way to get to a `LockedPair<T>`. Thus, we need to accomplish
three main items:

1.  Hold `T`
2.  Define a way to retrieve a mutable reference to the object
3.  Define a way to retrieve a constant reference to the object

For (1) we just need to concern ourselves with construction. If `T` is default-constructible, we want
`Locked<T>` to be as well, so we'll define that and let it be deleted whenever `T` is not
default-constructible. In that case, we'll have `Locked<T>` be initialized with some `T`, and once again
we'll see the forwarding reference so that we can both copy and move into `Locked`. Those are the
options for constructing `Locked<T>` -- once we have `Locked<T>` we only allow access to the wrapped
object through the `LockPair<T>` interfaces. That is to say, there is no way to get to the
contained object directly through `Locked<T>`.

For (2) and (3), we'll define `get_mut() -> LockPair<T>` and `get_const() -> LockPair<T const>` which
simply return `{std::unique_lock(_mtx), _val}` and `{std::shared_lock(_mtx), _val}` respectively,
passing them to `LockPair<T>`'s relevant constructor. Once again, we instruct `LockPair<T>` on what
kind of methods to generate _(whether the accessors are const or not)_ based on the template
parameter `T`, not the lock we give it. One special case we'll want to handle is when `Locked<T>`
itself is accessed through a const reference, say by accessing a class's member `Locked<T>` in the
body of a const-qualified method. In this case, we'll want to make attempting to get mutable
access a compilation error -- we'll just delete `get_mut()` for the const-qualified overload.

One final note on `Locked<T>` is that allowing `T` to be a pointer doesn't really make sense, at least
as far as I can imagine. For my use I don't want to permit it to hold a pointer, so I'll add a
constraint to the template requiring this. There's not a named constraint in `<type_traits>` for smart
pointers, and `std::is_pointer<T>` does not evaluate `true` for them, so I have `Locked<T>` specialized
for `std::shared_ptr` and `std::unique_ptr` with all methods deleted.

Thus, we have `Locked<T>`:

```cpp
template <typename T>
  requires(!std::is_pointer_v<T>)
class Locked {
  mutable std::shared_mutex _mtx;
  T _val;

public:
  // Permit a default ctor only if T has one
  Locked()
    requires(std::is_default_constructible_v<T>)
      : _val() {}

  template <typename U = std::remove_cv_t<T>>
  Locked(U &&u) : _val(std::forward<U>(u)) {};

  LockPair<T> get_mut() { return {std::unique_lock(_mtx), _val}; }
  LockPair<T> get_mut() const = delete;

  LockPair<T const> get_const() const { return {std::shared_lock(_mtx), _val}; }
};

/* Render smart pointer instantiations unusable */

template <typename T> class Locked<std::shared_ptr<T>> {
  Locked() = delete;

  template <typename U = std::remove_cv_t<T>> Locked(U &&u) = delete;

  LockPair<T> get_mut() = delete;
  LockPair<T> get_mut() const = delete;
  LockPair<T const> get_const() const = delete;
};

template <typename T> class Locked<std::unique_ptr<T>> {
  Locked() = delete;

  template <typename U = std::remove_cv_t<T>> Locked(U &&u) = delete;

  LockPair<T> get_mut() = delete;
  LockPair<T> get_mut() const = delete;
  LockPair<T const> get_const() const = delete;
};
```


## I promise it rolls {#i-promise-it-rolls}

To wrap this up, I'll just give two quick examples of this in use; one as part of an expression
where we don't want a longer living lock, and the other where we do.

For the first, I'll pull out a piece of `indy-tui`'s UI where I display some information about the on
track event. I want to show how much of the current event is left to go -- if the current on track
session is a practice or qualifying session, all that matters is the time left in the
event. Conversely, if it's a race, we should show the number of laps completed alongside the total number
of laps. To accomplish this, I need to:

1.  Determine the current session type, stored as a `Locked<std::optional<Telemetry::SessionType>>`
2.  If the session type is a race:
    -   Retrieve the current laps completed, giving us `std::optional<int>`
    -   If the `int` is present, convert it to a string, otherwise do nothing
    -   Retrieve the total laps, giving us `std::optional<int>`
    -   Again convert to a string if present, otherwise do nothing
    -   Update the row as `"Lap"` followed by `"{} / {}"` formatted with the strings resulting from the
        above steps or `"--"` if not present
3.  If the session type is a practice or qualifying session:
    -   Retrieve the time left in the session, stored as `Locked<std::optional<std::string>>`
    -   Update the row as `"Time left"` followed by the time left if present, otherwise `"--:--:--"`

To do this before would've required manually taking a shared lock of `completed_laps` and `total_laps`,
possibly requiring two extra scopes if I wanted to avoid locking both at the same time[^fn:14]. The `Table` UI element
that's in use here has the ability to update a full row or column with a vector, and the
`std::optional<int>` fields give us the ability to use the mapping methods `.transform()` and
`.and_then()` to do something with the (possibly) contained value and return another
`std::optional<T>`. Thus, if we had the ability to safely and cleanly access these fields as part of
an expression, we could keep this row update concise. Thankfully, I have my trusty new _(slightly
oxidized_) wheel:

```cpp
if (sess.session_type.get_const()->value_or(
    SessionType::PRACTICE) == SessionType::RACE)
  res = table.update_row(
    1, {"Lap", std::format(
                    "{} / {}",
                    sess.completed_laps.get_const()
                        ->transform([](auto i) { return std::to_string(i); })
                        .value_or("--"),
                    sess.total_laps.get_const()
                        ->transform([](auto i) { return std::to_string(i); })
                        .value_or("--"))});
 else
  res = table.update_row(
    1, {"Time left", sess.time_to_go.get_const()->value_or("--:--:--")});
```

For that second example, I'll pick out a smaller component[^fn:15]. In the logic to display the live leaderboard, I have a table of possibly changing size
dependent on two things -- the number of drivers, and the number of columns desired[^fn:16]. Thus, when I want to update one of these values, I
need to do some reconfiguration afterwards.

For some reason, I decided to have setters for the number of drivers and the columns individually,
pulling the reconfiguration out to another method which both call. Multiple operations are done
assuming all properties are unchanged throughout the reconfiguration, so all will need to be locked
in some way. These don't change much, so it wasn't too much of an issue to grab a unique lock on
each element _(the table, the vector of columns, and the number of drivers_). As discussed earlier
though, I needed to be sure we're not releasing the lock on the newly changed component -- I decided
to move the retrieved `LockPair` into the reconfiguration call. For the other components, I didn't
want the reconfiguration method to have to know what locks it needs to take based on the caller, so
I decided to simply have the reconfiguration method move in a `LockPair<T>` for each of the three
components. This way, the caller is the arbiter of which locks are taken when, moving them all into
the reconfiguration at the end regardless.

For updating the columns and the subsequent reconfiguration this gives us:

```cpp
void LeaderboardView::set_columns(std::vector<ColumnInfo> &&cs) {
  auto locked_columns = _cols.get_mut();
  *locked_columns = std::move(cs);

  reconstruct_table(table.get_mut(), std::move(locked_columns),
                    _num_drivers.get_mut());

  // We no longer own the lock on the columns, so even though
  // locked_columns goes out of scope, the lock is not released
}

void LeaderboardView::reconstruct_table(
    LockPair<UI::Table> &&locked_table,
    LockPair<std::vector<ColumnInfo>> &&locked_columns,
    LockPair<size_t> &&locked_num) {

  // Reset dimensions, adding 1 row to account for column titles
  locked_table->set_dim(Table::Dim(*locked_num, locked_columns->size()));

  // Set alignment based on the columns
  for (auto [idx, c] : std::views::enumerate(*locked_columns))
    locked_table->column_props(idx)->align = c.align;

  // All three locks are going to go out of scope, and we own them
  // here so each lock will be released
}
```


## Epilogue -- There are wheels everywhere for those with the eyes to see {#epilogue-there-are-wheels-everywhere-for-those-with-the-eyes-to-see}

Very quickly after finishing this implementation, I noticed `LockPair<T>` is actually useful beyond as
just a conduit for `Locked<T>`. Elsewhere in the project, I have shared components which are a little
more complex than `Locked<T>` feels good for. In one such case, I have a frequently rearranged
vector of driver objects which themselves have a fair amount going on. Between this and existence of
implementation that I don't see the benefit in updating, this is a shared component which still must
be manually locked.

However, as part of a refactor, I needed to give a part of the UI a way to get a snapshot of this
vector, as it's the current ordering of the drivers on track. These drivers objects are not exactly
the smallest, and this is one of the most frequently hit sections of the codebase, so I don't want
to be copying the vector each time. Ultimately, what I wanted was a way to hand off a const
reference to this vector with an automatically handled lock.

Hey wait that sounds familiar.

Using `LockPair`, I can do exactly that, even though the vector and its mutex are not related to a
`Locked<T>` object whatsoever. Thus, the call for the UI to retrieve the drivers in their on-track
ordering is:

```cpp
ThreadSafe::LockPair<std::vector<DriverTelemetry> const>
TelemetryBoard::reorder_and_get() {
  // Order drivers by rank, then calculate gap to leader
  {
    std::unique_lock lock(_driver_vec_mtx);
    /* ... Omitted as it's not relevant ... */
  }

  // Acquire and return a shared_lock on the drivers vec
  return {std::shared_lock(_driver_vec_mtx), _drivers};
}
```

[^fn:1]: See [the github repo](https://github.com/schelcc/indy-tui) for the project, or
    [#indy-tui](/tags/indy-tui) for more here
[^fn:2]: Huge shoutout to this wonderful talk from David Olsen
    [Back To Basics: C++ Concurrency](https://www.youtube.com/watch?v=8rEGu20Uw4g)
[^fn:3]: A tip. If you are early on in a project and are
    already in enough of a deadlock mess to think that logging lock acquisition is going to be
    helpful. Maybe like, do something else?  Oops.
[^fn:4]: We need not discuss
    the caveat. It is certainly not a queue. Certainly I did not try a thread-safe queue based on what I
    remember from my computer organization class. Certainly.
[^fn:5]: Read: mutilation
[^fn:6]: I did not end up trying a macro. The world should
    consider itself lucky there is a part of my brain capable of keeping me from reaching for macros and
    `goto`.
[^fn:7]: Fear not for later that day I
    watched Casey Muratori's [The Big OOPs](https://www.youtube.com/watch?v=wo84LFzx5nI) and that concluded
[^fn:8]: To
    verify, I wrote up a simple compiler explorer example -- as far as I can tell, GCC and Clang seem to
    do different things with this `set()` then `get()` sequence. In GCC I cannot get the two to disagree
    (i.e. another `set()` occurs in between pair `set()` and `get()` calls), but in Clang it seems relatively
    frequent. I might investigate this further later, but here's the link:
    <https://godbolt.org/z/hqWrfPve3>
[^fn:9]: I can't find
    a link to the standards discussion but someone is proposing nested structured bindings which would be so good
    please I am begging
[^fn:10]: Huge shoutout to Rose
    please check out their page <https://ikl.sh/>
[^fn:11]: And tangent I will. If the
    `operator->()` overload returns a pointer, it behaves exactly as expected, giving us the equivalent of
    `(*obj).foo()`. But, if it returns a non-pointer, `operator->()` is then applied on _that_ value,
    repeating this until a pointer is returned, which is then dereferenced as usual. This "drilling
    down" behavior was (to me) surprising but I guess makes sense -- if we have `T** ptr` and want to
    access the object, it would be quite annoying to have to do `ptr->->foo()`.
[^fn:12]: An earlier attempt saw me returning a pointer
    whenever the wrapped object did not have the dereference operator defined, otherwise I returned a
    reference to the result of dereferencing it. Because of that drill-down behavior this actually
    punched straight through things like std::optional and would've been a nightmare...
[^fn:13]: I've just now
    realized that, if desired, we could further template this to allow us to define the possible lock
    types, meaning we _could_ extend this past shared and unique lock if we so desired
[^fn:14]: Which I'd
    want to do to avoid unnecessary contention as these would be updated often
[^fn:15]: What's here is already a bit
    vestigial -- I need to update some pieces which would wind up mooting this part. But it's a good
    example.
[^fn:16]: The columns
    are functors which produce a string given a driver's current information, so adding a new column of
    information is simply adding a new functor
