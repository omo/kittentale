---
layout: post
title: "Scrolling Meets Compositor"
date: 2012-03-04 13:08
comments: true
categories: 
---

A few weeks ago, I noticed that some engineers from Apple landed a
small module behind `THREADED_SCROLLING` flag. Thread? The name sounds
interesting.  So I took a quick look. And here is a rough finding from
that: This is a machinery aiming to make scrolling as smooth as CSS
animation, it looks.

To find how it works, we need some basic understanding about
implementation of CSS animation and scrolling. Let me start from the
former, then proceed to the later, which includes the today's highlight.

CSS Animation and `ACCELERATED_COMPOSITING`
--------------------------------------------

WebKit runs CSS Animations smoothly. This is especially true
for ones over CSS 3D Transforms. To make this happen, a WebKit feature
called `ACCELERATED_COMPOSITING` was introduced at the beginning of
2009, which brought the power of GPU acceleration into WebKit.

Here is a basic idea behind the accelerated compositing: With
accelerated compositing enabled, WebKit renders the page into mutually exclusive parts called "layers", 
then render each of them into an offscreen image separately, 
and "composite" it after all these images are rendered. 

![image](https://lh5.googleusercontent.com/-mJPWNXIzV6g/T1YnMYrZRuI/AAAAAAAAD4I/AFfLieWjTrg/s800/P1010213.JPG)

The point here is that each rendered image of the layer - it's called a "backing store" in WebKit by the way - 
is located on the GPU side. Thus the composition of the set of backing stores into the final image can be 
done pretty quickly with little CPU load, thanks to the GPU speed.

Basically, a layer is corresponding to a subtree of the page's DOM.
And the whole DOM tree is splitted into layer subtrees based on a certain set of CSS properties.
For example, if there is a `<div>` which is styled to have
3d-transormation, the `<div>` will establish a new layer. 
And the `<div>`-rooted subtree will be painted into the layer's backing store.

Since each layer has its own backing store, we don't need to
repaint a layer even if others get modified and repainted. 
What we need is just re-composite these backing stores.
This composition is orders of magnitude fater than repainting the whole page - GPU is 
a device designed for these types of operations after all.

We also don't need to repaint a layer if only its "transformation" is
changed. The browser can handle such tranformations in the GPU side as
a part of the composition process.

Backing stores, fast compositions and cost-free transformations; 
Thanks for these characteristics, WebKit can render 3D-ish CSS animations quite smoothly.

Core Animation for Better Slickness
------------------------------------------

Things go further if you are using Apple ports of WebKit.
On Mac (or Apple Windows), the composition is implemented on top of 
[Core Animation](https://developer.apple.com/library/mac/#documentation/cocoa/conceptual/coreanimation_guide/introduction/introduction.html)
infrastructure. As the name implies, Core Animaton helps smooth animation like CSS 3D Transforms. 
And its implementation fits the backing store model of `ACCELERATED_COMPOSITING`. 
In reality, `ACCELERATED_COMPOSITING` was first desinged for Core Animations.

One great advantage of Core Animations - WebKit calls it "CA" for short - is that
it runs these animations in a different thread. So it can run smoothly
even if the main thread is getting stuck by, for example, heavy JavaScript programs.
Without threaded backends like CA, animations for CSS transforms can stuck when 
big chunk of computations including layout or JS happen in the main thread.

![image](https://lh3.googleusercontent.com/-_eo1XHpkpLc/T1YnOrQO8_I/AAAAAAAAD4c/x4kwOQO1KV4/s800/P1010215.JPG)

This out-of-thread animation will work especially well for mobile devices.
Because CPUs in mobile devices are relatively slow, and at the same time, they start
to have another CPU core. CSS animation on mobile is one of the best
place where CA-like system shines.

Chromium also has its own composition subsystem, which is called
[cc](http://trac.webkit.org/browser/trunk/Source/WebCore/platform/graphics/chromium/cc).
But I don't know much about that... Well, you can blame me ;-)

Scrolling - Slick or Quick?
------------------------------------------

Scrolling is another place where browsers can be slicker.

Don't take me wrong. Browsers do scrolling well. But it's more like *quick* than *slick*.
Typical page scrollings happen by per-line, or per-page basis.
If you hit "down" key, it scrolls the page about ten pixels. 
If you click scrollbar, it jumps over hundreds of pixels. 
(FYI, WebKit supports four types of scroll "granuality": line, page, document and pixel.
You can find them at [ScrollTypes.h](http://trac.webkit.org/browser/trunk/Source/WebCore/platform/ScrollTypes.h).)
This large-step scrolling has been working well. But it may not be felt slick to skip dozen pixels per frame.

On Mac (again!), the situation is different. Safari (and Mac Chromium) does scrolling slickly intead of quickly; 
If you click a scroll bar, you'll see an animated scroll. It scrolls down a page height using tens of frames, and each frame contains only small delta.
WebKit implements this slick scroll as [ScrollAnimator](http://trac.webkit.org/browser/trunk/Source/WebCore/platform/ScrollAnimator.h)
class and the family. They live behind `ENABLE(SMOOTH_SCROLLING)` flag. 

Note that the animated scrolling is a different gear from the accelerated compositing.
These two don't share same machinery for some reasons:
First, scrolling makes previously-invisible regions visible - This is why people scroll the page. 
So browsers need to paint these regions. This prevents us from just compositing existing backing stores.

Second, the scrolling animation isn't "declarative". In other word, it cannot be described on top of CA's animation model.
You need to write your own code to compute the scrolled position.

First one is especially tough. What worse is that we might have to re-layout if the page has something like `position: fixed`. 
There are also pages which monitor scrolling to trigger JavaScript for tweaking something funny.
All these stuff causes a pause and prevents smooth scrolling in a reasonable frame rate.
This is one of the reason why many non-Mac browsers choose quick, jumpy scrolling instead of slick one.

Scrolling on Touch Devices
----------------------------

This story has a blind side though. In fact, browsers for touch devices
like Mobile Safari take an advantage of that.

When you use Mobile Safari, you'll notice that their scrolling and
zooming is extremely smooth. But at the same time, you'll also notice
that sometimes the page painting doesn't catch up the scrolling (and
the zooming). That is the trick: While scrolling, the browser skips some
painting to gain the animation frame-rate.

In theory, this behavior
isn't correct. An ideal browser shouldn't show any blank region to the
user. But in practice, this is a totally sensible trade-off.

WebKit itself doesn't support this kind of scrolling-without-painting.
So browsers do it by themselves. 