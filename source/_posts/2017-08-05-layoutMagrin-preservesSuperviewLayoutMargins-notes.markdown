---
layout: post
title: "LayoutMargin & preservesSuperviewLayoutMargins 学习笔记"
date: 2017-08-05 13:30:51 +0800
comments: true
categories: iOS
---
<!-- more -->

### 发现问题
在IB中拖放tableView，大小为设置为屏幕大小后，添加了四个方向的约束，发现，约束的值居然不是和想象中不太一样，不是0。

![](https://github.com/engili/engili.github.io/raw/master/images/layout_note_1.png)

接着打开attribute inspector,发现约束信息如下：
![](https://raw.githubusercontent.com/engili/engili.github.io/master/images/layout_note_2.png)

在这里发现Second Item是superView 的 leading Margin. 而不是superView的leading，所以约束的值不是0，但leading margin又是什么？

### Layout Magrin

通过查找文档，发现这是iOS 8 以后，UIView 新增的一个属性，layoutMargins. 表示一个View的内边距。

### 文档

`@property(nonatomic) UIEdgeInsets layoutMargins;` The default spacing to use when laying out content in the view

```obj-c
typedef struct UIEdgeInsets {
    CGFloat top, left, bottom, right;  // specify amount to inset (positive) for each of the edges. values can be negative to 'outset'
} UIEdgeInsets;
```

这个属性类似Android中的padding，用于父View设置子View布局的内边距。

The default margins are eight points on each side.

默认值是8个点，可修改。

If the view is a view controller’s root view, the system sets and manages the margins. The top and bottom margins are set to zero points. The side margins vary depending on the current size class, but can be either 16 or 20 points. You cannot change these margins.

但如果，是ViewController的rootView，layoutMargins 是系统设置和控制的，不能修改。（文章最开始的tableView到 leading Margin 约束为 -16.就说明，left margin是16点。）

到这里，其实对layoutMargins理解都较为清晰，但是文档中还提到了一个与layoutMargins 相关的属性： preservesSuperviewLayoutMargins

When the edge of your view is close to the edge of the superview and the preservesSuperviewLayoutMargins property is YES, the actual layout margins may be increased to prevent content from overlapping the superview’s margins.

读完感觉字面意思都懂，但就是不知道是什么意思。

### preservesSuperviewLayoutMargins

`@property(nonatomic) BOOL preservesSuperviewLayoutMargins;` A Boolean value indicating whether the current view also respects the margins of its superview. The default value of this property is NO.

一个BOOL值，决定当前View布局是否也考虑父View的layoutMargins. 默认是NO.

到这里还是晕的，没看懂这个属性是干嘛，继续看文档。

### 文档
When the value of this property is YES, the superview’s margins are also considered when laying out content. This margin affects layouts where the distance between the edge of a view and its superview is smaller than the corresponding margin.

当这个属性是YES的时候，父view的layoutMargins会被考虑到当前View的布局中。也就是说，当前View和父View的内边距，小于父View相应layoutMargins中对应的内边距时，父View的内边距会决定当前View的布局。还是有些抽象。

For example, you might have a content view whose frame precisely matches the bounds of its superview. When any of the superview’s margins is inside the area represented by the content view and its own margins, UIKit adjusts the content view’s layout to respect the superview’s margins. The amount of the adjustment is the smallest amount needed to ensure that content is also inside the superview’s margins.

这里大概的意思就是，你有一个contentView，它的大小和父View一样大的你设置了contentView的preservesSuperviewLayoutMargins为YES，当你对contentView中的子View进行布局的时候，如果有子View的所在位置（比如：子View到contentView的左边距，小于父View的LayoutMargin 对应的左内边距）UIKit就会对子View进行一些调整。 感觉看了这一段稍微清晰了一些，可是写代码想实现一个demo的时候，发现这个preservesSuperviewLayoutMargins属性好像并没有什么卵用。 最后再Google上搜到了这篇文章。 [iOS8 Day-by-Day :: Day 32 :: Layout Margins](https://www.shinobicontrols.com/blog/ios8-day-by-day-day-32-layout-margins)

然后自己实现了一个 demo 如下：

```obj-c
- (void)viewDidLoad {
    [super viewDidLoad];

    //创建一个红色的View，大小和手机屏幕一样大，
    //layoutMargin 为（100，100，100，100）
    UIView *redView = [[UIView alloc] init];
    redView.backgroundColor = [UIColor redColor];
    redView.translatesAutoresizingMaskIntoConstraints = NO;
    redView.layoutMargins = UIEdgeInsetsMake(100, 100, 100, 100);
    [self.view addSubview:redView];
    [self.view addConstraints:[NSLayoutConstraint constraintsWithVisualFormat:@"H:|-(0)-[redView]-(0)-|" options:NSLayoutFormatDirectionLeadingToTrailing metrics:nil views:NSDictionaryOfVariableBindings(redView)]];
    [self.view addConstraints:[NSLayoutConstraint constraintsWithVisualFormat:@"V:|-(0)-[redView]-(0)-|" options:NSLayoutFormatDirectionLeadingToTrailing metrics:nil views:NSDictionaryOfVariableBindings(redView)]];

    //创建一个绿色的View，作为我们的ContentView，父View为红色的View
    //约束为到红色View各边距为10，但红色View 的layoutMargin 还是再绿色View内
    //设置绿色View preservesSuperviewLayoutMargins 为YES
    UIView *greenView = [[UIView alloc] init];
    greenView.backgroundColor = [UIColor greenColor];
    greenView.translatesAutoresizingMaskIntoConstraints = NO;
    greenView.preservesSuperviewLayoutMargins = YES;
    [redView addSubview:greenView];
    [redView addConstraints:[NSLayoutConstraint constraintsWithVisualFormat:@"H:|-(10)-[greenView]-(10)-|" options:NSLayoutFormatDirectionLeadingToTrailing metrics:nil views:NSDictionaryOfVariableBindings(greenView)]];
    [redView addConstraints:[NSLayoutConstraint constraintsWithVisualFormat:@"V:|-(10)-[greenView]-(10)-|" options:NSLayoutFormatDirectionLeadingToTrailing metrics:nil views:NSDictionaryOfVariableBindings(greenView)]];

    //创建一个蓝色View，作为绿色View的子View
    UIView *blueView = [[UIView alloc] init];
    blueView.backgroundColor = [UIColor blueColor];
    blueView.translatesAutoresizingMaskIntoConstraints = NO;
    [greenView addSubview:blueView];

    //注意，leading 约束对应的第二item 属性为 leading.margin
    [greenView addConstraint: [NSLayoutConstraint constraintWithItem:blueView attribute:NSLayoutAttributeLeading relatedBy:NSLayoutRelationEqual toItem:greenView attribute:NSLayoutAttributeLeadingMargin multiplier:1.0 constant:0.0]];
    // trailing 属性 为父View trailig - 8.0// 8.0 是默认layout Margin
    [greenView addConstraint: [NSLayoutConstraint constraintWithItem:blueView attribute:NSLayoutAttributeTrailing relatedBy:NSLayoutRelationEqual toItem:greenView attribute:NSLayoutAttributeTrailing multiplier:1.0 constant:-8.0]];
    [greenView addConstraint: [NSLayoutConstraint constraintWithItem:blueView attribute:NSLayoutAttributeTop relatedBy:NSLayoutRelationEqual toItem:greenView attribute:NSLayoutAttributeTop multiplier:1.0 constant:0.0]];
    [greenView addConstraint: [NSLayoutConstraint constraintWithItem:blueView attribute:NSLayoutAttributeBottom relatedBy:NSLayoutRelationEqual toItem:greenView attribute:NSLayoutAttributeBottom multiplier:1.0 constant:0.0]];
}
```

![](https://github.com/engili/engili.github.io/raw/master/images/layout_note_3.png)

### 总结

最后总结一下preservesSuperviewLayoutMargins生效的要点：（如果总结的不对希望大家指正~） 1.preservesSuperviewLayoutMargins 只在自动布局情况下生效 2.contentView（设置preservesSuperviewLayoutMargins的View）的父View的LayoutMargin中，至少存在contentView 某一个方向到父View的边距，小于父View LayoutMargin 对应的内边距。（就比如上面栗子中，绿色View到红色View的左边距是10，红色View的LayoutMargin中对应的左内边距为100） 3.设置自动布局约束的时候，一定要设置Margin相关的约束(iOS8以上有效)

```obj-c
typedef NS_ENUM(NSInteger, NSLayoutAttribute) {
   ...

    NSLayoutAttributeLeftMargin NS_ENUM_AVAILABLE_IOS(8_0),
    NSLayoutAttributeRightMargin NS_ENUM_AVAILABLE_IOS(8_0),
    NSLayoutAttributeTopMargin NS_ENUM_AVAILABLE_IOS(8_0),
    NSLayoutAttributeBottomMargin NS_ENUM_AVAILABLE_IOS(8_0),
    NSLayoutAttributeLeadingMargin NS_ENUM_AVAILABLE_IOS(8_0),
    NSLayoutAttributeTrailingMargin NS_ENUM_AVAILABLE_IOS(8_0),
    NSLayoutAttributeCenterXWithinMargins NS_ENUM_AVAILABLE_IOS(8_0),
    NSLayoutAttributeCenterYWithinMargins NS_ENUM_AVAILABLE_IOS(8_0),

};
```

### 思考

现在基本上弄清楚了，LayoutMargin和preservesSuperviewLayoutMargins属性，以及如何使用，但是对他们在自动布局的实际使用场景还是没有思考太清楚，没有明白苹果为什么会设置这两个属性，特别是preservesSuperviewLayoutMargins属性。 = =