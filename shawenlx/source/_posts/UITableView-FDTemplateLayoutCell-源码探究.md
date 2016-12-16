---
title: UITableView+FDTemplateLayoutCell 源码探究
date: 2016-11-27 13:53:00
tags: source_code
categories: source_code
---
## UITableView+FDTemplateLayoutCell 源码探究

------

- 在我们日常的业务中，常常伴随大量的UITableView，然而动态地计算Cell的高度常常困扰着我。自从使用了这个组件之后，一切都变得没那么复杂。所以深入学习下这个框架的组件的实现原理。
- 框架地址：[https://github.com/forkingdog/UITableView-FDTemplateLayoutCell](https://github.com/forkingdog/UITableView-FDTemplateLayoutCell)

------

### 代码文件目录

```objc
- UITableView+FDIndexPathHeightCache.h
- UITableView+FDIndexPathHeightCache.m
- UITableView+FDKeyedHeightCache.h
- UITableView+FDKeyedHeightCache.m
- UITableView+FDTemplateLayoutCell.h
- UITableView+FDTemplateLayoutCell.m
- UITableView+FDTemplateLayoutCellDebug.h
- UITableView+FDTemplateLayoutCellDebug.m
```

------
<!--more-->
首先，介绍一下这几个类的基本功能，再层层推进，逐一分析。

```objc
- UITableView+FDIndexPathHeightCache，主要负责cell通过NSIndexPath进行缓存高度的功能
- UITableView+FDKeyedHeightCache，主要负责cell通过key值进行缓存高度的功能
- UITableView+FDTemplateLayoutCell，提供接口方法方便用户定义cell的数据源，以及帮助我们计算cell的高度
- UITableView+FDTemplateLayoutCellDebug，提供一些Debug打印信息
```

------

关于这个框架，坦白说，从代码中看，作者无疑秀了一波runtime底层的功底，让我这种小白起初一脸懵逼。自然我得换种思路来解读这个框架，那就是从字数最少的类入手吧。

### UITableView+FDTemplateLayoutCellDebug

```objc
@interface UITableView (FDTemplateLayoutCellDebug)

//设置Debug模式是否打开
@property (nonatomic, assign) BOOL fd_debugLogEnabled;

//通过该方法，传递NSLog打印对应的Debug信息
- (void)fd_debugLog:(NSString *)message;

@end
```

```objc
@implementation UITableView (FDTemplateLayoutCellDebug)

- (BOOL)fd_debugLogEnabled {
    return [objc_getAssociatedObject(self, _cmd) boolValue];
}

- (void)setFd_debugLogEnabled:(BOOL)debugLogEnabled {
    objc_setAssociatedObject(self, @selector(fd_debugLogEnabled), @(debugLogEnabled), OBJC_ASSOCIATION_RETAIN);
}

- (void)fd_debugLog:(NSString *)message {
    if (self.fd_debugLogEnabled) {
        NSLog(@"** FDTemplateLayoutCell ** %@", message);
    }
}

@end
```

- 在分类中，如果要声明属性，可以通过使用关联度对象( AssociatedObject ), 通过objc_setAssociatedObject() 添加属性，objc_getAssociatedObject() 获取属性。实际上，相当于在运行时系统中动态地在内存中开辟一块空间，存储debugLogEnabled这个BOOL变量，类似懒加载的方式，通过runtime实现setter & getter方法。
- 关于runtime的知识点，推荐这篇博客:[http://yulingtianxia.com/blog/2014/11/05/objective-c-runtime/](http://yulingtianxia.com/blog/2014/11/05/objective-c-runtime/)

------

### UITableView+FDKeyedHeightCache

```objc
#import <UIKit/UIKit.h>

@interface FDKeyedHeightCache : NSObject

//判断缓存中是否存在key为值的缓存高度
- (BOOL)existsHeightForKey:(id<NSCopying>)key;

//对指定key的cell设置高度为height
- (void)cacheHeight:(CGFloat)height byKey:(id<NSCopying>)key;

//从缓存中获取对应key的cell的高度height值
- (CGFloat)heightForKey:(id<NSCopying>)key;

//从缓存中删除指定key的cell的值
- (void)invalidateHeightForKey:(id<NSCopying>)key;

//移除缓存中所有的cell的高度缓存值
- (void)invalidateAllHeightCache;
@end

@interface UITableView (FDKeyedHeightCache)
@property (nonatomic, strong, readonly) FDKeyedHeightCache *fd_keyedHeightCache;
@end
```

- 先来看看FDKeyedHeightCache类中声明的属性

```objc
@property (nonatomic, strong) NSMutableDictionary<id<NSCopying>, NSNumber *> *mutableHeightsByKeyForPortrait;

@property (nonatomic, strong) NSMutableDictionary<id<NSCopying>, NSNumber *> *mutableHeightsByKeyForLandscape;
```

不难看出，这是两个指定泛型的可变字典。

- mutableHeightsByKeyForPortrait : 用于缓存设备竖直放置时，对应key的cell的高度值。
- mutableHeightsByKeyForLandscape : 用于缓存设备横向放置时，对应key的cell的高度值。

------

- FDKeyedHeightCache中的接口方法

```objc
- (BOOL)existsHeightForKey:(id<NSCopying>)key {
    NSNumber *number = self.mutableHeightsByKeyForCurrentOrientation[key];
    return number && ![number isEqualToNumber:@-1];
}

- (void)cacheHeight:(CGFloat)height byKey:(id<NSCopying>)key {
    self.mutableHeightsByKeyForCurrentOrientation[key] = @(height);
}

- (CGFloat)heightForKey:(id<NSCopying>)key {
#if CGFLOAT_IS_DOUBLE
    return [self.mutableHeightsByKeyForCurrentOrientation[key] doubleValue];
#else
    return [self.mutableHeightsByKeyForCurrentOrientation[key] floatValue];
#endif
}

- (void)invalidateHeightForKey:(id<NSCopying>)key {
    [self.mutableHeightsByKeyForPortrait removeObjectForKey:key];
    [self.mutableHeightsByKeyForLandscape removeObjectForKey:key];
}

- (void)invalidateAllHeightCache {
    [self.mutableHeightsByKeyForPortrait removeAllObjects];
    [self.mutableHeightsByKeyForLandscape removeAllObjects];
}
```

- 这些方法并不晦涩，看到这里，大家不禁会问，self.mutableHeightsByKeyForCurrentOrientation从何而来，这也是我觉得这个类中，细节处理比较好的地方，由于此处考虑到缓存的高度区别了设备方向，所以框架作者，通过一个getter方法来获取对应的存放高度的字典。

```objc
- (NSMutableDictionary<id<NSCopying>, NSNumber *> *)mutableHeightsByKeyForCurrentOrientation {
    return UIDeviceOrientationIsPortrait([UIDevice currentDevice].orientation) ? self.mutableHeightsByKeyForPortrait: self.mutableHeightsByKeyForLandscape;
}
```

- 根据UIDeviceOrientationIsPortrait()函数，传入当前设备的放置方向（[UIDevice currentDevice].orientation

  ）进行判断。从而便可以通过属性简洁判断需要从那个字典中取值了。

------

### UITableView+FDIndexPathHeightCache

```objc
@interface FDIndexPathHeightCache : NSObject

// 如果您使用索引路径获取高度缓存，则自动启用
@property (nonatomic, assign) BOOL automaticallyInvalidateEnabled;

// Height cache
- (BOOL)existsHeightAtIndexPath:(NSIndexPath *)indexPath;
- (void)cacheHeight:(CGFloat)height byIndexPath:(NSIndexPath *)indexPath;
- (CGFloat)heightForIndexPath:(NSIndexPath *)indexPath;
- (void)invalidateHeightAtIndexPath:(NSIndexPath *)indexPath;
- (void)invalidateAllHeightCache;

@end

@interface UITableView (FDIndexPathHeightCache)
@property (nonatomic, strong, readonly) FDIndexPathHeightCache *fd_indexPathHeightCache;
@end

@interface UITableView (FDIndexPathHeightCacheInvalidation)
/// 当你不想通过删除缓存中的高度来刷新数据源重新计算时，可以调用这个方法。
/// 该方法中用过runtime重写了tableView中修改cell的一些方法，例如插入cell，删除cell，移动cell，以及reloadData方法。
- (void)fd_reloadDataWithoutInvalidateIndexPathHeightCache;
@end
```

- 首先看看FDIndexPathHeightCache中设置的属性

```objc
typedef NSMutableArray<NSMutableArray<NSNumber *> *> FDIndexPathHeightsBySection;

@interface FDIndexPathHeightCache ()
@property (nonatomic, strong) FDIndexPathHeightsBySection *heightsBySectionForPortrait;
@property (nonatomic, strong) FDIndexPathHeightsBySection *heightsBySectionForLandscape;
@end
```

通过前面key的高度缓存分析，不难猜出这几个属性是干什么的。

- 由于通过NSIndexPath获取高度缓存，NSIndexPath对应section, 以及indexPath。FDIndexPathHeightsBySection这个数组，通过数组嵌套字典的数据结构来存储，不同的section组中对应的cell的高度缓存。

------

- FDIndexPathHeightCache中的方法

  由于头文件声明的几个接口方法，与FDKeyedHeightCache中的思路类似，就不再费口舌了，大家翻看源码便一目了然。

```objc
- (void)enumerateAllOrientationsUsingBlock:(void (^)(FDIndexPathHeightsBySection *heightsBySection))block {
    block(self.heightsBySectionForPortrait);
    block(self.heightsBySectionForLandscape);
}

- (void)invalidateHeightAtIndexPath:(NSIndexPath *)indexPath {
    [self buildCachesAtIndexPathsIfNeeded:@[indexPath]];
    [self enumerateAllOrientationsUsingBlock:^(FDIndexPathHeightsBySection *heightsBySection {
        heightsBySection[indexPath.section][indexPath.row] = @-1;
    }];
}

- (void)invalidateAllHeightCache {
    [self enumerateAllOrientationsUsingBlock:^(FDIndexPathHeightsBySection *heightsBySection) {
        [heightsBySection removeAllObjects];
    }];
}

- (void)buildCachesAtIndexPathsIfNeeded:(NSArray *)indexPaths {
    // Build every section array or row array which is smaller than given index path.
    [indexPaths enumerateObjectsUsingBlock:^(NSIndexPath *indexPath, NSUInteger idx, BOOL *stop) {
        [self buildSectionsIfNeeded:indexPath.section];
        [self buildRowsIfNeeded:indexPath.row inExistSection:indexPath.section];
    }];
}

- (void)buildSectionsIfNeeded:(NSInteger)targetSection {
    [self enumerateAllOrientationsUsingBlock:^(FDIndexPathHeightsBySection *heightsBySection) {
        for (NSInteger section = 0; section <= targetSection; ++section) {
            if (section >= heightsBySection.count) {
                heightsBySection[section] = [NSMutableArray array];
            }
        }
    }];
}

- (void)buildRowsIfNeeded:(NSInteger)targetRow inExistSection:(NSInteger)section {
    [self enumerateAllOrientationsUsingBlock:^(FDIndexPathHeightsBySection *heightsBySection) {
        NSMutableArray<NSNumber *> *heightsByRow = heightsBySection[section];
        for (NSInteger row = 0; row <= targetRow; ++row) {
            if (row >= heightsByRow.count) {
                heightsByRow[row] = @-1;
            }
        }
    }];
}

```

- 这几个封装的方法，主要一点就是通过block来回调，判断删除NSIndexPath对应的cell高度缓存。

------

- 在这个类中，最核心的莫过于UITableView (FDIndexPathHeightCacheInvalidation) 这个分类的实现细节，废话少说，继续看代码。

```objc
//我们只是转发主调用，在崩溃报告中，最顶层的方法在堆栈可能存在FD，
//但它真的不是我们的错误，你应该检查你的表视图的数据源和
//重新加载时显示单元格不匹配。
static void __FD_TEMPLATE_LAYOUT_CELL_PRIMARY_CALL_IF_CRASH_NOT_OUR_BUG__(void (^callout)(void)) {
    callout();
}
#define FDPrimaryCall(...) do {__FD_TEMPLATE_LAYOUT_CELL_PRIMARY_CALL_IF_CRASH_NOT_OUR_BUG__(^{__VA_ARGS__});} while(0)
```

- 调用的接口方法

```objc
- (void)fd_reloadDataWithoutInvalidateIndexPathHeightCache {
    FDPrimaryCall([self fd_reloadData];);
}
```

- 这个方法，主要调用的是［self fd_reloadData］，看到这里的时候，我们的第一反应应该是此处通过runtime 交换了系统方法的实现。这是一种动态的拦截技巧，也算是基础的runtime知识了，懵逼的小伙伴可以认真阅读下前面提到的关于runtime的大牛博文。

------

- 既然如此，先来看看作者重写了哪些系统的方法吧。

```objc
+ (void)load {
    // All methods that trigger height cache's invalidation
    SEL selectors[] = {
        @selector(reloadData),
        @selector(insertSections:withRowAnimation:),
        @selector(deleteSections:withRowAnimation:),
        @selector(reloadSections:withRowAnimation:),
        @selector(moveSection:toSection:),
        @selector(insertRowsAtIndexPaths:withRowAnimation:),
        @selector(deleteRowsAtIndexPaths:withRowAnimation:),
        @selector(reloadRowsAtIndexPaths:withRowAnimation:),
        @selector(moveRowAtIndexPath:toIndexPath:)
    };
    
    for (NSUInteger index = 0; index < sizeof(selectors) / sizeof(SEL); ++index) {
        SEL originalSelector = selectors[index];
        SEL swizzledSelector = NSSelectorFromString([@"fd_" stringByAppendingString:NSStringFromSelector(originalSelector)]);
        Method originalMethod = class_getInstanceMethod(self, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(self, swizzledSelector);
        method_exchangeImplementations(originalMethod, swizzledMethod);
    }
}
```

- 通过method_exchangeImplementations() C函数， 将重写的方法，一一交换成重写的方法。
- 在这些fd_方法中的实现细节中，需要注意的一点就是，如果对应的fd_indexPathHeightCache设置了automaticallyInvalidateEnabled属性为YES时，对应的方法对高度缓存做相应的处理，重新更新fd_indexPathHeightCache中存储的高度缓存。
- 当第一次reloadData，或者cell的行数发生变化（增减行，section) ，会先在tableview不处于滚动状态的时候异步计算那些没有被计算过的cell的高度，做预缓存，这个想法非常赞。
- 使用者需要小心，这些调用是异步的, tableview delegate有可能会在预缓存计算的时候不存在了，导致程序崩溃，所以使用者在tableview需要析构的时候，在对应的tableview controller的dealloc中讲self.tableview.delegate = nil；，确保delegate后续不会是一个野指针。

------

### UITableView+FDTemplateLayoutCell

至此，我们已经分析了几个子类的实现逻辑，唯一剩下一个分类，也是我们使用这个框架的入口 **FDTemplateLayoutCell**分类。全面了解这个组件近在咫尺。

```objc
@interface UITableView (FDTemplateLayoutCell)

/* 为给定的重用标识符访问内部模板布局单元格。
 * 一般来说，你不需要知道这些模板布局单元格。
 * @param identifier重用必须注册的单元格的标识符。
*/ 
- (__kindof UITableViewCell *)fd_templateCellForReuseIdentifier:(NSString *)identifier;

/* 返回由重用标识符指定并配置的类型的单元格的高度, 并通过block来配置。
 * 单元格将被放置在固定宽度，垂直扩展的基础上，相对于其动态内容，使用自动布局。 
 * 因此，这些必要的单元被设置为自适应，即其内容总是确定它的宽度给定的宽度等于tableview的宽度。
 * @param identifier用于检索和维护模板的字符串标识符cell通过系统方法
 * '- dequeueReusableCellWithIdentifier：'
 * @param configuration用于配置和提供内容的可选块到模板单元格。 
 * 配置应该是最小的滚动性能足以计算单元格的高度。
*/
- (CGFloat)fd_heightForCellWithIdentifier:(NSString *)identifier configuration:(void (^)(id cell))configuration;

/* 计算的高度将通过其索引路径进行高速缓存，当需要时返回高速缓存的高度，因此，可以节省大量额外的高度计算。
 * 无需担心数据源更改时使缓存高度无效，它将在调用“-reloadData”或任何触发方法时自动完成UITableView的重新加载。
 * @param indexPath此单元格的高度缓存所属的位置。
*/
- (CGFloat)fd_heightForCellWithIdentifier:(NSString *)identifier cacheByIndexPath:(NSIndexPath *)indexPath configuration:(void (^)(id cell))configuration;

/* 此方法通过模型实体的标识符缓存高度。
 * 如果你的模型改变，调用“-invalidateHeightForKey:(id <NSCopying>)key”到无效缓存并重新计算，它比“cacheByIndexPath”方便得多。
 * @param key model entity的标识符，其数据配置一个单元格。
*/ 
- (CGFloat)fd_heightForCellWithIdentifier:(NSString *)identifier cacheByKey:(id<NSCopying>)key configuration:(void (^)(id cell))configuration;

@end

@interface UITableView (FDTemplateLayoutHeaderFooterView)

/* 返回在具有重用标识符的表视图中注册的Header或Footer视图的高度。
 * 在调用“ - [UITableView registerNib / Class：forHeaderFooterViewReuseIdentifier]”之后使用它，
 * 与“-fd_heightForCellWithIdentifier：configuration：”相同。
 * 它将调用“-sizeThatFits：”
 * UITableViewHeaderFooterView的子类不使用自动布局。
*/ 
- (CGFloat)fd_heightForHeaderFooterViewWithIdentifier:(NSString *)identifier configuration:(void (^)(id headerFooterView))configuration;

@end

@interface UITableViewCell (FDTemplateLayoutCell)
/* 指示这是仅用于计算的模板布局单元格。
 * 当配置单元格时，如果有非UI的副作用，你可能需要这个。
 * 类似:
 *   - (void)configureCell:(FooCell *)cell atIndexPath:(NSIndexPath *)indexPath {
 *       cell.entity = [self entityAtIndexPath:indexPath];
 *       if (!cell.fd_isTemplateLayoutCell) {
 *           [self notifySomething]; // non-UI side effects
 *       }
 *   }
*/
@property (nonatomic, assign) BOOL fd_isTemplateLayoutCell;

/* 启用以强制此模板布局单元格使用“框架布局”而不是“自动布局”，
 * 并且通过调用“-sizeThatFits：”来询问单元格的高度，所以你必须重写这个方法。
 * 仅当要手动控制此模板布局单元格的高度时才使用此属性
 * 计算模式，默认为NO。
*/
@property (nonatomic, assign) BOOL fd_enforceFrameLayout;

@end
```

------

- 先来看看我们平时开发中最频繁调用的两个方法
- (CGFloat)fd_heightForCellWithIdentifier:(NSString *)identifier cacheByIndexPath:(NSIndexPath *)indexPath configuration:(void (^)(id cell))configuration;
- (CGFloat)fd_heightForCellWithIdentifier:(NSString *)identifier cacheByKey:(id<NSCopying>)key configuration:(void (^)(id cell))configuration;

```objc
- (CGFloat)fd_heightForCellWithIdentifier:(NSString *)identifier cacheByIndexPath:(NSIndexPath *)indexPath configuration:(void (^)(id cell))configuration {
    if (!identifier || !indexPath) {
        return 0;
    }
    
    // Hit cache
    if ([self.fd_indexPathHeightCache existsHeightAtIndexPath:indexPath]) {
        return [self.fd_indexPathHeightCache heightForIndexPath:indexPath];
    }
    
    CGFloat height = [self fd_heightForCellWithIdentifier:identifier configuration:configuration];
    [self.fd_indexPathHeightCache cacheHeight:height byIndexPath:indexPath];

    return height;
}

- (CGFloat)fd_heightForCellWithIdentifier:(NSString *)identifier cacheByKey:(id<NSCopying>)key configuration:(void (^)(id cell))configuration {
    if (!identifier || !key) {
        return 0;
    }
    
    // Hit cache
    if ([self.fd_keyedHeightCache existsHeightForKey:key]) {
        CGFloat cachedHeight = [self.fd_keyedHeightCache heightForKey:key];
        return cachedHeight;
    }
    
    CGFloat height = [self fd_heightForCellWithIdentifier:identifier configuration:configuration];
    [self.fd_keyedHeightCache cacheHeight:height byKey:key];
    return height;
}


```

- 这两个方法，分别是对cell通过NSIndexPath 或者 key值 进行高度缓存，读取高度的时候，先从缓存cache中读取，如果缓存中没有，在通过[self fd_heightForCellWithIdentifier:identifier configuration:configuration]方法进行计算高度并加入缓存中。

```objc
- (CGFloat)fd_heightForCellWithIdentifier:(NSString *)identifier configuration:(void (^)(id cell))configuration {
    if (!identifier) {
        return 0;
    }
    
    UITableViewCell *templateLayoutCell = [self fd_templateCellForReuseIdentifier:identifier];
    
    //手动调用以确保与实际单元格的一致行为。 （显示在屏幕上）
    [templateLayoutCell prepareForReuse];
    
    if (configuration) {
        configuration(templateLayoutCell);
    }
    
    return [self fd_systemFittingHeightForConfiguratedCell:templateLayoutCell];
}
```

- 通过blocks进行配置并计算cell的高度，主要通过[self fd_templateCellForReuseIdentifier:identifier]方法创建一个UITableViewCell的实例templateLayoutCell，最后再把templateLayoutCell放入[self fd_systemFittingHeightForConfiguratedCell:templateLayoutCell]中进行计算返回高度。

```objc
- (__kindof UITableViewCell *)fd_templateCellForReuseIdentifier:(NSString *)identifier {
    NSAssert(identifier.length > 0, @"Expect a valid identifier - %@", identifier);
    
    NSMutableDictionary<NSString *, UITableViewCell *> *templateCellsByIdentifiers = objc_getAssociatedObject(self, _cmd);
    if (!templateCellsByIdentifiers) {
        templateCellsByIdentifiers = @{}.mutableCopy;
        objc_setAssociatedObject(self, _cmd, templateCellsByIdentifiers, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    
    UITableViewCell *templateCell = templateCellsByIdentifiers[identifier];
    
    if (!templateCell) {
        templateCell = [self dequeueReusableCellWithIdentifier:identifier];
        NSAssert(templateCell != nil, @"Cell must be registered to table view for identifier - %@", identifier);
        templateCell.fd_isTemplateLayoutCell = YES;
        templateCell.contentView.translatesAutoresizingMaskIntoConstraints = NO;
        templateCellsByIdentifiers[identifier] = templateCell;
        [self fd_debugLog:[NSString stringWithFormat:@"layout cell created - %@", identifier]];
    }
    
    return templateCell;
}

```

- 将所有创建的templateCell放置在一个字典templateCellsByIdentifiers中，并通过runtime将其加入内存中作为属性，实际上，templateCell 也是通过identifier在复用队列中获取复用的。所以，cell在使用前，应先注册为cell的复用对象。
- 最后调用的[self fd_systemFittingHeightForConfiguratedCell:templateLayoutCell]进行高度计算。当然也是最关键的一个操作，既然这是一个高度计算的框架，那么计算的步骤当然是重中之重。

```objc
- (CGFloat)fd_systemFittingHeightForConfiguratedCell:(UITableViewCell *)cell {
    CGFloat contentViewWidth = CGRectGetWidth(self.frame);
    
    if (cell.accessoryView) {
        //如果有定制accessoryView，则去除这个宽度
        contentViewWidth -= 16 + CGRectGetWidth(cell.accessoryView.frame);
    } else {
        //如果有系统accessoryView展示，则去除对应的宽度。
        static const CGFloat systemAccessoryWidths[] = {
            [UITableViewCellAccessoryNone] = 0,
            [UITableViewCellAccessoryDisclosureIndicator] = 34,
            [UITableViewCellAccessoryDetailDisclosureButton] = 68,
            [UITableViewCellAccessoryCheckmark] = 40,
            [UITableViewCellAccessoryDetailButton] = 48
        };
        contentViewWidth -= systemAccessoryWidths[cell.accessoryType];
    }
    
    CGFloat fittingHeight = 0;
    
    if (!cell.fd_enforceFrameLayout && contentViewWidth > 0) {
        //如果是自动布局，则将contentView的宽度约束添加进去。
        //这样做的目的是让UILabel之类的内容可能多行的控件适应这个宽度折行（当然前提是我们已经设置好了这些控件的布局约束）。然后调用systemLayoutSizeFittingSize来计算高度。
        //最后移除刚才临时添加的contentView宽度约束。
        NSLayoutConstraint *widthFenceConstraint = [NSLayoutConstraint constraintWithItem:cell.contentView attribute:NSLayoutAttributeWidth relatedBy:NSLayoutRelationEqual toItem:nil attribute:NSLayoutAttributeNotAnAttribute multiplier:1.0 constant:contentViewWidth];
        [cell.contentView addConstraint:widthFenceConstraint];
        
        fittingHeight = [cell.contentView systemLayoutSizeFittingSize:UILayoutFittingCompressedSize].height;
        [cell.contentView removeConstraint:widthFenceConstraint];
    }
    
    if (fittingHeight == 0) {
        // 尝试调用 '- sizeThatFits:'进行高度计算.
        // 注意：配件高度不应包括分隔线视图高度。
        fittingHeight = [cell sizeThatFits:CGSizeMake(contentViewWidth, 0)].height;
    }
    
    // 进行完前面的逻辑后高度如果仍然为0，则使用默认行高44
    if (fittingHeight == 0) {
        fittingHeight = 44;        
    }
    
    // 添加一像素作为tableView分割线高度。
    if (self.separatorStyle != UITableViewCellSeparatorStyleNone) {
        fittingHeight += 1.0 / [UIScreen mainScreen].scale;
    }
    
    return fittingHeight;
}

```

至此，就大致将这个框架分析的差不多了，源码中，对类的实例化均为采用runtime添加AssociatedObject的方式。就不做解释了。

------

### 最后

- 赏花不忘种花人，把作者关于这个框架优秀的性能分析复习下。

- 地址：[http://blog.sunnyxx.com/2015/05/17/cell-height-calculation/](http://blog.sunnyxx.com/2015/05/17/cell-height-calculation/)

  ​