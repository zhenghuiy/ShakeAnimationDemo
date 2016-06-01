# ShakeAnimationDemo
Android shake animation demo

大家好，我是[光源](http://www.jianshu.com/p/699ca079598f)。在日常使用app或者玩游戏的过程中，我们经常可以看到某个view通过抖动来吸引用户注意，今天就来说说怎么实现这个动画。

具体需求是，**实现一个抖动动画要求同时对大小、旋转角度进行更改且可定制**。

要实现动画，我们首先应该想到的是 Android 中动画相关的内容。

Android 中一共有三类动画：

- View Animation
又称补间动画，在 android.view.animation.Animation 类之下衍生了五个子类。


  | 类名| 作用|
  | ---| ---- |
  | [AlphaAnimation](https://developer.android.com/reference/android/view/animation/AlphaAnimation.html) | 渐变透明度 |
  | [RotateAnimation](https://developer.android.com/reference/android/view/animation/RotateAnimation.html) | 旋转    |
  | [ScaleAnimation](https://developer.android.com/reference/android/view/animation/ScaleAnimation.html) | 尺寸缩放  |
  | [TranslateAnimation](https://developer.android.com/reference/android/view/animation/TranslateAnimation.html) | 位置平移  |
  | [AnimationSet](https://developer.android.com/reference/android/view/animation/AnimationSet.html) | 动画集合  |

​通过前四个类，基本可以解决大部分动画需求，再使用 AnimationSet 使动画具有组合的能力。

- Drawable Animation
又称逐帧动画，通过设置多个帧在一定时间内不断进行帧的变换形成动画的效果，类似 gif 图。通过 xml 中的 animation-list 标签定义动画，再在 java 代码中用 AnimationDrawable 类来进行控制。

- Property Animation
 View Animation 虽然可以解决大部分动画，但还是有些无法实现，而 Drawable Animation 则太过费时费力，所以在 Android 3.0（API 11）引入了属性动画，属性动画实现原理就是修改控件的属性值实现的动画。具体实现又分为 ValueAnimator 和 ObjectAnimator，这里不展开。

回到需求本身，从需求上看，三种方式都可以实现（其实对最接近动画本质的逐帧动画而言，还真没有不能实现的动画 ），这里不妨三种方式都尝试一下(**为方便代码展示，以下尽量使用java代码实现动画**)。

# 方案一：使用 View Animation
看代码：

````java
private void startShakeByViewAnim(View view, float scaleSmall, float scaleLarge, float shakeDegrees, long duration) {
        if (view == null) {
            return;
        }
        //TODO 验证参数的有效性

        //由小变大
        Animation scaleAnim = new ScaleAnimation(scaleSmall, scaleLarge, scaleSmall, scaleLarge);
        //从左向右
        Animation rotateAnim = new RotateAnimation(-shakeDegrees, shakeDegrees, Animation.RELATIVE_TO_SELF, 0.5f, Animation.RELATIVE_TO_SELF, 0.5f);

        scaleAnim.setDuration(duration);
        rotateAnim.setDuration(duration / 10);
        rotateAnim.setRepeatMode(Animation.REVERSE);
        rotateAnim.setRepeatCount(10);

        AnimationSet smallAnimationSet = new AnimationSet(false);
        smallAnimationSet.addAnimation(scaleAnim);
        smallAnimationSet.addAnimation(rotateAnim);

        view.startAnimation(smallAnimationSet);
    }
````

使用 ScaleAnimation + RotateAnimation 形成一边先小后大一边摇晃的动画效果。为了效果，需要使 view的缩放效果速率远小于摇晃效果，这里采用10倍差距。效果如下：

![](http://upload-images.jianshu.io/upload_images/1432874-c6b83aab3755927c.gif?imageMogr2/auto-orient/strip)

这个方案的优点是：
1. 使用了常见的动画方案，使用成本与代码阅读成本较低；

缺点是：
1. 为了实现左右连续抖动的效果，把两个动画的初始值都没有设置为 view 本身的属性大小，导致初始和结束时会有突变；
2. 缩放效果和摇晃效果的速率区别是通过 duration 的倍数 + 动画的 reverse repeat 来实现的，意味着 duration 必须作为初始化参数传入；
3. 在 view 变大时，摇晃的焦点还是以前的位置，导致摇晃的效果不大好。
4. 对动画的定制粒度太大，比如这里的需求是先小后大，实际上是有两个动画，但是 View Animation 中并没有串行执行动画的方法提供，这里是用取巧的方式实现，颇花费了一番心思。

# 方案二：使用属性动画

看代码：

```java
private void startShakeByPropertyAnim(View view, float scaleSmall, float scaleLarge, float shakeDegrees, long duration) {
        if (view == null) {
            return;
        }
        //TODO 验证参数的有效性

        //先变小后变大
        PropertyValuesHolder scaleXValuesHolder = PropertyValuesHolder.ofKeyframe(View.SCALE_X,
                Keyframe.ofFloat(0f, 1.0f),
                Keyframe.ofFloat(0.25f, scaleSmall),
                Keyframe.ofFloat(0.5f, scaleLarge),
                Keyframe.ofFloat(0.75f, scaleLarge),
                Keyframe.ofFloat(1.0f, 1.0f)
        );
        PropertyValuesHolder scaleYValuesHolder = PropertyValuesHolder.ofKeyframe(View.SCALE_Y,
                Keyframe.ofFloat(0f, 1.0f),
                Keyframe.ofFloat(0.25f, scaleSmall),
                Keyframe.ofFloat(0.5f, scaleLarge),
                Keyframe.ofFloat(0.75f, scaleLarge),
                Keyframe.ofFloat(1.0f, 1.0f)
        );

        //先往左再往右
        PropertyValuesHolder rotateValuesHolder = PropertyValuesHolder.ofKeyframe(View.ROTATION,
                Keyframe.ofFloat(0f, 0f),
                Keyframe.ofFloat(0.1f, -shakeDegrees),
                Keyframe.ofFloat(0.2f, shakeDegrees),
                Keyframe.ofFloat(0.3f, -shakeDegrees),
                Keyframe.ofFloat(0.4f, shakeDegrees),
                Keyframe.ofFloat(0.5f, -shakeDegrees),
                Keyframe.ofFloat(0.6f, shakeDegrees),
                Keyframe.ofFloat(0.7f, -shakeDegrees),
                Keyframe.ofFloat(0.8f, shakeDegrees),
                Keyframe.ofFloat(0.9f, -shakeDegrees),
                Keyframe.ofFloat(1.0f, 0f)
        );

        ObjectAnimator objectAnimator = ObjectAnimator.ofPropertyValuesHolder(view, scaleXValuesHolder, scaleYValuesHolder, rotateValuesHolder);
        objectAnimator.setDuration(duration);
        objectAnimator.start();
    }
```

为了能使多个动画同时进行，使用属性动画中的 PropertyValuesHolder 来实现。分别对 scaleX、scaleY、 rotation三个属性做属性动画处理。实现效果如下：

![](http://upload-images.jianshu.io/upload_images/1432874-efe37e0b3bead651.gif?imageMogr2/auto-orient/strip)

从代码上看，这里实现缩放速率和摇晃速率不一致是通过 keyFrame 的设置实现的，与 duration 参数解耦。

这个方案的优点是：

1. 对动画的定制粒度较小，理论上可以对无限小的动画帧进行定制。
2. 基于第一点的缘故，实现效果较好，基本可以实现任何动画效果。


# 总结

本文提出两种方案，上文也提到了，实际上任何动画都可以用帧动画来实现，只不过费时费力，所以方案三我就不多说了。**另外还有一个方案四，就是自定义Animation，在一个动画周期内对控件属性进行修改，这种方案虽然属于Animation的范畴，但是实质与属性动画一致，这里就不做扩展，感兴趣的读者可以留言讨论。** 

从上面的比较可以看出，View Animation 比较适合于一些简单的动画效果，可以达到快速开发的目的。而一些复杂的、复合的动画效果，用属性动画则事半功倍，效果也会比较好。当然属性动画的局限性在于，如果对应的控件没有动画所需的属性，则“巧妇”也难以解决。

为了便于大家比较，我写了一个demo放在github上，在保证各种变量尽量一致的情况下可以看到两种方案的实现效果——当然如果你有更好的方案欢迎留言告知。

![](http://upload-images.jianshu.io/upload_images/1432874-98f12aa520dd60d0.gif?imageMogr2/auto-orient/strip)

代码地址：https://github.com/zhenghuiy/ShakeAnimationDemo

# 参考资料

http://blog.csdn.net/harvic880925/article/details/50752838
