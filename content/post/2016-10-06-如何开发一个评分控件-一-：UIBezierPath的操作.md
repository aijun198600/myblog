---
layout: post
title: 如何开发一个评分控件(一)：UIBezierPath的操作
date: 2016-10-06
tags: 
    - iOS
    - CALayer
categories: [ Code ]
---

> 评分控件在APP中非常常用，一般常见的形状为星型或者心型（❤）。一般我们比较常用的方法是一个一个图片或者按钮去堆砌，但是这样应付简单整数可以，比如4.7分这样非整数就并不好显示。本文将实现一个能复用的评分控件，不仅仅可以准确地显示非整数的评分，也可以灵活地自定义不同的参数，甚至设置各种不同的形状。

# 背景

我们主要是使用UIBezierPath对象来实现星星或者心（❤）的形状，我们自己用代码去写不同的形状就比较复杂了，这里我们会使用PaintCode来减少工作量。PaintCode的具体操作就不多做介绍了，大家可以参考下面的文章：

[PaintCode 教程一](http://www.jianshu.com/p/5e75408812df)
[PaintCode 教程二](http://www.jianshu.com/p/8fe3595f7435)
[使用paitcode来完成applewatch刻度盘动画效果](http://www.jianshu.com/p/a9502b37e8cf)
[使用paintcode 制作一个星星评分视图](http://www.jianshu.com/p/e2efd7132bc1)

其实在上面的文章已经有星星评分控件的一种实现方法了，不过存在着不能使用非纯色或者透明色背景颜色的缺点，而且也没有进行封装。我们会通过UIBezierPath对象的运算操作来改善这个问题。

# UIBezierPath

我们首先将PaintCode生成代码复制到自定义UIView的```- (void)drawRect:(CGRect)rect```方法中，代码如下：
```
- (void)drawRect:(CGRect)rect {
    UIBezierPath* starPath = [UIBezierPath bezierPath];
    [starPath moveToPoint: CGPointMake(56, 13)];
    [starPath addLineToPoint: CGPointMake(64.82, 25.86)];
    [starPath addLineToPoint: CGPointMake(79.78, 30.27)];
    [starPath addLineToPoint: CGPointMake(70.27, 42.64)];
    [starPath addLineToPoint: CGPointMake(70.69, 58.23)];
    [starPath addLineToPoint: CGPointMake(56, 53)];
    [starPath addLineToPoint: CGPointMake(41.31, 58.23)];
    [starPath addLineToPoint: CGPointMake(41.73, 42.64)];
    [starPath addLineToPoint: CGPointMake(32.22, 30.27)];
    [starPath addLineToPoint: CGPointMake(47.18, 25.86)];
    [starPath closePath];
    
    [UIColor.grayColor setFill];
    [starPath fill];
}
```
代码生成的界面也很简单，就是一个灰色的星星。

![灰色星星.png](/images/2016-10-06/gray-star.webp)

星星已经得到了，现在我们想要只显示半颗星星，这样就可以实现我们想要的**0.5分**的效果了。那如何做呢？
办法当然有了，我们可以获取UIBezierPath对象的位置和尺寸，然后拿一个方块去遮住它的半边，那不就可以了。继续在自定义UIView的```- (void)drawRect:(CGRect)rect```方法后面添加代码：
```
    //获取path的尺寸和位置
    CGRect pathBounds = starPath.bounds;
    pathBounds.size.width = pathBounds.size.width * 0.5;
    UIBezierPath *rectPath = [UIBezierPath bezierPathWithRect:pathBounds];
    [UIColor.redColor setFill];
    [rectPath fill];
```

![遮住的星星.png](/images/2016-10-06/cover-star.webp)

如果把颜色改成白色的，那么半个星星已经达到了。

### 组合与裁剪

下面我们将通过一系列的魔法来实现星星，半边是红色的，半边是灰色的，这个魔法就是UIBezierPath对象的组合和裁剪方法。使用```- (void)appendPath:(UIBezierPath *)*bezierPath*```可以添加其他的UIBezierPath对象，而设置```usesEvenOddFillRule```就可以决定最终的路径是交集(Intersection)或是并集(Union)，从而达到我们想要的裁剪效果。

好了，我们继续在自定义UIView的```- (void)drawRect:(CGRect)rect```方法后面添加代码：
```
    [starPath appendPath:rectPath];
    [starPath setUsesEvenOddFillRule:YES];
    [starPath addClip];
    [[UIColor yellowColor] setFill];
    [starPath fill];
```

![第一次组合.png](/images/2016-10-06/first-combine-star.webp)

从上面的效果可以看出来，```usesEvenOddFillRule```的效果是将两个路径组合在一起并去除掉交集，所以这次的路径并不包括左边的半颗星星。我们如果用现在的路径继续去和rectPath去组合，就能得到右边的半颗星星了。

我们在```drawRect```方法中添加代码：
```
    [starPath appendPath:rectPath];
    [starPath setUsesEvenOddFillRule:YES];
    [starPath addClip];
    [[UIColor blueColor] setFill];
    [starPath fill];
```

![第二次组合.png](/images/2016-10-06/second-combine-star.webp)

经过第二次的组合操作， 我们可以看到效果已经非常接近想要的效果，我们现在可以试着把组合过程的红色和黄色代码都去掉，效果已经出来了啊。

![去掉组合颜色.png](/images/2016-10-06/filter-combine-color.webp)

算是已经达成目标了，我们刚开始的灰色替换成红色，把蓝色替换成灰色，就成了想要的效果了。我们也可以尝试把rectPath的宽度比例修改成其他比例，效果如下：

![宽度比例为0.3.png](/images/2016-10-06/rate-0.3.webp)
![宽度比例为0.5.png](/images/2016-10-06/rate-0.5.webp)
![宽度比例为0.8.png](/images/2016-10-06/rate-0.8.webp)

好啦，单个星星的比例已经搞定了，此时的自定义的UIView也可以其他背景颜色，都不会影响我们的星星形状了。下面就是最终的代码：
```
- (void)drawRect:(CGRect)rect {
    UIBezierPath *starPath = [UIBezierPath bezierPath];
    [starPath moveToPoint: CGPointMake(56, 13)];
    [starPath addLineToPoint: CGPointMake(64.82, 25.86)];
    [starPath addLineToPoint: CGPointMake(79.78, 30.27)];
    [starPath addLineToPoint: CGPointMake(70.27, 42.64)];
    [starPath addLineToPoint: CGPointMake(70.69, 58.23)];
    [starPath addLineToPoint: CGPointMake(56, 53)];
    [starPath addLineToPoint: CGPointMake(41.31, 58.23)];
    [starPath addLineToPoint: CGPointMake(41.73, 42.64)];
    [starPath addLineToPoint: CGPointMake(32.22, 30.27)];
    [starPath addLineToPoint: CGPointMake(47.18, 25.86)];
    [starPath closePath];
    
    [[UIColor redColor] setFill];
    [starPath fill];
    
    //获取path的尺寸和位置
    CGRect pathBounds = starPath.bounds;
    pathBounds.size.width = pathBounds.size.width * 0.8;
    UIBezierPath *rectPath = [UIBezierPath bezierPathWithRect:pathBounds];
    
    [starPath appendPath:rectPath];
    [starPath setUsesEvenOddFillRule:YES];
    [starPath addClip];

    [starPath appendPath:rectPath];
    [starPath setUsesEvenOddFillRule:YES];
    [starPath addClip];
    [[UIColor grayColor] setFill];
    [starPath fill];
}
```

### 平移和缩放

通过上一步的操作我们已经解决了星星不同比例的问题，但是还存在一个很大的问题，PaintCode生成的UIBezierPath是固定大小的，而我们所使用的控件需要通过frame或者constraint(约束)来改变大小的。不过UIBezierPath对象是矢量的，也就是说可以通过平移和缩放来适配UIView的大小。

这一次我们将通过复制PaintCode生成的UIBezierPath对象，然后通过平移和缩放操作来改变Path的位置，最后组合成5颗星星，而且大小可以随着UIView的改变而改变。

依然通过自定义UIView的```- (void)drawRect:(CGRect)rect```方法中添加一个灰色的星星：
```
- (void)drawRect:(CGRect)rect {
    UIBezierPath* starPath = [UIBezierPath bezierPath];
    [starPath moveToPoint: CGPointMake(56, 13)];
    [starPath addLineToPoint: CGPointMake(64.82, 25.86)];
    [starPath addLineToPoint: CGPointMake(79.78, 30.27)];
    [starPath addLineToPoint: CGPointMake(70.27, 42.64)];
    [starPath addLineToPoint: CGPointMake(70.69, 58.23)];
    [starPath addLineToPoint: CGPointMake(56, 53)];
    [starPath addLineToPoint: CGPointMake(41.31, 58.23)];
    [starPath addLineToPoint: CGPointMake(41.73, 42.64)];
    [starPath addLineToPoint: CGPointMake(32.22, 30.27)];
    [starPath addLineToPoint: CGPointMake(47.18, 25.86)];
    [starPath closePath];
    
    [UIColor.grayColor setFill];
    [starPath fill];
}
```
不过我们为了显示明显，讲UIView的背景颜色设置为橙色了。

![带背景颜色的灰色星星.png](/images/2016-10-06/gray-star-with-background.webp)

我们是通过UIBezierPath对象的```- (void)applyTransform:(CGAffineTransform)*transform*```方法来进行形变的，在```drawRect```方法中继续添加形变代码:
```
    //复制平移操作
    CGRect pathBounds = starPath.bounds;
    UIBezierPath *totalPath = [UIBezierPath bezierPath];
    for (int i = 0; i < 5; i ++) {
        UIBezierPath *copyPath = [UIBezierPath bezierPath];
        [copyPath appendPath:starPath];
        [copyPath applyTransform:CGAffineTransformMakeTranslation((pathBounds.size.width + 5.0)*i,0)];
        [totalPath appendPath:copyPath];
    }

    [[UIColor greenColor] setFill];
    [totalPath fill];
```
通过复制和平移动作，我们就得到了五个绿色的星星啦，并且把路径存在totalPath中。

![复制平移为五颗星星.png](/images/2016-10-06/copy-five-star.webp)

虽然我们得到了五颗星星，但是它还是大小固定的，而且星星的位置不是居中的，我们将通过缩放操作来适配大小。继续在复制平移的操作后面添加代码：
```
    //缩放操作    
    CGRect totalPathRect = totalPath.bounds;
    CGFloat scale;
    if (totalPathRect.size.width / totalPathRect.size.height >= rect.size.width / rect.size.height) {
        scale = rect.size.width / totalPathRect.size.width;
    }else {
        scale = rect.size.height / totalPathRect.size.height;
    }
    [totalPath applyTransform:CGAffineTransformMakeScale(scale,scale)];
```

![缩放操作之后的星星.png](/images/2016-10-06/scale-five-star.webp)

缩放操作之后的星星位置不对，我们需要通过平移对位置进行修正，同时把刚开始填充的灰色去掉。

```
    //修正origin的x，y值, 使path位置居中
    totalPathRect = totalPath.bounds;
    CGFloat x = (rect.size.width - totalPathRect.size.width) / 2.0;
    x = x - totalPathRect.origin.x;
    CGFloat y = (rect.size.height - totalPathRect.size.height) / 2.0;
    y = y - totalPathRect.origin.y;
    [totalPath applyTransform:CGAffineTransformMakeTranslation(x, y)];
```
修正后的效果如下：

![修正位置的效果.png](/images/2016-10-06/fix-scale-five-star.webp)

本文的代码可以在[https://github.com/aijun198600/AJScoreViewExample](https://github.com/aijun198600/AJScoreViewExample)查看。

下一篇文章我们将实现星星控件的封装，其中会使用CALayer的mask遮罩来实现控件。
如果需要使用评分控件的话可以直接去[https://github.com/aijun198600/AJScoreView](https://github.com/aijun198600/AJScoreView)下载使用，其中使用drawRect实现的方法在drawRect分支上。
