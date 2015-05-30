---
layout: post
title: Smooth Canvas animations using TextureView
feature-img: "img/sample_feature_img.png"
---

On multiple occasions I was faced with enthusiastic idea of implementing a CSS inspired animation. The problem with those is they are time consuming and in the end, tend not to look as good as expected. A little stuttering in the animation, wrong type of interpolation or color mixing and your loader looses all charm. In this article I would like to show off a technique I use to keep animations as smooth and as charming as possible.
<!--more-->
Let's use this one as an example [Source](http://codepen.io/wifeo/pen/FBpJq):

{% include loader1.html %}

The code presented here assumes API level 14+. Also, I decided to use Kotlin but the translation to Java should be easy enough.

You can find the complete [source code here](https://gist.github.com/andrzej-tokarski/a129abd82a5529229f9c).

#### Dissecting the animation

First, let have a closer look at the animation example. All credit for the animation design goes to [wifeo.com](http://www.wifeo.com/code/). Enabling CSS animation classes one at the time reveals two main components:

{% include loader2.html %}

### Implementation

A standard approach would be to create some Views and move them around using [ObjectAnimator](http://developer.android.com/guide/topics/graphics/prop-animation.html#object-animator). This would require adding 2-3 layers to our layout and a lot of fiddling with AnimatorSets. I would rather follow [the trend](http://stackoverflow.com/questions/30418710/how-does-the-gmail-app-achieve-its-flat-view-hierarchy) and keep the hierarchy [as flat as possible](http://developer.android.com/training/improving-layouts/optimizing-layout.html). Adding those might not have a great impact itself but can become a part of a issue later. Also, using AnimatorSet has [another](http://stackoverflow.com/questions/11986455/android-multiple-animatorset-reset) [issues](http://stackoverflow.com/questions/17492523/loop-animatorset).

Another way is to get a hold of a Canvas object and draw all elements manually. This can be tedious in some cases but drawing moving circles is not one of those. The simplest method of implanting a Canvas into a layout is extending View's onDraw method and animating it by calling invalidate() repeatedly. But we can do better then that.

#### TextureView 

The [TextureView documentation](http://developer.android.com/reference/android/view/TextureView.html) is a little bit cryptic but gives all the information:

* TextureView can be used to display a "content stream". In most cases it is using SurfaceTextures but there is also a mention of a Canvas object. 
* TextureView is a better behaving SurfaceView and implements all features that one can expect of a View class. It blends properly and can be moved around.
* It requires hardware acceleration to work.

Note that onDraw() method is final and, as explained, cannot be used for drawing. Drawing code has to wait for the SurfaceTexture to be available and stop the animation when the surface is destroyed. This can be done by implementing TextureView.SurfaceTextureListener:

{% highlight kotlin %}
    setSurfaceTextureListener(object:TextureView.SurfaceTextureListener {
        override fun onSurfaceTextureDestroyed(surface: SurfaceTexture?): Boolean {
            stopAnimation()
            animationStarted = false
            return true
        }
        override fun onSurfaceTextureAvailable(surface: SurfaceTexture?, width: Int, height: Int) {
            if(!animationStarted) {
                animationStarted = true
                startAnimation(width, height)
            }
        }
        override fun onSurfaceTextureSizeChanged(surface: SurfaceTexture?, width: Int, height: Int) {}
        override fun onSurfaceTextureUpdated(surface: SurfaceTexture?) {}
    })
{% endhighlight %}

I'm not storing SurfaceTexture reference because what we need is to know if it is available for drawing. To get a Canvas object we have to use a pair of lockCanvas() and unlockCanvasAndPost(canvas) calls. All operations done between those two will be translated to OpenGL and sent to the GPU. 

{% highlight kotlin %}
val canvas: Canvas? = lockCanvas()
if(canvas == null) return
...
unlockCanvasAndPost(canvas)
{% endhighlight %}

Here you can put all your "drawing logic". This way we draw a single (static) image. The onSurfaceTextureAvailable() callback will be called once (or more the once, but NOT once-per-frame) so we still need a way to repeatedly redraw the frame. Handler.postDelayed() is one method or you can even spawn a Thread to call your drawFrame() method but the effect will be resource intensive and/or far from buttery smooth.

#### ObjectAnimator secret sauce

The key to the smooth animation is to draw frames as fast as the screen is being refreshed. In fact, the perfect situation is when the next frame is ready just before the screen will be redrawn by the system. In multi-threaded environment getting everything in sync is not an easy task. If you want to dive deeper and know how this is achieved you should watch a [presentation about Project Butter](https://www.youtube.com/watch?v=Q8m9sHdyXnE) and read a [Graphics architecture documentation](https://source.android.com/devices/graphics/architecture.html). 

One element we are interested in is the [Choreographer](http://developer.android.com/reference/android/view/Choreographer.html). It is the object in charge of emitting frame synchronisation pulses. It knows when last frame was displayed and notifies the rest of the system that they should start drawing. It's a pretty low-level tool amd yt can be used directly by calling 

{% highlight kotlin %}
Choreographer.getInstance().postFrameCallback(Choreographer.FrameCallback callback)
{% endhighlight %}

but the recommended way of getting the timing right is to... use ObjectAnimators. They use Choreographer internally but are much more approachable. The added bonus is that the code will be more familiar. So let's take a simple ValueAnimator and use it to start drawing at the right moment:

{% highlight kotlin %}
val animator = ValueAnimator.ofFloat(0f, 2f)
animator.addUpdateListener(object:ValueAnimator.AnimatorUpdateListener {
            override fun onAnimationUpdate(animation: ValueAnimator) {
                drawFrame()
            }
})
{% endhighlight %}

Note that drawFrame() will run on main thread so you should keep it simple.

##### CSS animation easing

Looks like all "steps" in CSS animation are calculated separately. There is one ease curve between each of:

{% highlight css %}
0%{transform:rotate(0deg) scale(1);}
50%{transform:rotate(360deg) scale(1.3);}
100%{transform:rotate(720deg) scale(1);}
{% endhighlight %}

That means that it cannot be represented using single Interpolator. You have to use LinearInterpolator and calculate multiple interpolations based on that. 

##### Going back and forth

In loader animation are defined in CSS as a full cycle:

* scale starts at 1 goes to 1.3 and ends up at 1
* rotation starts at 0 and ends up back at 0 (full circle without changing direction)

The easiest way of defining both is to split LinearInterpolator range in two: 0f -> 1f -> 2f and reversing scale animation when value is bigger then 1f.

{% highlight kotlin %}
    val posInterpolator = if(Build.VERSION.SDK_INT >= 21) {
            PathInterpolator(0.25f, 0.1f, 0.25f, 1f)
        } else {
            com.codewise.animations.AccelerateDecelerateInterpolator()
            // https://developer.android.com/sdk/api_diff/22/changes.html
        }
    override fun drawFrame(width: Float, height: Float, value: Float) {
        val interpolatedPosition = if(value < 1f) {
            posInterpolator.getInterpolation(value)
        } else {
            posInterpolator.getInterpolation(value-1f)
        }
        val reverse = value < 1f
        drawFrame(width * 0.7f, height * 0.7f, interpolatedPosition, reverse)
    }
{% endhighlight %}

This way the drawing code will be much shorter but also a little bit hackish.

##### AccelerateDecelerateInterpolator before ApiLevel 21

Looks like Google decided to change the implementation of AccelerateDecelerateInterpolator in ApiLevel 22 making it incompatible with older versions. I decide to include a copy of an older implementation. 

##### Transparent background

You need to explicitly warn the system about the need to keep the background transparent by calling setOpaque(false). Also, to clear the canvas before drawing the next frame you have to use canvas.drawColor(Color.TRANSPARENT, PorterDuff.Mode.CLEAR) to make sure it has any effect. 

##### Drawing itself

Original animation scales the whole container. I decided to keep it simple and scale the circles instead. I did that by recalculating the diameter. There is a slight difference in the result but it is very hard to notice. The last step is to create the drawing code itself. It's a little bit tedious and very dependent on the design itself but here it is. I started with two helper methods. One for drawing a ball itself, somewhere between two positions and one for calculating the position. 

{% highlight kotlin %}
private fun drawBall(x1: Float, x2: Float, y1: Float, y2: Float, r1: Float, r2: Float, position: Float, reverse: Boolean, color: Int, paint: Paint, canvas: Canvas) {
    val alpha = (255 * interpolate(1f, 0.5f, position, reverse)).toInt()
    paint.setColor(color)
    paint.setAlpha(alpha)
    canvas.drawCircle(
        interpolate(x1, x2, position, reverse),
        interpolate(y1, y2, position, reverse),
        interpolate(r1, r2, position, reverse),
        paint)
}

private fun interpolate(start: Float, end: Float, delta: Float, reverse: Boolean): Float {
    if(reverse) {
        return end + (start-end) * delta;
    } else {
        return start + (end-start) * delta;
    }
}
{% endhighlight %}

And putting it all together is a drawFrame() method called from precise-timed onAnimationUpdate() callback:

{% highlight kotlin %}
private fun drawFrame(width: Float, height: Float, position: Float, reverse: Boolean) {
    val canvas: Canvas? = lockCanvas()

    if(canvas == null)
        return

    canvas.translate((canvas.getWidth() - width) / 2f, (canvas.getHeight() - height) / 2f)
    canvas.drawColor(Color.TRANSPARENT, PorterDuff.Mode.CLEAR)

    val w2 = width / 2f
    val w4 = width / 4f

    val r_1 = 0.8f * w4
    val r_2 = 1f * w4

    canvas.rotate(360f * position, w2, w2)

    drawBall(w4, w2,
            w4, w2,
            r_1, r_2,
            position, reverse, color1, paint, canvas)

    drawBall(3*w4, w2,
            w4, w2,
            r_1, r_2,
            position, reverse, color2, paint, canvas)

    drawBall(w4, w2,
            3*w4, w2,
            r_1, r_2,
            position, reverse, color3, paint, canvas)

    drawBall(3*w4, w2,
            3*w4, w2,
            r_1, r_2,
            position, reverse, color4, paint, canvas)

    unlockCanvasAndPost(canvas)
}
{% endhighlight %}

### The takeaway

This task is a nice example of a hidden trap for a developer. It's very easy to ignore the context and just run with a new Thread() in extended View and get a poor result. But a nicer effect can be achieved without much added complexity and with the same Canvas-drawing code just by using the TextureView and ObjectAnimators APIs.