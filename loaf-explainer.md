# Long Animation Frames (LoAF)
Long Tasks Revamped

## Diclaimer
This is work in progress. Feedback welcome, lots of things might change etc.

## History

Long tasks have long been a way to diagnose and track lack of responsiveness or "jankiness", which
eventually affects core web vital metrics like [INP](https://web.dev/inp/), or Lighthouse metrics
like [Total Blocking Time](https://developer.chrome.com/en/docs/lighthouse/performance/lighthouse-total-blocking-time/).
Developers have been using them with varying degrees of success, and now we can learn from the
experience and see what can be improved going forward.

## Where long tasks fall short

Long tasks rely on the underlying notion of a [task](https://html.spec.whatwg.org/#concept-task).
This is a somewhat well-specified term, but we found that it has a few shortcomings:

1. A task does not include the [update the rendering](https://html.spec.whatwg.org/#update-the-rendering)
phase. That phase includes `requestAnimationFrame` callbacks, resize observers, scroll observers and
so on. This means that a lot of the busy time that blocks feedback or animation is not actually
counted as part of the "long task", and developers can game their long task timing by moving long
operations into a `requestAnimationFrame` callback.

1. Some operations that should be tasks are not specified or implemented to use tasks.
For example, it is not specified how UI events are integrated into the event loop and
the use of tasks there is implementation-specific.

1. A task in implementations is used for internal scheduling, and not just as an
implementation of the spec concept of task. This means that changes to implementation detail related
to scheduling affects the measurement of long tasks, sometimes in unexpected, incompatible or
arbitrary ways. A big implementation change in chrome silently changed the meaning of long tasks,
when we started updating the rendering as part of a new task.

1. A task may contain multiple callbacks, e.g. dispatch several events. This makes
it sometimes confusing to decipher what was the root cause of a long task.

All of the above are part of the same issue - a task is an incomplete and inaccurate cadence to
measure main-thread blocking. It's either too granular (as several tasks together may be the cause
of blocking) or too coarse (as it may batch together several event handlers, and callbacks such as
`requestAnimationFrame` are not tasks in themselves).

### The Current Situation

The [HTML event loop processing model](https://html.spec.whatwg.org/#event-loop-processing-model)
can be roughly described as such:

```js
while (true) {
    const taskStartTime = performance.now();
    // It's unspecified where UI events fit in. Should each have their own task?
    const task = eventQueue.pop();
    if (task)
        task.run();
    if (performance.now() - taskStartTime > 50)
        reportLongTask();

    if (!hasRenderingOpportunity())
        continue;

    invokeAnimationFrameCallbacks();
    while (needsStyleAndLayout()) {
        styleAndLayout();
        invokeResizeObservers();
    }
    markPaintTiming();
    render();
}
```

However, the Chromium implementation is more like this:

```js
while (true) {
    const startTime = performance.now();
    const task = eventQueue.pop();
    if (task)
        task.run();
    uiEventQueue.processEvents({rafAligned: false});
    if (performance.now() - startTime > 50)
        reportLongTask();

    if (!hasRenderingOpportunity())
        continue;

    eventQueue.push(() => {
        // A new task! so this would report a separate longtask.
        uiEventQueue.processEvents({rafAligned: true});
        invokeAnimationFrameCallbacks();
        while (needsStyleAndLayout()) {
            styleAndLayout();
            invokeResizeObservers();
        }
        markPaintTiming();
        render();
    });
}
```


## Introducing LoAF

LoAF (long animation frame) is a new proposed cadence (period of time) and performance entry type,
meant to be a progression of the long task concept.

It's the time measured between when the main thread started doing any work (see `startTime`
[here](https://html.spec.whatwg.org/#event-loop-processing-model)), until it is either
[ready to paint](https://html.spec.whatwg.org/#event-loop-processing-model:mark-paint-timing) or
idle (has nothing to do). It may include more than one task, though usually up to 2. Because it ends
at the paint-mark time, it includes all the rendering obsevers
(requestAnimationFrame, ResizeObserver etc.) and may or may not include presentation time
(as that is somewhat implementation specific).

In addition to making the cadence fit better with what it measures, the entry could include extra
information to help understand what made it long, and what kind of consequences it had:

- Time spent in the different phases (rendering, layout-and-style).
- Time spent in forced layout/style calculations - e.g. calling `getBoundingClientRect`, doing more
processing, and then rendering (also known as "layout thrashing" or "forced reflow").
- Is the frame blocking input-feedback *in practice*. Note that a frame that blocks actual UI event
 would also be accessible via [event timing](https://w3c.github.io/event-timing/).
- User scripts processed during the time of the frame, be it callbacks, event handlers, promise
resolvers, or script block parsing and evaluation, and information about those scripts (source, how
long they've been delayed for).

## How a LoAF entry might look like
```js
const someLongAnimationFrameEntry = {
    entryType: "long-animation-frame",

    // See details below...
    startTime: frameStartTime,

    // https://html.spec.whatwg.org/#event-loop-processing-model (17)
    // This is a well-specified and interoperable time, but doesn't include presentation time.
    // It's the time after all the animations and observers are done, style and layout are done,
    // and all that's left is painting & compositing.
    duration,

    // Whether this long frame was blocking input/animation in practice
    // A LOaF can block both, in which case ui-event would take precedent.
    // (Not implemented yet)
    blocking: 'ui-event' | 'animation' | 'none',

    // https://html.spec.whatwg.org/#update-the-rendering
    renderStart,

    // https://html.spec.whatwg.org/#update-the-rendering (#14)
    styleAndLayoutStart,

    // The time the animation frame was queued. This could be before startTime, which means that
    // the animation frame was delayed, or after, which means that it was deferred - several updates
    // were batched together before scheduling a frame.
    desiredRenderStart,

    // The implementation-specific time when the frame was actually presented. Should be anytime
    // between the previous task's |paintTime| and this task's |taskStartTime|.
    // (Not implemented yet)
    presentationTime,

    // Time of the first UI event (mouse/keyboard etc.)
    firstUIEventTimestamp,
    scripts: [
        {
            type: "user-callback" | "classic-script" |
                        "module-script" | "event-listener" | "resolve-promise" | "reject-promise"

            // these can be classic callbacks, event handlers, or promise resolvers
            // The name is the object.function of the registration function (the function initially
            // called to generate this callback).
            name: "HTMLImgElement.onload" | "Window.requestAnimationFrame" | "Response.json",

            // when the function was invoked
            startTime,

            // If this script was parsed/compiled, this would be the time after compilation.
            // Otherwise it would be equal to startTime
            executionStart,

            // the duration between startTime and when the subsequent microtask queue has finished
            // processing
            duration,

            // Total time spent in forced layout/style inside this function
            forcedStyleAndLayoutDuration,

            // The time when the callback was queued, e.g. the event timeStamp or the time when
            // the timeout was supposed to be invoked.
            desiredExecutionStart,

            // In the case of promise resolver this would be the invoker's source location
            sourceLocation: "functionName@URL:line:col",

            // Relationship between the (same-origin) window where this script was executed and
            // this window.
            windowAttribution: "self" | "descendant" | "ancestor" | "same-page" | "other"

            // A reference to the same-origin window that originated the script, if it's still
            // alive.
            window,
        }
    ]
}
```

### Processing model

The new proposal:

```js

let frameTiming = null;

while (true) {
    if (frameTiming === null) {
        frameTiming = new AnimationFrameTiming();
        frameTiming.startTime = performance.now();
    }

    const task = eventQueue.pop();
    if (task)
        task.run();

    if (!hasDocumentThatNeedsRender()) {
        frameTiming.renderEnd = performance.now();
        if (frameTiming.renderEnd - frameStartTime > 50)
            reportLongAnimationFrame();
        frameTiming = null;
        continue;
    }

    if (!hasRenderingOpportunity())
        continue;

    invokeAnimationFrameCallbacks();
    frameTiming.styleAndLayoutStart = performance.now();
    for (const document of documentsInThisEventLoop) {
        while (document.needsStyleOrLayout()) {
            document.calculateStyleAndLayout();
            invokeResizeObserverCallbacks();
        }
    }
    frameTiming.renderEnd = performance.now();
    markPaintTiming();
    if (frameTiming.renderEnd - frameStartTime > 50)
        reportLongAnimationFrame();

    frameTiming = null;
    render();
}
```

### Security & Privacy Considerations

At the most part, LoAF only exposes information across same-origin windows. Information about
scripts within a window is already observable, e.g. using resource timing or a service worker.

However, LoAF might expose rendering information for a particular document tree that may be
cross-origin (same-agent/site). The details about rendering the frame, such as
`styleAndLayoutStartTime`, are proposed to be visible to all the same-agent windows that are
rendered serially. That's because this information is already observable, by using
`requestAnimationFrame` and `ResizeObserver` and measuring the delay between them. The premise is
that global "update the rendering" timing information is already observable across same-agent
windows, so exposing it directly does not leak new cross-origin information. However, the idea
exposing less information to cross-origin same-agent subframes (as in, expose ) is open for
discussion.

### Notes, complexity, doubts, future ideas, TODOs

1. One complexity inherited from long tasks is the fact that the event loop is shared across
windows of the same [agent](https://tc39.es/ecma262/#sec-agents) (or process). The solution here is
a bit different but relies on similar principles:

    1. Only frames in visible pages report long frames.

    1. An observer fires only if its rendering was blocked by the long frame in practice, or if
    the long task (that didn't cause a render) belonged to that page.

    1. Breakdown to scripts is only available to the frame where they were invoked. Other frames
    receive an "opaque" breakdown: attribution of a blocking task to a different window - similar to
    the existing [attribution](https://w3c.github.io/longtasks/#sec-TaskAttributionTiming).


1. To avoid the magic 50hz number, consider making the threshold configurable,
or rely on "discarded rendering opportunities" as the qualifier for sluggishness alongside (or
instead of) millisecond duration.

1. Consider separating layout & style durations.

1. Exposing source locations might be a bit tricky or implementation defined.
This can be an optional field but in any case requires some research.

1. TBT & TTI are lighthouse values that rely on long tasks. Should they be modified to use LoAFs
instead? Are those metrics useful?

1. Clarify how this correlates to [JS Profiler markers](https://github.com/WICG/js-self-profiling/blob/main/markers.md). In general performance observer aspire to be expose succinct important information with
minimal overhead, while profiling exposes "everything" with some more overhead, but the differences
and relationship can be further understood.

## Overlap with [Event Timing](https://w3c.github.io/event-timing/)

With all the new data that LoAFs expose, their overlap with event timing grows. This is true, but it's only a problem if we look at them as separate APIs.

The [dozen-or-so different entry types](https://w3c.github.io/timing-entrytypes-registry/) that go into a performance timeline are not
separate APIs per-se, but rather queries into the same dataset - the chain of events that helps us understand sluggishness, jank, and instability.

As such, event-timing and LoAF query that dataset differently:
- event timing is driven by input->feedback, regardless of cause, which
could be long main thread processing but not necessarily. This is useful to catch regressions in UX for real users without assuming what causes them.
- LoAF is about catching discarded rendering opportunities. This is a useful
to diagnose the root cause of high INP or sluggishness.
