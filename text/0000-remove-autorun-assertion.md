# Microtasks in Backburner.js: Embrace the Autorun

## Attribution

The following plan was devised in cooperation between [@rwjblue](https://github.com/stefanpenner), [@krisselden](https://github.com/krisselden),  [@stefanpenner](http://github.com/stefanpenner),  and others.

## Summary

Since Embers inception, setting a property that is used in a template does not
directly update the DOM, instead we batch the update to avoid things like
layout thrashing. The queueing system that Ember uses to manage these queues of
work is a small stand alone library:
[Backburner](https://github.com/BackburnerJS/backburner.js).

Ember exposes a few different APIs in the `@ember/runloop` module (known as
`Ember.run` and `Ember.run.*` in globals land) that are backed by Backburner.
Let's take a quick look at a couple key methods:


- `run(someFunction)` — This invokes `someFunction` and then **forces** a
  synchronous flush of any work queued up by `someFunction`.
- `join(someFunction)` — This invokes `someFunction` from within the current
  runloop instance (starting one if one isn’t already started), any work
  scheduled from within `someFunction` will be flushed when the runloop that
  was joined completes.
- `schedule(queueName, someFunction)` — This schedules `someFunction` to be
  invoked when the `queueName` queue is flushed. Generally, this means at the
  end of the current runloop but when there is no runloop Backburner will
  create an “autorun” to run on the next turn of the JS runtime’s event loop
  (essentially a `setTimeout(flush, 0)`.

Internally, Ember prefers `join` over `run` usage to avoid creating nested
runloops and forcing a flush multiple times during a given action/interaction
cycle. In fact, the nested runloop situation will cause *exactly* what we are
trying to avoid by using a batching / queueing system in the first place!

At this point you may be asking: 


> If Ember itself doesn’t use it directly and its ‘bad’, why do I have to
> “sprinkle” magical `Ember.run` invocations throughout my app and test code?

To answer that, we need a history lesson first👴. As some of you may remember,
early versions of Ember didn’t *really* have a testing story. Clearly that was
a major issue, and a bunch of effort was put into creating a basic test
harness. It quickly became clear that operations as simple as
`controller.set('title','Awesome Title!')` were quite difficult to test because
the rendering was being batched for a future DOM update. This is why `run`
became a “thing” outside of Embers own internals.  Wrapping that `.set` in a
`run` would forcibly flush any scheduled work and we could test again 🎉 . As
this work progressed it became clear that if a developer accidentally forgot to
wrap any “side effecty” code in a `run` they would once again be prey to the
issue of not being able to test. In order to *avoid* trolling our users who
were just starting to try to write tests, an assertion was added so that any
scheduling of tasks while testing would throw an error if it wasn’t done from
within a runloop. Thus the dreaded “autorun assertion” was born…

Fast forward a few years, and we have **massively** more tools to deal with
this sort of asynchrony in JavaScript (yippie!). As you may have guessed by
now, emberjs/rfcs#232 and emberjs/rfcs#268 essentially remove the need for even
testing code to need this “force flush” behavior by leveraging the new tools in
JS (namely promises and `async` / `await`) to wait until things have settled
naturally (much closer to how a “real” app would work in production).

Given that we have a good path for waiting on scheduled work to be completed in
tests (leveraging standard JS features), which was the original reason for
adding the autorun assertion, can we remove that assertion and *Embrace the
Autorun*?

The basic premise of this RFC is to detail how we can migrate to *relying* on
the autorun behavior, remove the autorun assertion, removing the majority of
“run loop soup” that Ember developers have to deal with today.

## Motivation

There are a number of challenges that face Ember developers that we are now in
a position to address.

**Native Promises**

Ember has deep integration with [RSVP](https://github.com/tildeio/rsvp.js)
promises which ensures that anytime an `RSVP.Promise` resolves, it is properly
scheduled into the runloop queues. This integration allows us to avoid having
to `run` wrap every `.then` callback that may schedule work (e.g. set a
rendered property) throughout your app.

Now that native promises are available in the majority of browsers that Ember
supports, applications are commonly forced to interoperate with them.
Unfortunately, the integration that Ember has with RSVP is not possible with
native promises and therefore when an application needs to use them it often
has to either wrap in an RSVP promise or manually `run` wrap any `.then`
callbacks that would schedule future work.

For example:

```hbs
{{! app/templates/components/gist-edit.hbs }}
{{flash-alert}}
{{! ...snip...}
<button {{action 'update'}}>Update Gist</button>
```

```js
// app/components/gist-edit.js
export default Component.extend({
  actions: {
    update() {
      fetch('https://api.github.com/gists', {
        method: 'post',
        body: JSON.stringify({ /* snip */ })
      })
        .then(() => {
          this.set('flash-alert', 'successfully updated!');
        });
    }
  }
});
```

The example uses native `fetch` to make a `POST` request, then updates the
“flash message” to let the user know that the update was a success. Prior to
the changes proposed by this RFC this example would trigger an autorun (to
schedule the rendering updates after the `fetch` request is completed) which
means it would trigger an autorun assertion in tests. 😩

**async / await**

I personally believe that `async` / `await` is **the best** feature added to
JavaScript in the “post-ES5” era, and I want to use it **EVERYTIME** I have to
`.then` off of another promise 😈. 

Unfortunately, this leads us directly into the issues mentioned above with
native promises (since `async` / `await` is *essentially* using the native
promise under the hood).

**Autorun Assertion**

The autorun assertion is something that was created to allow Ember developers
to be able to test. However at this point, it is often needlessly noisy. For
example, which of the two examples below seem better:

```js
// Example A
test('does a thing', function(assert) {
  // createRecord schedules work to flush all live record arrays to be updated
  // with the autorun assertion, `run` wrapping is required
  let subject = run(() => this.owner.lookup('service:store').createRecord('foo'));
  // ...snip...
});

// Example B
test('does a thing', async function(assert) {
  let subject = this.owner.lookup('service:store').createRecord('foo');
  await settled(); // naturally allow the scheduled work to be performed...
  // ...snip...
});
```

**Performance**

Autoruns today, have more overhead then they should. This is because they use
`setTimeout`, instead of a MicroTask(`Promise`/`MutationObserver`).
`setTimeout` is both throttled to at best 4ms, but also deprioritized relative
to other operations (e.g. paint/click/xhr/ws/gc/etc). This can easily cause
large gapped idle times.

This becomes quite obvious when using fetch, where fetch is quite fast but we
end up often idling needlessly for the autorun to flush.

**Why Now?**

As of Ember 3.0.0 we have dropped support for browsers that do not have support
for `Promise` / `MutationObserver`. This means that we are finally in the
position to use the next turn mechanism that we have always intended to
leverage, and offer it consistently across all our supported browsers.  Prior
to this, we were worried about providing divergent behaviors between different
browsers.

## Detailed design

**Design Summary**

In broad strokes the way to unlock the features mentioned above is to migrate
auto run scenarios (which are already async today via `setTimeout`) to use the
microtask queue (via `promise.then` when possible, falling back to
`MutationObserver` for support of IE11) and transition between queues during an
autorun triggered flush via microtasks. 

Using this technique, we make autoruns something that we can reliably embrace
(as they will now be avoiding the issues that originally forced use to add the
assertion) and can therefore remove the current auto run assertion.

At this point, using `Ember.run` would be completely optional, but still have
*exactly* the same semantics. Doing a manual run (even mid auto run flush) will
synchronously flush the queues just as it does today. 

**Detailed Steps**

The steps listed below assume a somewhat deep knowledge of
[Backburner.js](https://github.com/BackburnerJS/backburner.js) internals, and
is mostly intended as a “road-map” for how to accomplish the stated goals
above.


1. Make autoruns schedule into the microtask queue
   ([BackburnerJS/backburner.js#306](https://github.com/BackburnerJS/backburner.js/pull/306))
2. When a given `DeferredActionQueue` instance is being flushed as a result of
   an auto-run, use the microtask queue to advance from one queue to the next
   (also included in the PR above)
3. Add `settled` as a first class API to Backburner (Stef to work on specific
   semantic).
4. Remove Ember’s auto run assertion (most/all of the objectives in the
   motivation section are addressed at this point).
5. Profit 💸

After each of the items in the listing above, we should be able to update the
version of backburner without introducing breakage.

## How we teach this

This feature (like many of our recent features) is all about *removing*
something that has to be taught. If we can embrace the auto-run in normal code,
then we can remove a whole (fairly complex and poorly understood) concept from
the Ember developer mindset.

The primary teaching concept here will be to ensure to review the guides for
mention of run-wrapping, and remove them. In additional the [run-loop
guide](https://guides.emberjs.com/v2.18.0/applications/run-loop/) will need to
be updated. Specifically, we need to change verbiage that mentions the original
reasoning for the autorun assertion, remove instructions to `Ember.run` wrap
things, etc.

## Drawbacks

Possible issues with things like `window.open` when ran from within the
microtask queue. [@stefanpenner](https://github.com/stefanpenner) has done a
bunch of research in this space over in
[BackburnerJS/backburner.js#181](https://github.com/BackburnerJS/backburner.js/issues/181).
Stef will continue working on this with browser vendors…

## Alternatives

There are a couple alternatives all of which have been rejected for one reason
or another, but I’ll list them here anyways:

**Replace Backburner**
This is the “nuclear” option. Replacing Backburner with another system that is
*only* attempting to manage the run loop queue scheduling (and not also
managing things like .`debounce`, `.throttle`, `.later`) would allow a fairly
significant simplification since the new library would be able to drop legacy
support and only provide microtask queue flushing. Unfortunately, this doesn’t
support existing Ember versions and would be a breaking change. Breaking
changes are painful. We don’t need more pain.

**Do Nothing**
To me, this is a non-option. The list of features in the motivation section is
sufficiently important that I do not believe we have the option of “doing
nothing”…

## Unresolved questions
- ???

