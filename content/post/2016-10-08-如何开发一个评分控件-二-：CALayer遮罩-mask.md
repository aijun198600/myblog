---
layout: post
title: 如何开发一个评分控件(二)：CALayer遮罩(mask)
date: 2016-10-08
tags: 
    - iOS
    - CALayer
categories: [ Code ]
---

![评分控件的最终效果.png](/images/2016-10-08/last-rate-component.webp)

> 前一篇文章已经把UIBezierPath对象的一些操作基础都做了说明，现在离最终效果只差最后一步了。需要了解前面文章的内容，可以点击链接[如何开发一个评分控件(一)](http://www.jianshu.com/p/ea74c10a03c2)进行查看。
如果只想直接使用已经封装好的控件，可以直接点击[https://github.com/aijun198600/AJScoreView](https://github.com/aijun198600/AJScoreView)使用。

下面开始正文吧， 我们将会使用两种不同的方法来实现，一种是在UIView的```drawRect```方法中直接绘制，一种是使用CAShapeLayer的```mask```来实现。

# 准备开始

首先我们需要先下载一张svg矢量图，比如twitter的小鸟图标，然后svg拖动到PaintCode里面，就可以自动生成UIBezierPath对象的代码了。

![PaintCode矢量图.png](/images/2016-10-08/paint-code-twitter.webp)

通过PaintCode生成的UIBezierPath对象代码，我们只需要UIBezierPath的对象，不需要颜色填充，整理后代码如下:

```
    UIBezierPath* path3611Path = [UIBezierPath bezierPath];
    [path3611Path moveToPoint: CGPointMake(93.72, 242.19)];
    [path3611Path addCurveToPoint: CGPointMake(267.68, 68.23) controlPoint1: CGPointMake(206.18, 242.19) controlPoint2: CGPointMake(267.68, 149.02)];
    [path3611Path addCurveToPoint: CGPointMake(267.5, 60.33) controlPoint1: CGPointMake(267.68, 65.58) controlPoint2: CGPointMake(267.62, 62.95)];
    [path3611Path addCurveToPoint: CGPointMake(298, 28.67) controlPoint1: CGPointMake(279.44, 51.7) controlPoint2: CGPointMake(289.82, 40.93)];
    [path3611Path addCurveToPoint: CGPointMake(262.89, 38.29) controlPoint1: CGPointMake(287.05, 33.54) controlPoint2: CGPointMake(275.26, 36.82)];
    [path3611Path addCurveToPoint: CGPointMake(289.78, 4.48) controlPoint1: CGPointMake(275.51, 30.72) controlPoint2: CGPointMake(285.2, 18.75)];
    [path3611Path addCurveToPoint: CGPointMake(250.95, 19.32) controlPoint1: CGPointMake(277.96, 11.48) controlPoint2: CGPointMake(264.88, 16.57)];
    [path3611Path addCurveToPoint: CGPointMake(206.32, 0) controlPoint1: CGPointMake(239.79, 7.43) controlPoint2: CGPointMake(223.91, 0)];
    [path3611Path addCurveToPoint: CGPointMake(145.18, 61.13) controlPoint1: CGPointMake(172.56, 0) controlPoint2: CGPointMake(145.18, 27.38)];
    [path3611Path addCurveToPoint: CGPointMake(146.76, 75.07) controlPoint1: CGPointMake(145.18, 65.93) controlPoint2: CGPointMake(145.71, 70.6)];
    [path3611Path addCurveToPoint: CGPointMake(20.74, 11.19) controlPoint1: CGPointMake(95.95, 72.52) controlPoint2: CGPointMake(50.89, 48.19)];
    [path3611Path addCurveToPoint: CGPointMake(12.46, 41.92) controlPoint1: CGPointMake(15.49, 20.23) controlPoint2: CGPointMake(12.46, 30.72)];
    [path3611Path addCurveToPoint: CGPointMake(39.67, 92.82) controlPoint1: CGPointMake(12.46, 63.13) controlPoint2: CGPointMake(23.25, 81.86)];
    [path3611Path addCurveToPoint: CGPointMake(11.98, 85.17) controlPoint1: CGPointMake(29.64, 92.51) controlPoint2: CGPointMake(20.21, 89.75)];
    [path3611Path addCurveToPoint: CGPointMake(11.97, 85.95) controlPoint1: CGPointMake(11.97, 85.43) controlPoint2: CGPointMake(11.97, 85.68)];
    [path3611Path addCurveToPoint: CGPointMake(61.02, 145.88) controlPoint1: CGPointMake(11.97, 115.56) controlPoint2: CGPointMake(33.04, 140.28)];
    [path3611Path addCurveToPoint: CGPointMake(44.9, 148.04) controlPoint1: CGPointMake(55.88, 147.28) controlPoint2: CGPointMake(50.48, 148.04)];
    [path3611Path addCurveToPoint: CGPointMake(33.41, 146.93) controlPoint1: CGPointMake(40.96, 148.04) controlPoint2: CGPointMake(37.13, 147.65)];
    [path3611Path addCurveToPoint: CGPointMake(90.52, 189.4) controlPoint1: CGPointMake(41.19, 171.23) controlPoint2: CGPointMake(63.76, 188.9)];
    [path3611Path addCurveToPoint: CGPointMake(14.58, 215.57) controlPoint1: CGPointMake(69.6, 205.8) controlPoint2: CGPointMake(43.23, 215.57)];
    [path3611Path addCurveToPoint: CGPointMake(-0, 214.72) controlPoint1: CGPointMake(9.66, 215.57) controlPoint2: CGPointMake(4.79, 215.29)];
    [path3611Path addCurveToPoint: CGPointMake(93.72, 242.19) controlPoint1: CGPointMake(27.06, 232.07) controlPoint2: CGPointMake(59.19, 242.19)];
    path3611Path.miterLimit = 4;
```
我们将这段代码复制到```drawRect```方法中，并将上一篇文章中的复制平移的代码复制过来:

```
    //复制平移操作
    CGRect pathBounds = path3611Path.bounds;
    UIBezierPath *totalPath = [UIBezierPath bezierPath];
    for (int i = 0; i < 5; i ++) {
        
        UIBezierPath *copyPath = [UIBezierPath bezierPath];
        [copyPath appendPath:path3611Path];
        [copyPath applyTransform:CGAffineTransformMakeTranslation((pathBounds.size.width + 5.0)*i,0)];
        
        [totalPath appendPath:copyPath];
    }
    
    //缩放操作
    CGRect totalPathRect = totalPath.bounds;
    CGFloat scale;
    if (totalPathRect.size.width / totalPathRect.size.height >= rect.size.width / rect.size.height) {
        scale = rect.size.width / totalPathRect.size.width;
    }else {
        scale = rect.size.height / totalPathRect.size.height;
    }
    [totalPath applyTransform:CGAffineTransformMakeScale(scale,scale)];
    
    //修正origin的x，y值, 使path位置居中
    totalPathRect = totalPath.bounds;
    CGFloat x = (rect.size.width - totalPathRect.size.width) / 2.0;
    x = x - totalPathRect.origin.x;
    CGFloat y = (rect.size.height - totalPathRect.size.height) / 2.0;
    y = y - totalPathRect.origin.y;
    [totalPath applyTransform:CGAffineTransformMakeTranslation(x, y)];
    
    [[UIColor greenColor] setFill];
    [totalPath fill];
```
和我们预期的一样，五只小鸟出来了！

![五只小鸟.png](/images/2016-10-08/five-bird.webp)

# DrawRect实现

现在我们继续和上一篇文章一样，画出3.5分的效果，即是让3.5只小鸟一种颜色，2.5只小鸟，中间的那只小鸟会有两种颜色，这和我们上一篇文章绘制半个星星的原理是一样的。

首先也是选择部分的正方形的UIBezierPath对象，盖住3.5只小鸟的地方，然后使用设置```setUsesEvenOddFillRule```为```YES```，反转两次就会得到右边剩下的UIBezierPath对象。

```
    CGFloat score = 3.5;
    NSInteger i = floor(score);
    CGFloat decimal = score - i;
    CGFloat scaleW = pathBounds.size.width * scale;
    CGFloat scalePadding = 5.0 * scale;
    CGFloat selectW = (scaleW + scalePadding) * i + scaleW * decimal;
    CGRect selectRect = CGRectMake(0, 0, selectW, rect.size.height);
    UIBezierPath *selectRectPath = [UIBezierPath bezierPathWithRect:selectRect];
    
    [totalPath appendPath:selectRectPath];
    [totalPath setUsesEvenOddFillRule:YES];
    [totalPath addClip];
    
    [totalPath appendPath:selectRectPath];
    [totalPath setUsesEvenOddFillRule:YES];
    [totalPath addClip];
    [[UIColor grayColor] setFill];
    [totalPath fill];
```
其中```scaleW = pathBounds.size.width * scale```是缩放后的小鸟的frame大小，```5.0 * scale```是每个小鸟之间的间距。

![3.5只小鸟.png](/images/2016-10-08/five-bird-3.5.webp)

看起来我们的目标已经达到了，后续封装只要把```UIBezierPath```和```score```存起来就可以了，当```score```设置时，只要调用```[self setNeedDisplay]```强制重绘界面就可以了。

# CAShapeLayer实现
使用CAShapeLayer和mask来实现评分控件，代码很多地方都是一样的，因为都是使用同样的```UIBezierPath```对象来进行填充。

首先创建一个```AJScoreView```类继承于UIView，然后添加两个CAShapeLayer用来显示两种不同的颜色，然后添加一个CALayer作为选择颜色的maskLayer。因为这次我们需要根据UIView的frame来调整大小，因为我们这次把代码挪动到```layoutSubviews```方法中。下面就是我们新建类的m文件，其中包括初始的方法。
```
#import "AJScoreView.h"

@implementation AJScoreView {
    CAShapeLayer *unSelectedLayer;
    CAShapeLayer *selectedLayer;
    CALayer *maskLayer;
}

- (id)initWithCoder:(NSCoder *)aDecoder {
    self = [super initWithCoder:aDecoder];
    if(self != nil) {
        [self commonInit];
    }
    return self;
}

- (id)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if(self != nil) {
        [self commonInit];
    }
    return self;
}

#pragma mark - Data Init methods
- (void)commonInit {
    
    unSelectedLayer = [CAShapeLayer layer];
    selectedLayer = [CAShapeLayer layer];
    maskLayer = [CALayer layer];
    maskLayer.backgroundColor = [UIColor blackColor].CGColor;
    
    [self.layer addSublayer:unSelectedLayer];
    [self.layer addSublayer:selectedLayer];
    
    [self setNeedsLayout];
    
}

- (void)layoutSubviews {
    
    [super layoutSubviews];
    
    CGRect rect = self.bounds;
    
}

@end
```
我们把第一步生成五只小鸟的代码也复制到```layoutSubviews```方法的后面，然后把填充的颜色赋值进去。

```
    //确定背景颜色
    unSelectedLayer.frame = CGRectMake(0, 0, self.bounds.size.width, self.bounds.size.height);
    unSelectedLayer.path = totalPath.CGPath;
    unSelectedLayer.fillColor = [UIColor grayColor].CGColor;
    
    selectedLayer.frame = CGRectMake(0, 0, self.bounds.size.width, self.bounds.size.height);
    selectedLayer.path = totalPath.CGPath;
    selectedLayer.fillColor = [UIColor greenColor].CGColor;
```
运行之后的效果就和第一步是一样的，都是五只绿色的小鸟，不过实际绿色的layer底部还有一层灰色的layer。现在我们需要将绿色的layer设置一个蒙版，将底部的那部分灰色的layer露出来。
```
    //确定选择的区域
    CGFloat score = 3.5;
    NSInteger i = floor(score);
    CGFloat decimal = score - i;
    CGFloat scaleW = pathBounds.size.width * scale;
    CGFloat scalePadding = 5.0 * scale;
    CGFloat selectW = (scaleW + scalePadding) * i + scaleW * decimal;
    CGRect selectRect = CGRectMake(0, 0, selectW, rect.size.height);
    
    maskLayer.frame = selectRect;
    selectedLayer.mask = maskLayer;
```
像上面的代码一样，```selectRect```的计算的区域还是和drawRect部分是一样。

![DrawRect 上、CAShapeLayer下.png](/images/2016-10-08/on-draw-or-calayer.webp)

可以看出来两个效果是一样的。我们现在只需要将```score ```和```path```剥离出来就可以组成复用的控件了。两者的区别是，当```score```设置时，当使用DrawRect时需要调用```[self setNeedDisplay];```强制重绘界面，而使用CAShapeLayer时就需要调用```[self setNeedsLayout];```强制重新调整布局就好了。

下一章我们将讲解storyboard的支持，以及控件事件的支持。本文详细的代码可以查看[https://github.com/aijun198600/AJScoreViewExample](https://github.com/aijun198600/AJScoreViewExample)。
