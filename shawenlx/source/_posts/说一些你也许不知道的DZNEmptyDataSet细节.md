---
title: 说一些你也许不知道的DZNEmptyDataSet细节
date: 2016-12-07 22:54:00
tags: source_code
categories: source_code
---
### 前言：

​   关于这个框架，之所以分析它的源码，只是想知道这么几个问题：它是如何做到自动检测UITableView以及UICollectionView是否存在数据并且响应刷新界面，以及兼顾系统方法和框架的封装和处理技巧。让我们带着这些问题一起来看看源码吧。

------

### 关于框架

- Github源码地址: [https://github.com/dzenbot/DZNEmptyDataSet](https://github.com/dzenbot/DZNEmptyDataSet)
- 版本：1.8.1
- [点击该链接查看如何使用这个框架](http://cocoadocs.org/docsets/DZNEmptyDataSet/1.8.1/)

------

### 文件目录

- UIScrollView+EmptyDataSet.h
- UIScrollView+EmptyDataSet.m

------
<!--more-->

粗略地浏览下头文件，发现核心的部分主要是实现两个协议，分别为DZNEmptyDataSetSource和DZNEmptyDataSetDelegate。这两个协议中的协议方法均为@optional类型。

```objc
@interface UIScrollView (EmptyDataSet)

@property (nonatomic, weak) IBOutlet id <DZNEmptyDataSetSource> emptyDataSetSource;
@property (nonatomic, weak) IBOutlet id <DZNEmptyDataSetDelegate> emptyDataSetDelegate;
/** YES if any empty dataset is visible. */
@property (nonatomic, readonly, getter = isEmptyDataSetVisible) BOOL emptyDataSetVisible;

/**
 *调用UITableView或者UICollectionView的［-reloadData］方法便会相应此方法。
 *并且 当且仅当列表数据源为空的时候才会触发。
 */
- (void)reloadEmptyDataSet;
@end
```

------

### DZNEmptyDataSetSource

- 该协议主要作用于数据源为空时的**对空白界面元素的设置。**
- 其中包括对Title、description、image、imageTintColor、imageAnimation、buttonTitle、buttonImage等属性的设置。
- 该协议提供了一套配置的接口，既方便用户根据需求设置相应的样式，当然也提供了**自定义界面的接口。

```objc
/**
 * 设置默认空白界面处理视图的标题Title.
 * 若需要设置富文本，则返回时设置(NSAttributedString *)类型。
 */
- (NSAttributedString *)titleForEmptyDataSet:(UIScrollView *)scrollView;

/**
 * 设置默认空白界面处理视图的描述description文本。
 * 若需要设置富文本，则返回时设置(NSAttributedString *)类型。
 */
- (NSAttributedString *)descriptionForEmptyDataSet:(UIScrollView *)scrollView;

/**
 * 设置默认空白界面布局的图片。
 */
- (UIImage *)imageForEmptyDataSet:(UIScrollView *)scrollView;

/**
 * 设置默认空白界面布局图片的前景色，默认为nil.
 */
- (UIColor *)imageTintColorForEmptyDataSet:(UIScrollView *)scrollView;

/**
 * 设置默认空白界面图片的动画效果。
 */
- (CAAnimation *) imageAnimationForEmptyDataSet:(UIScrollView *) scrollView;

/**
 * 设置默认空白界面响应按钮的标题，通常我们可以设置为"重新加载"等文本。
 * 如果需要显示不同的标题样式，可以返回富文本。
 * 并传入UIControlState进行设置。点击或者普通状态等。
 */
- (NSAttributedString *)buttonTitleForEmptyDataSet:(UIScrollView *)scrollView forState:(UIControlState)state;

/**
 * 设置默认空白界面响应按钮的图片。
 * 并传入UIControlState进行设置。点击或者普通状态等。
 */
- (UIImage *)buttonImageForEmptyDataSet:(UIScrollView *)scrollView forState:(UIControlState)state;

/**
 * 设置默认空白界面响应按钮的背景图片。默认不设置。
 * 并传入UIControlState进行设置。点击或者普通状态等。
 */
- (UIImage *)buttonBackgroundImageForEmptyDataSet:(UIScrollView *)scrollView forState:(UIControlState)state;

/**
 * 设置默认空白界面的背景颜色。默认为[UIColor clearColor]
 */
- (UIColor *)backgroundColorForEmptyDataSet:(UIScrollView *)scrollView;

/**
 * 设置默认空白界面的自定义视图View, View中可以高度自定义，包括按钮图片以及标题等元素。
 * 并传入UIControlState进行设置。点击或者普通状态等。
 * 返回自定义视图，将会忽略以下方法的配置。
 * -offsetForEmptyDataSet 和 -spaceHeightForEmptyDataSet
 */
- (UIView *)customViewForEmptyDataSet:(UIScrollView *)scrollView;

/**
 * 设置界面的垂直和水平方向的对齐约束， 默认为CGPointZero
 */
- (CGPoint)offsetForEmptyDataSet:(UIScrollView *)scrollView DEPRECATED_MSG_ATTRIBUTE("Use -verticalOffsetForEmptyDataSet:");
- (CGFloat)verticalOffsetForEmptyDataSet:(UIScrollView *)scrollView;

/**
 * 设置界面元素的垂直间距，默认为11px。
 */
- (CGFloat)spaceHeightForEmptyDataSet:(UIScrollView *)scrollView;
```

**开个小玩笑，我从未使用过该框架自带的样式，多数时候我们的需求还是以自定义为主，当然啦，这并不影响我们阅读源码，带着愉快地心情解读一下该框架优秀的地方也是蛮不错的。**

------

### DZNEmptyDataSetDelegate

- 该协议主要作用于处理该空白界面的代理。用于**获取代理的响应回调。**

```objc
/**
 * 实现该方法告诉代理EmptyDataSetView显示时以淡入的模式，默认为YES。
 */
- (BOOL)emptyDataSetShouldFadeIn:(UIScrollView *)scrollView;

/**
 * 实现该方法告诉代理EmptyDataSetView显示时应该被渲染。默认为YES。
 */
- (BOOL)emptyDataSetShouldDisplay:(UIScrollView *)scrollView;

/**
 * 实现该方法告诉代理该视图可以响应点击事件，默认为YES。
 */
- (BOOL)emptyDataSetShouldAllowTouch:(UIScrollView *)scrollView;

/**
 * 实现该方法告诉代理该视图允许滚动，默认为NO。
 */
- (BOOL)emptyDataSetShouldAllowScroll:(UIScrollView *)scrollView;

/**
 * 实现该方法告诉代理该视图中的图片允许执行动画，默认为NO。
 */
- (BOOL)emptyDataSetShouldAnimateImageView:(UIScrollView *)scrollView;

/**
 * 实现该方法告诉代理emptyDataSetView被点击
 * 使用该方法要么对textfield或者searchBar调用了resignFirstResponder方法。
 */
- (void)emptyDataSetDidTapView:(UIScrollView *)scrollView DEPRECATED_MSG_ATTRIBUTE("Use emptyDataSet:didTapView:");

/**
 * 实现该方法告诉代理，响应按钮点击事件被触发
 * @param scrollView 该滚动视图的子类实现了该方法。
 */
- (void)emptyDataSetDidTapButton:(UIScrollView *)scrollView DEPRECATED_MSG_ATTRIBUTE("Use emptyDataSet:didTapButton:");

/**
 * 实现该方法告诉代理empty dataset view被点击触发。
 * 使用该方法要么对textfield或者searchBar调用了resignFirstResponder方法。
 */
- (void)emptyDataSet:(UIScrollView *)scrollView didTapView:(UIView *)view;

/**
 * 实现该方法告诉代理，响应按钮点击事件被触发
 */
- (void)emptyDataSet:(UIScrollView *)scrollView didTapButton:(UIButton *)button;

/**
 * 实现该方法告诉代理，emptyDataView视图即将出现。
 */
- (void)emptyDataSetWillAppear:(UIScrollView *)scrollView;

/**
 * 实现该方法告诉代理，emptyDataView视图已经出现。
 */
- (void)emptyDataSetDidAppear:(UIScrollView *)scrollView;

/**
 * 实现该方法告诉代理，emptyDataView视图即将消失。
 */
- (void)emptyDataSetWillDisappear:(UIScrollView *)scrollView;

/**
 * 实现该方法告诉代理，emptyDataView视图已经消失。
 */
- (void)emptyDataSetDidDisappear:(UIScrollView *)scrollView;
```

不知道大家有没有注意到，**这些代理方法均以emptyDataSet作为方法前缀**，相信我们写UITableView的代理方法非常频繁吧，那你一定也能注意到这样一个**编程规范**，这样做的好处在于**我们可以利用自动补全提示的功能快速索引我们想要的方法**。这些细节还是有很多品味咀嚼的地方，必须引起我们的高度重视，这样才能写出更规范的代码。

------

看到这里，相信你对如何实现协议方法来实现你的目的已经不是大问题，然而这还远远不够。当我打开.m文件，猛然觉得接口方法仅是冰山一角，有一个更大的宝藏藏在实现文件中，继续细细品味。

```objc
@interface UIView (DZNConstraintBasedLayoutExtensions)
- (NSLayoutConstraint *)equallyRelatedConstraintWithView:(UIView *)view attribute:(NSLayoutAttribute)attribute;
@end

@interface DZNEmptyDataSetView : UIView
//...
@end

#pragma mark - UIScrollView+EmptyDataSet
static char const * const kEmptyDataSetSource =     "emptyDataSetSource";
static char const * const kEmptyDataSetDelegate =   "emptyDataSetDelegate";
static char const * const kEmptyDataSetView =       "emptyDataSetView";

#define kEmptyImageViewAnimationKey @"com.dzn.emptyDataSet.imageViewAnimation"

@interface UIScrollView () <UIGestureRecognizerDelegate>
@property (nonatomic, readonly) DZNEmptyDataSetView *emptyDataSetView;
@end
```

实现文件中，主要包含以上三个类。请"自动忽略"掉前两个类。无关紧要，主要功能是设置该框架的界面元素以及布局约束，代码也容易理解，自行打开框架源码查看，便不做赘述，着重记录介绍**UIScrollView+EmptyDataSet**这个分类的实现。

------

### UIScrollView+EmptyDataSet

先浏览下这个分类中的代码模块。阅读源码的时候应该从大方向入手，看看代码分块主要包含哪些模块，再逐一突破。**换言之，先找到入口，再慢慢探索！**

```objc
#pragma mark - Getters (Public)
#pragma mark - Getters (Private)
#pragma mark - Data Source Getters
#pragma mark - Delegate Getters & Events (Private)
#pragma mark - Setters (Public)
#pragma mark - Setters (Private)
#pragma mark - Reload APIs (Public)
#pragma mark - Reload APIs (Private)
#pragma mark - Method Swizzling
#pragma mark - UIGestureRecognizerDelegate Methods
```

#### #pragma mark - Getters (Public)

```objc
- (id<DZNEmptyDataSetSource>)emptyDataSetSource {
    return objc_getAssociatedObject(self, kEmptyDataSetSource);
}

- (id<DZNEmptyDataSetDelegate>)emptyDataSetDelegate {
    return objc_getAssociatedObject(self, kEmptyDataSetDelegate);
}

- (BOOL)isEmptyDataSetVisible {
    UIView *view = objc_getAssociatedObject(self, kEmptyDataSetView);
    return view ? !view.hidden : NO;
}
```

- 该模块主要通过runtime获取属性设置DZNEmptyDataSetSource，DZNEmptyDataSetDelegate 以及isEmptyDataSetVisible的属性getter方法。
- isEmptyDataSetVisible属性主要用于判断当前的EmptyDataSetView是否可见。这里的`!view.hidden`是返回YES的，默认初始化EmptyDataSetView的hidden为YES，默认不可见。在后面的setter方法中可以看到。

#### #pragma mark - Setters (Public)

```objc
- (void)setEmptyDataSetSource:(id<DZNEmptyDataSetSource>)datasource {
    if (!datasource || ![self dzn_canDisplay]) {
        [self dzn_invalidate];
    }
    
    objc_setAssociatedObject(self, kEmptyDataSetSource, datasource, OBJC_ASSOCIATION_ASSIGN);
    
    // 通过添加runtime替换原生的-reloadData方法的实现方法为-dzn_reloadData方法。
    [self swizzleIfPossible:@selector(reloadData)];
    
    // 特别注意的是对于UITableView, 我们也注入方法-dzn_reloadData到-endUpdates方法中。
    if ([self isKindOfClass:[UITableView class]]) {
        [self swizzleIfPossible:@selector(endUpdates)];
    }
}

- (void)setEmptyDataSetDelegate:(id<DZNEmptyDataSetDelegate>)delegate {
    if (!delegate) {
        [self dzn_invalidate];
    }
    objc_setAssociatedObject(self, kEmptyDataSetDelegate, delegate, OBJC_ASSOCIATION_ASSIGN);
}
```

- 这两个Setter方法，主要通过runtime对两个代理属性进行设置。
- `[self dzn_invalidate]`为移除视图的方法。
- `[self dzn_canDisplay]`为判断父视图是否为UITableView, UICollectionView以及UIScrollView。
- 关于方法如何替换交换将在下面分析。

#### #pragma mark - Getters (Private)

```objc
- (DZNEmptyDataSetView *)emptyDataSetView {
    DZNEmptyDataSetView *view = objc_getAssociatedObject(self, kEmptyDataSetView);
    if (!view) {
        view = [DZNEmptyDataSetView new];
        //...
        [self setEmptyDataSetView:view];
    }
    return view;
}

- (BOOL)dzn_canDisplay {
    if (self.emptyDataSetSource && [self.emptyDataSetSource conformsToProtocol:@protocol(DZNEmptyDataSetSource)]) {
        if ([self isKindOfClass:[UITableView class]] || [self isKindOfClass:[UICollectionView class]] || [self isKindOfClass:[UIScrollView class]]) {
            return YES;
        }
    }
    return NO;
}

- (NSInteger)dzn_itemsCount {
    NSInteger items = 0;
    
    // UIScollView 没有响应 'dataSource' 方法，所以不进行统计
    if (![self respondsToSelector:@selector(dataSource)]) {
        return items;
    }
    // UITableView support
    if ([self isKindOfClass:[UITableView class]]) {
        UITableView *tableView = (UITableView *)self;
        id <UITableViewDataSource> dataSource = tableView.dataSource;
        NSInteger sections = 1;        
        if (dataSource && [dataSource respondsToSelector:@selector(numberOfSectionsInTableView:)]) {
            sections = [dataSource numberOfSectionsInTableView:tableView];
        }
        if (dataSource && [dataSource respondsToSelector:@selector(tableView:numberOfRowsInSection:)]) {
            for (NSInteger section = 0; section < sections; section++) {
                items += [dataSource tableView:tableView numberOfRowsInSection:section];
            }
        }
    }
    // UICollectionView support
    else if ([self isKindOfClass:[UICollectionView class]]) {
        //...类似于UITableView的处理方式。
    }
    return items;
}
```

- `(DZNEmptyDataSetView *)emptyDataSetView`通过runtime初始化mptyDataSetView，并设置了一些默认的属性参数
- `(BOOL)dzn_canDisplay`判断当前视图是否可以加载空白页面，当且仅当self实现了代理，以及为UITableView, UICollectionVIew和UIScrollView.
- `(NSInteger)dzn_itemsCount`统计self的dataSource的元素个数，通过`- numberOfSections `和 `- numberOfItemsInSection`方法进行统计。

#### #pragma mark - Setters (Private)

```objc
- (void)setEmptyDataSetView:(DZNEmptyDataSetView *)view{
    objc_setAssociatedObject(self, kEmptyDataSetView, view, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
```

- 通过runtime设置私有View。

#### #pragma mark - Data Source Getters

```objc
- (NSAttributedString *)dzn_titleLabelString;

- (NSAttributedString *)dzn_detailLabelString;

- (UIImage *)dzn_image; 

- (CAAnimation *)dzn_imageAnimation;

- (UIColor *)dzn_imageTintColor;

- (NSAttributedString *)dzn_buttonTitleForState:(UIControlState)state;

- (UIImage *)dzn_buttonImageForState:(UIControlState)state;

- (UIImage *)dzn_buttonBackgroundImageForState:(UIControlState)state;

- (UIColor *)dzn_dataSetBackgroundColor;

- (CGFloat)dzn_verticalOffset;

- (CGFloat)dzn_verticalSpace;

- (UIView *)dzn_customView {
    if (self.emptyDataSetSource && [self.emptyDataSetSource respondsToSelector:@selector(customViewForEmptyDataSet:)]) {
        UIView *view = [self.emptyDataSetSource customViewForEmptyDataSet:self];
        if (view) NSAssert([view isKindOfClass:[UIView class]], @"You must return a valid UIView object for -customViewForEmptyDataSet:");
        if (!self.isNotFirst) {
            self.isNotFirst = YES;
            return nil;
        }
        return view;
    }
    return nil;
}
```

- 该模块的方法主要通过DataSource的代理方法设置对应的系统默认的View的属性。
- `- dzn_customView`用于设置自定义视图，也是我们最常用的一个方法。同样也是通过协议方法获取对应的自定义视图。

#### #Delegate Getters & Events (Private)

```objc
- (BOOL)dzn_shouldFadeIn {
    //...
    return YES;
}

- (BOOL)dzn_shouldDisplay {
    //...
    return YES;
}

- (BOOL)dzn_isTouchAllowed {
    //...
    return YES;
}

- (BOOL)dzn_isScrollAllowed {
    //...
    return NO;
}

- (BOOL)dzn_isImageViewAnimateAllowed {
    if (self.emptyDataSetDelegate && [self.emptyDataSetDelegate respondsToSelector:@selector(emptyDataSetShouldAnimateImageView:)]) {
        return [self.emptyDataSetDelegate emptyDataSetShouldAnimateImageView:self];
    }
    return NO;
}

- (void)dzn_willAppear {
    if (self.emptyDataSetDelegate && [self.emptyDataSetDelegate respondsToSelector:@selector(emptyDataSetWillAppear:)]) {
        [self.emptyDataSetDelegate emptyDataSetWillAppear:self];
    }
}

- (void)dzn_didAppear {
    if (self.emptyDataSetDelegate && [self.emptyDataSetDelegate respondsToSelector:@selector(emptyDataSetDidAppear:)]) {
        [self.emptyDataSetDelegate emptyDataSetDidAppear:self];
    }
}

- (void)dzn_willDisappear {
    if (self.emptyDataSetDelegate && [self.emptyDataSetDelegate respondsToSelector:@selector(emptyDataSetWillDisappear:)]) {
        [self.emptyDataSetDelegate emptyDataSetWillDisappear:self];
    }
}

- (void)dzn_didDisappear {
    if (self.emptyDataSetDelegate && [self.emptyDataSetDelegate respondsToSelector:@selector(emptyDataSetDidDisappear:)]) {
        [self.emptyDataSetDelegate emptyDataSetDidDisappear:self];
    }
}

- (void)dzn_didTapContentView:(id)sender {
    if (self.emptyDataSetDelegate && [self.emptyDataSetDelegate respondsToSelector:@selector(emptyDataSet:didTapView:)]) {
        [self.emptyDataSetDelegate emptyDataSet:self didTapView:sender];
    }
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wdeprecated-declarations"
    else if (self.emptyDataSetDelegate && [self.emptyDataSetDelegate respondsToSelector:@selector(emptyDataSetDidTapView:)]) {
        [self.emptyDataSetDelegate emptyDataSetDidTapView:self];
    }
#pragma clang diagnostic pop
}

- (void)dzn_didTapDataButton:(id)sender {
    if (self.emptyDataSetDelegate && [self.emptyDataSetDelegate respondsToSelector:@selector(emptyDataSet:didTapButton:)]) {
        [self.emptyDataSetDelegate emptyDataSet:self didTapButton:sender];
    }
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wdeprecated-declarations"
    else if (self.emptyDataSetDelegate && [self.emptyDataSetDelegate respondsToSelector:@selector(emptyDataSetDidTapButton:)]) {
        [self.emptyDataSetDelegate emptyDataSetDidTapButton:self];
    }
#pragma clang diagnostic pop
}
```

- 以上这些方法，均是将代理协议方法封装成私有方法，当且仅当代理协议方法被实现的前提下，这些私有方法才会生效。
- 关于`pragma clang diagnostic`，是忽略系统编译警告。可以参考这篇博文。[http://nshipster.cn/clang-diagnostics/](http://nshipster.cn/clang-diagnostics/)

#### #pragma mark - Reload APIs

```objc
#pragma mark - Reload APIs (Public)
- (void)reloadEmptyDataSet {
    [self dzn_reloadEmptyDataSet];
}

#pragma mark - Reload APIs (Private)
- (void)dzn_reloadEmptyDataSet {
    if (![self dzn_canDisplay]) {
        return;
    }
  
    if ([self dzn_shouldDisplay] && [self dzn_itemsCount] == 0) {
        // 通知该空白视图即将出现
        [self dzn_willAppear];
        
        DZNEmptyDataSetView *view = self.emptyDataSetView;
        
        if (!view.superview) {
            // Send the view all the way to the back, in case a header and/or footer is present, as well as for sectionHeaders or any other content
            if (([self isKindOfClass:[UITableView class]] || [self isKindOfClass:[UICollectionView class]]) && self.subviews.count > 1) {
                [self insertSubview:view atIndex:0];
            } else {
                [self addSubview:view];
            }
        }
        // 重新设置视图元素
        [view prepareForReuse];
        
        UIView *customView = [self dzn_customView];
        // 判断是否存在自定义视图
        if (customView) {
            view.customView = customView;
        } else {
            //系统默认初始化配置
        }
        
        //..一些其它相关配置，包括布局，偏移和界面动画等。
        
        // 通知该空白视图已经初始化完成。
        [self dzn_didAppear];
    } else if (self.isEmptyDataSetVisible) {
        [self dzn_invalidate];
    }
}

- (void)dzn_invalidate {
    // 通知该视图即将消失
    [self dzn_willDisappear];
    
    if (self.emptyDataSetView) {
        [self.emptyDataSetView prepareForReuse];
        [self.emptyDataSetView removeFromSuperview];
        
        [self setEmptyDataSetView:nil];
    }
    
    self.scrollEnabled = YES;
    // 通知该视图已经消失
    [self dzn_didDisappear];
}
```

- dzn_reloadEmptyDataSet 方法，通过判断当前是否存在自定义视图，若有则替换为自定义的View，否则则根据上述的配置私有方法对视图进行配置。
- dzn_invalidate 方法则是在视图消失的时候，对父视图进行重新设置，以及移除该空白界面的数据源和子视图。

#### #pragma mark - Method Swizzling

```objc
static NSMutableDictionary *_impLookupTable;
static NSString *const DZNSwizzleInfoPointerKey = @"pointer";
static NSString *const DZNSwizzleInfoOwnerKey = @"owner";
static NSString *const DZNSwizzleInfoSelectorKey = @"selector";

void dzn_original_implementation(id self, SEL _cmd) {
    // 从查找表获取原始实现
    NSString *key = dzn_implementationKey(self, _cmd);
    
    NSDictionary *swizzleInfo = [_impLookupTable objectForKey:key];
    NSValue *impValue = [swizzleInfo valueForKey:DZNSwizzleInfoPointerKey];
    
    IMP impPointer = [impValue pointerValue];
    
    //然后注入额外的实现重新加载空数据集
    //在调用原始实现之前，确实按时更新“isEmptyDataSetVisible”标志。
    [self dzn_reloadEmptyDataSet];
    
    // 如果找到，调用原始实现
    if (impPointer) {
        ((void(*)(id,SEL))impPointer)(self,_cmd);
    }
}

NSString *dzn_implementationKey(id target, SEL selector) {
    if (!target || !selector) {
        return nil;
    }
    
    Class baseClass;
    if ([target isKindOfClass:[UITableView class]]) baseClass = [UITableView class];
    else if ([target isKindOfClass:[UICollectionView class]]) baseClass = [UICollectionView class];
    else if ([target isKindOfClass:[UIScrollView class]]) baseClass = [UIScrollView class];
    else return nil;
    
    NSString *className = NSStringFromClass([baseClass class]);
    
    NSString *selectorName = NSStringFromSelector(selector);
    return [NSString stringWithFormat:@"%@_%@",className,selectorName];
}

- (void)swizzleIfPossible:(SEL)selector {
    // 检查目标是否响应selector
    if (![self respondsToSelector:selector]) {
        return;
    }
    
    // 创建查找表
    if (!_impLookupTable) {
        _impLookupTable = [[NSMutableDictionary alloc] initWithCapacity:2];
    }
    
    // 我们确保每个UITableView或UICollectionView的setImplementation方法，被调用一次。
    for (NSDictionary *info in [_impLookupTable allValues]) {
        Class class = [info objectForKey:DZNSwizzleInfoOwnerKey];
        NSString *selectorName = [info objectForKey:DZNSwizzleInfoSelectorKey];
        
        if ([selectorName isEqualToString:NSStringFromSelector(selector)]) {
            if ([self isKindOfClass:class]) {
                return;
            }
        }
    }
    
    NSString *key = dzn_implementationKey(self, selector);
    NSValue *impValue = [[_impLookupTable objectForKey:key] valueForKey:DZNSwizzleInfoPointerKey];
    
    // 如果这个类的实现已经存在，跳过！
    if (impValue || !key) {
        return;
    }
    
    // 通过Swizzle注入额外的实现
    Method method = class_getInstanceMethod([self class], selector);
    IMP dzn_newImplementation = method_setImplementation(method, (IMP)dzn_original_implementation);
    
    // 将新实现存储在查找表中
    NSDictionary *swizzledInfo = @{DZNSwizzleInfoOwnerKey: [self class],
                                   DZNSwizzleInfoSelectorKey: NSStringFromSelector(selector),
                                   DZNSwizzleInfoPointerKey: [NSValue valueWithPointer:dzn_newImplementation]};
    
    [_impLookupTable setObject:swizzledInfo forKey:key];
}
```

- 此处有一篇作者推荐的关于method swizzing技术的博文: [The Right Way to Swizzle in Objective-C](https://blog.newrelic.com/2014/04/16/right-way-to-swizzle/)
- dzn_original_implementation方法用于调用原始实现方法
- dzn_implementationKey方法获取交换方法的方法名，用过类名以及对应方法名进行封装，防止交换错方法，因为界面中可能存在多个需要空白视图的父视图reloadData方法，避免交换错误。这个设计细节值得学习。
- swizzleIfPossible:(SEL)selector方法用于判断当前方法是否允许交换，比如父视图是否存在并且为约定的类型，以及是否调用了需要交换的方法。
- **值得着重注意的一个比较有意思的地方，也是我第一次看到这样的交换的地方，就是该方法中，通过一个字典作为一个查找表，确保当存在多个空白视图时，每个UITableView 或者 UICollectionView的原始方法只被交换了一次，避免了重复交换导致bug。这个细节处理应该是这个框架的精髓了！**

#### #pragma mark - UIGestureRecognizerDelegate Methods

```objc
- (BOOL)gestureRecognizerShouldBegin:(UIGestureRecognizer *)gestureRecognizer
{
    if ([gestureRecognizer.view isEqual:self.emptyDataSetView]) {
        return [self dzn_isTouchAllowed];
    }
    return [super gestureRecognizerShouldBegin:gestureRecognizer];
}

- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldRecognizeSimultaneouslyWithGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer
{
    UIGestureRecognizer *tapGesture = self.emptyDataSetView.tapGesture;
    
    if ([gestureRecognizer isEqual:tapGesture] || [otherGestureRecognizer isEqual:tapGesture]) {
        return YES;
    }
    
    // defer to emptyDataSetDelegate's implementation if available
    if ( (self.emptyDataSetDelegate != (id)self) && [self.emptyDataSetDelegate respondsToSelector:@selector(gestureRecognizer:shouldRecognizeSimultaneouslyWithGestureRecognizer:)]) {
        return [(id)self.emptyDataSetDelegate gestureRecognizer:gestureRecognizer shouldRecognizeSimultaneouslyWithGestureRecognizer:otherGestureRecognizer];
    }
    return NO;
}
```

- 该方法设置该空白视图的响应手势，通过返回协议方法中定义的手势响应回调。

------

### 最后

通过源码的分析，我也解决了开头提出的问题。该框架通过分类扩展，以及代理协议的方式，达到了监听视图是否该在没有数据源的情况下显示空白视图。不过个人觉得，框架中仍有一些代码可以写得稍微精简些，比如一些判断可以封装一下。这样会少些一些重复的代码。哈哈，仅仅是吹毛求疵罢了。总体这个框架还是非常赞的，使用该框架，也完善了一些用户体验，值得推荐。
