---
title: 变色龙：CAGradientLayer
date: 2016-12-15 16:03:16
tags: iOS
categories: iOS
---

### 项目需求

​   最近项目中需要绘制一根颜色渐变的挂线，当然也可以让UI设计做图，不过考虑到如果渐变色采用一种会变化的展示效果也许会更加炫酷些。绘制这根挂线，CAGradientLayer是不二之选。

------

### 关于CAGradientLayer

​   CAGradientLayer可以用来生成两种或更多颜色平滑渐变的Layer。使用Layer层的绘制，有一个很明显的优势就是：**绘制过程采用了硬件加速**。

------
<!--more-->

### CAGradientLayer的相关属性

```objc
@interface CAGradientLayer : CALayer

/* 存放每个渐变分割点的CGColorRef对象数组. 数组默认为nil. 可支持动画. */

@property(nullable, copy) NSArray *colors;

/* 一个可选的NSNumber对象数组，定义每个对象的位置梯度分割点作为范围[0,1]中的值。 
 * 值必须为单调递增的。 如果给出一个nil数组，分割点在[0,1]范围内均匀分布。 
 * 当呈现时，在颜色被映射到输出颜色空间之前插值。 数组默认为nil。 支持动画。 
 * locations数组并不是强制要求的，但是如果你给它赋值了就一定要确保locations的数组大小和colors数组大小一定要相同，否则你将会得到一个空白的渐变。
 */
@property(nullable, copy) NSArray<NSNumber *> *locations;

/* 绘制到图层中时渐变的开始点和结束点的坐标空间。 
 * 起始点对应于第一梯度停止，终点到最后一个梯度停止。 
 * 两点都是在单位坐标空间中定义，然后映射到图层的边界矩形。 （即[0,0]是左下角的，[1,1]是右上角。）
 * 默认值分别是[.5,0]和[.5,1]。 默认支持的是竖直方向。两者都支持动画。 
 */

@property CGPoint startPoint;
@property CGPoint endPoint;

/* 将绘制的渐变的种类。 目前唯一允许value是`axial'（默认值）。 */

@property(copy) NSString *type;

@end
  
/** `type' values. **/

CA_EXTERN NSString * const kCAGradientLayerAxial
    CA_AVAILABLE_STARTING (10.6, 3.0, 9.0, 2.0);
NS_ASSUME_NONNULL_END
```



### CAGradientLayer坐标系

​   渐变色的作用范围，变化梯度的方向，颜色变换的作用点都和CAGradientLayer的坐标系统有关。根据下图的坐标，设定好起点和终点，渐变色的方向就会根据起点指向终点的方向来渐变了。

![loc](/images/loc.png)



- CAGradientLayer的坐标系统是从（0，0）到（1，1）绘制的矩形
- CAGradientLayer的frame值的size不为正方形的话，坐标系统会被拉伸
- CAGradientLayer的startPoint和endPoint会直接决定颜色的绘制方向
- CAGradientLayer的颜色分割点时以0到1的比例来计算的



### CAGradientLayer渐变方向

```objc
@property CGPoint startPoint;
@property CGPoint endPoint;
```

- 可以通过设置startPoint 以及 endPoint来设置渐变的方向，可以是水平方向的，也可以是竖直方向的，当然也可以沿着对角线进行渐变。系统默认支持的是竖直方向。

------

### 颜色渐变的水平线

- 通过CAGradientLayer 绘制一根颜色渐变的水平线。并通过CABasicAnimation执行颜色移动的动画。

```objc
#import "ViewController.h"

#define SCREEN_WIDTH [UIScreen mainScreen].bounds.size.width

@interface ViewController () <CAAnimationDelegate>

@property (nonatomic, strong) CAGradientLayer   *lineLayer;
@property (nonatomic, strong) UIView            *lineView;
@property (nonatomic, strong) NSMutableArray    *colorsArray;

@property (nonatomic, strong) CABasicAnimation  *gradientAnimation;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor blackColor];
    [self.view addSubview:self.lineView];
    [self.lineLayer addAnimation:self.gradientAnimation forKey:@"gradientAnimate"];
}

- (void)configAnimation {
    //将颜色数组中的最后一个元素加入到第一个元素中，达到颜色移动的效果。
    id lastColor = self.colorsArray.lastObject;
    [self.colorsArray removeLastObject];
    [self.colorsArray insertObject:lastColor atIndex:0];
    self.lineLayer.colors = self.colorsArray;
    [self.lineLayer addAnimation:self.gradientAnimation forKey:@"gradientAnimate"];
}

#pragma mark - <CAAnimationDelegate>
- (void)animationDidStop:(CAAnimation *)anim finished:(BOOL)flag {
    [self configAnimation];
}

#pragma mark - getter 
- (CAGradientLayer *)lineLayer {
    if (!_lineLayer) {
        _lineLayer = [CAGradientLayer layer];
        _lineLayer.frame = self.lineView.frame;
        _lineLayer.startPoint = CGPointMake(0.0, 0.5);
        _lineLayer.endPoint = CGPointMake(1.0, 0.5);
        _lineLayer.colors = self.colorsArray;
    }
    return _lineLayer;
}

- (NSMutableArray *)colorsArray {
    if (!_colorsArray) {
        _colorsArray = [NSMutableArray array];
        for (NSInteger hue = 0; hue != 360; hue += 5) {
            UIColor *color = [UIColor colorWithHue:1.0 * hue / 360.0
                                                saturation:1.0
                                                brightness:1.0
                                                     alpha:1.0];
            [_colorsArray addObject:(__bridge id)[color CGColor]];
        }
    }
    return _colorsArray;
}

- (UIView *)lineView {
    if (!_lineView) {
        _lineView = [[UIView alloc] initWithFrame:CGRectMake(0, 120, SCREEN_WIDTH, 1)];
        [_lineView.layer addSublayer:self.lineLayer];
    }
    return _lineView;
}

- (CABasicAnimation *)gradientAnimation {
    if (!_gradientAnimation) {
        _gradientAnimation = [CABasicAnimation animationWithKeyPath:@"colorsAnimation"];
        _gradientAnimation.delegate = self;
        _gradientAnimation.duration = 0.06;
        _gradientAnimation.toValue = self.colorsArray;
        [_gradientAnimation setRemovedOnCompletion:YES];
        _gradientAnimation.fillMode = kCAFillModeForwards;
    }
    return _gradientAnimation;
}

@end
```



### 代码效果图

![loc](/images/gradientLayerLine.gif)

------

### 总结

- 其实这个小实验只是个人的自娱自乐。对于Layer的学习仍需要不断积累。多数复杂的动画都是基于简单动画的组合和扩展。
- 另外，有一个不错的学习教材。[iOS Core Animation](https://zsisme.gitbooks.io/ios-/content/chapter6/cagradientLayer.html)

