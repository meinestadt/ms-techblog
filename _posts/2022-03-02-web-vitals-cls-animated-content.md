---
title:  Can animations help to fix CLS warnings?
excerpt_separator: "<!--more-->"
categories:
  - IT 
tags:
  - Web Animations
  - Core Web Vitals
  - CLS
published: true
---

There has been quite the [Google update](https://blog.chromium.org/2020/05/introducing-web-vitals-essential-metrics.html) which brings some new metrics to consider if you want to adhere to the Google guidelines about user experience.

Recently we spend some time fixing [CLS](https://web.dev/cls/) issues across the meinestadt.de platform.
Since most of the reported [Lighthouse](https://developers.google.com/web/tools/lighthouse) warnings could be fixed by adding missing [skeleton CSS](https://yossiabramov.com/blog/css-skeleton-loading) for dynamic contents, we got left with some non trivial issues where final dimension can not be determined. The problem:

> A third-party banner image with an unknown **dimension** / **aspect ratio** loads right into the head and pushes subsequent elements out of the viewport.

This becomes even more problematic on mobile devices where the image is scaled to 100% screen width and the resulting height depends on the viewport width.

Since we were not allowed to crop the image down to a common aspect ratio, we have to deal with a wide variety of different image heights in our pages which makes it impossible to use a Skeleton without creating a layout.

![CLS shift unknown img ratio](/assets/images/blog/001_cls/blog_cls_1.png)

In search of alternative options, we come across Googles' advice for using **CSS animations** to fix CLS problems.

>  [Animations and transitions](https://web.dev/cls/#animations-and-transitions), when done well, are a great way to update content on the page without surprising the user. Content that shifts abruptly and unexpectedly on the page almost always creates a bad user experience. But content that moves gradually and naturally from one position to the next can often help the user better understand what's going on, and guide them between state changes.

Unfortunately there is no clear definition for a „_CLS friendly animation_“ and this is where our investigation starts.

- Fast running CSS transitions might not differ that much from an instant layout shift.
- Slow running animations most likely feel unnatural to our users.

#### How does animation speed affect the CLS score of an element?

A smooth CSS animation runs at 60fps. Means a new frame is rendered every `16ms`. For easy evidence we started with a **1px shift per frame** which feels quite slow.

```
Transition height  (Δy):  180px
Transition duration (t):    3s (3000ms)
=> 1px gets movement per frame (~16,50ms)
```

<iframe height="550" width="635" style="width: 100%;" scrolling="no" title="CLS 1px/frame" src="https://codepen.io/exodus4d/embed/LYyBYXp?default-tab=result&theme-id=light" frameborder="no" loading="lazy" allowtransparency="true" allowfullscreen="true">
  See the Pen <a href="https://codepen.io/exodus4d/pen/LYyBYXp">
  CLS 1px/frame</a> by Mark Friedrich (<a href="https://codepen.io/exodus4d">@exodus4d</a>)
  on <a href="https://codepen.io">CodePen</a>.
</iframe>

The output we get from DevTools after taking a performance record of the transition looks promising: No content shift warnings!

[Lighthouse 1 -  performance]

> Very slow running CSS animations don't trigger CLS reports, even if subsequent visible content gets pushed out of viewport.

We can speed up the transition and **decrease the duration** to find the threshold for the *critical* speed on which Lighthouse starts to report layout shifts.

- A: Duration `990ms`  ➜ Lighthouse reports layout shift. ✖
- B: Duration `1000ms` ➜. Lighthouse reports layout shift.  ✖
- C: Duration `1010ms` ➜ No warnings  ✔

<iframe height="550" width="635" style="width: 100%;" scrolling="no" title="CLS critical duration" src="https://codepen.io/exodus4d/embed/jOmvWGX?default-tab=result&theme-id=light" frameborder="no" loading="lazy" allowtransparency="true" allowfullscreen="true">
  See the Pen <a href="https://codepen.io/exodus4d/pen/jOmvWGX">
  CLS critical duration</a> by Mark Friedrich (<a href="https://codepen.io/exodus4d">@exodus4d</a>)
  on <a href="https://codepen.io">CodePen</a>.
</iframe>

A shift of less than of equal to `3px` per frame (constant 60fps) seems to be fine without any reported CLS warnings. Or in other words `1px` can be moved every `5.55ms`.

Neither the size of a transitioned element nor the start offset of an animation affects the CLS score, as long as we keep inside the limit.

We can decrease the `duration` and `Δy` and get the same positive results, as long as the pixel shift remains identical.

<iframe height="550" width="635" style="width: 100%;" scrolling="no" title="CLS variable height" src="https://codepen.io/exodus4d/embed/rNmZeaL?default-tab=result&theme-id=light" frameborder="no" loading="lazy" allowtransparency="true" allowfullscreen="true">
  See the Pen <a href="https://codepen.io/exodus4d/pen/rNmZeaL">
  CLS variable height</a> by Mark Friedrich (<a href="https://codepen.io/exodus4d">@exodus4d</a>)
  on <a href="https://codepen.io">CodePen</a>.
</iframe>

#### Non linear transitions

Since most of the transitions are non linear with a non constant pixel shift per frame speed of pixel movement.

> An `ease-out` transition starts fast and slows down towards the end. Even if the average pixel shift would not cause CLS issues, we get CLS warnings for the first frames of the `ease-out`  transition.

<iframe height="550" width="950" style="width: 100%;" scrolling="no" title="CLS ease-out" src="https://codepen.io/exodus4d/embed/eYWLZXE?default-tab=result&theme-id=light" frameborder="no" loading="lazy" allowtransparency="true" allowfullscreen="true">
  See the Pen <a href="https://codepen.io/exodus4d/pen/eYWLZXE">
  CLS ease-out</a> by Mark Friedrich (<a href="https://codepen.io/exodus4d">@exodus4d</a>)
  on <a href="https://codepen.io">CodePen</a>.
</iframe>

Chrome DevTools can be used to figure out animation frames with CLS warnings.

- A: `linear` ➜ No warnings (previous example). ✔
- B: `ease-out` ➜ CLS warnings on start. ✖
- C: `ease-in-out` ➜ CLS warnings in the middle. ✖

![enter image description here](/assets/images/blog/001_cls/blog_cls_2.png)

From a mathematical point of view, a `cubic-bezier` animation is a **time based function**. The slope of a tangent through any point is the **instantaneous** animation `speed`.

> **Speed** is a scalar quantity – it is the rate of change in the distance travelled by an object (pixel).

From previous example we know (A: `linear`) has an **average** (constant) `speed` close to the limit, as from Lighthouse will penalise a frame with a CLS score. The non linear animations (B: `ease-out`; C: `ease-in-out`) get CLS warnings for individual frames where the slope of its tangent is greater than the constant `speed` from A.

![enter image description here](/assets/images/blog/001_cls/blog_cls_3.png)

In the 2nd half of the transition (slide up), we see the same amount of CLS-flagged frames.
This might be obvious since both run at the same (positive) `speed`, just in a reversed direction... but allows us to
draw conclusions about the `velocity`.

> **Velocity** represents the rate of change of position of an object (pixel). The magnitude of a velocity vector shows the `speed` of an object.

The **direction** of an animation (+up/-down/..) has no effect on the CLS score.

Since the final CLS score of an element is an **accumulation if the individual scores** per frame, we end up with penalties for animation frames where pixel movement exceeds the limit.

We can **increase the duration** or **decrease the animated distance**. This leads to an overall reduction of  (average) pixel shift per frame, of a non-linear transition. At some point even frames with fast pixel movement stay inside the limit.

#### Conclusion

In order to avoid CLS warnings, CSS transitions can be used to slide in dynamic loaded content. The limiting factor is the speed of a transition and we have to make sure the transition does not move faster than `3px` per frame.