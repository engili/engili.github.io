---
layout: post
title: "UIKit Animations 学习笔记"
date: 2018-03-24 21:33:48 +0800
comments: true
categories: iOS
---
<!-- more -->

### UIKit Animations 学习笔记

#### 1. UIView 可动画属性 (Animatable Property)

- frame
    - 当transform 属性不为identity时，frame是未定义的，使用bounds或center做动画
- bounds
- center
- transform
- alpha
- backgroundColor
- ~~contentStretch (iOS 7.0弃用）~~
    - ~~Modify this property to change the way the view’s contents are stretched to fill the available space.~~

#### 2. Animations Block-Based Methods

`class func animate(withDuration duration: TimeInterval, animations: @escaping () -> Void)`
- 这个方法使用curveEaseInOut和transitionNone选项，动画期间动画的View不响应用户交互

`class func animate(withDuration duration: TimeInterval, animations: @escaping () -> Void, completion: ((Bool) -> Void)? = nil)`
- completion,如果duration值是0，block会在下一个runloop cycle被调用

`class func animate(withDuration duration: TimeInterval, delay: TimeInterval, options: UIViewAnimationOptions = [], animations: @escaping () -> Void, completion: ((Bool) -> Void)? = nil)`
- options 可以设置一个或者多个，以数组的形式

##### 2.1 UIViewAnimationOptions
###### 2.1.1 常规属性（可以设置多个）
- layoutSubviews
    - layout子View，让他们随父view一起动画
必须重写view的LayoutSubViews方法

```obj-c
//  MTAnimateView.swift
class MTAnimateView: UIView {
    var subview: UIView!
    override func layoutSubviews() {
        let bounds = self.bounds
        subview.frame = CGRect(x: 0, y: 0, width: bounds.width / 2, height: bounds.height / 2)
    }
}

//UIViewController.swift
class ViewController: UIViewController {
    var redView: MTAnimateView!
    //动画block 扩大bounds，长宽各100，不带`layoutSubviews` 动画效果就很奇怪
    func animateWithLayoutSubViews() {
        let orginBounds = redView.bounds
        UIView.animate(withDuration: 1.0, delay: 0.0, options: .layoutSubviews, animations: {
            self.redView.bounds = CGRect(x: orginBounds.origin.x, y: orginBounds.origin.y, width: orginBounds.width + 100, height: orginBounds.height + 100)
        }) { (_) in
            self.redView.bounds = orginBounds
        }
    }

}
```

- allowUserInteraction
    - 响应动画时的用户交互
- beginFromCurrentState
    - If this key is not present, all in-flight animations are allowed to finish before the new animation is started. If another animation is not in flight, this key has no effect.
- \`repeat`
    - 重复动画

```swift
UIView.animate(withDuration: 0.5, delay: 0.0, options: .repeat, animations: {
            //重复执行3次
            UIView.setAnimationRepeatCount(3)
            ...
        }, completion: nil)
```

- autoreverse
    - 与repeat配合使用，动画能回溯
- overrideInheritedDuration
    - 忽略嵌套动画的时间,使用自己设置的duration
- overrideInheritedCurve
    - 忽略嵌套动画的时间曲线，使用自己设置的时间曲线

    嵌套动画
    
    ```swift
    UIView.animate(withDuration: 2.0, delay: 0.0, options: .curveEaseIn, animations: {
        self.redView.center = redViewNewCenter
        UIView.animate(withDuration: 0.5, delay: 0.0, options: [.curveEaseOut, .overrideInheritedDuration, .overrideInheritedCurve], animations: {
            self.blueView.center = blueViewNewCenter
        }, completion: { (_) in
            self.blueView.center = blueViewOriginalCenter
        })
    }) { (_) in
        self.redView.center = redViewOriginalCenter
}
    ```
    
- allowAnimatedContent
    - 动画的时候，重绘View，没设置这个选项的时候，动画都是使用的截图
- showHideTransitionViews
    - 用于转场动画
When present, this key causes views to be hidden or shown (instead of removed or added) when performing a view transition. Both views must already be present in the parent view’s hierarchy when using this key. If this key is not present, the to-view in a transition is added to, and the from-view is removed from, the parent view’s list of subviews.

- overrideInheritedOptions

###### 2.1.2 速度属性（可以设置一个）

- curveEaseInOut
    - 开始慢，中间加速，结尾慢
- curveEaseIn
    - 开始慢，一直加速
- curveEaseOut
    - 开始快，一直减速
- curveLinear
    - 均匀的

###### 2.1.3 转场动画属性 （可以设置一个）
- transitionFlipFromLeft
- transitionFlipFromRight
- transitionCurlUp
- transitionCurlDown
- transitionCrossDissolve
- transitionFlipFromTop
- transitionFlipFromBottom
- preferredFramesPerSecond30
- preferredFramesPerSecond60


#### 3. 嵌套动画
嵌套动画，是嵌套在其他动画block中的动画，他开始动画的时间和他父动画一样，并且继承父动画的时间，时间曲线和其他动画配置。 可以通过overrideInheritedDuration、overrideInheritedCurve 来使用自己的配置做动画

#### 4.转场动画 (动画效果比较浮夸，用的较少)

`class func transition(with view: UIView, duration: TimeInterval, options: UIViewAnimationOptions = [], animations: (() -> Void)?, completion: ((Bool) -> Void)? = nil)`
- 在view中进行转场动画
- 在animations中可以add、remove、hide、show view，如果想要其他动画效果，需要配置allowAnimatedContent
示例
```swift
UIView.transition(with: redView, duration: 1.0, options: .transitionFlipFromLeft, animations: {
            self.greenView.isHidden = false
            self.blueView.isHidden = true
        }, completion: nil)
```

`class func transition(from fromView: UIView, to toView: UIView, duration: TimeInterval, options: UIViewAnimationOptions = [], completion: ((Bool) -> Void)? = nil)`

- 这个方法，从fromView转场到toView
- 系统实现是直接从视图层级中移除，可以使用 showHideTransitionViews 只是显示和隐藏两个view

#### 5.UIViewPropertyAnimator
##### 5.1.1 构造方法
`convenience init(duration: TimeInterval, curve: UIViewAnimationCurve, animations: (() -> Void)? = nil)`

- 构造函数 创建出的animator是inactive状态
- 需要调用startAnimation()开始动画
    
    
##### 5.1.2 添加动画

`func addAnimations(_ animation: @escaping () -> Void)
`

- 可以通过这个方法，添加多个动画block
- inactive 状态下，添加的动画，进行时间是配置的时间，active状态下，添加的动画，进行的时间，是animtor剩余的时间
- stopped状态下，添加动画，会报错   

 
##### 5.1.3 添加动画结束回调

`func addCompletion(_ completion: @escaping (UIViewAnimatingPosition) -> Void)`

- 动画正常结束时，执行回调
- 调用`stopAnimation(_:)` 参数设置true,执行回调
- 调用`stopAnimation(_:)` 参数设置false,在`finishAnimation(at:)`调用后，执行回调
    
##### 5.1.4 continueAnimation

调整一个暂停动画的 时间函数 和 时间

`
func continueAnimation(withTimingParameters parameters: UITimingCurveProvider?, durationFactor: CGFloat)
`
- 调用该方法，需要animtor处于active且暂停的状态
- 当状态处于非active,或动画正在进行，或者 isInterruptible属性是false时 会报错
- parameters: 新的时间函数，但是为了避免，动画有跳动的现象，一般改变时间函数后，会有一定的过渡效果
- durationFactor: 时间乘法因子，和原有的时间，相乘，获取最终的时间
- 当这次动画结束后，才会重写animator的timing和duration属性


#####5.2 UIViewAnimating

###### 5.2.1 UIViewAnimating

- inactive
    - 构造方法创建的animator初始状态
- active
    - 当调用startAnimation() 或 pauseAnimation() 后，处于这个状态
- stopped
    - 动画自然结束，或者调用stopAnimation(_:)
    - 调用stopAnimation(_:),会让动画停留在当前值，而不是最终动画设定的值

    ![](https://github.com/engili/engili.github.io/raw/master/images/animation_states_2x.png)

###### 5.2.2 UIViewAnimatingPosition

- end
- start
- current

###### 5.2.3 动画状态改变

- startAnimation()
    - 开始一个动画，或者恢复一个暂停的动画
    - stopped状态的动画，不能调用该方法
- pauseAnimation()
    - 暂停动画
    - inactive动画调用后，转为active状态，并且动画处于暂停状态
- stopAnimation(_:) 使动画，在当前状态停止

`func stopAnimation(_ withoutFinishing: Bool)`

- withoutFinishing:
    - 设置为true,animator进入inactive状态
    - 设置为false,animator进入stopped状态，之后可以进行一些其他操作，
- finishAnimation(at:)使得动画结束，animator回到inactive


###### 5.2.4 可交互动画


- fractionComplete
    - 动画完成百分比
    - 通过修改这个值，可以使animator进行到相应的过程

```swift
@objc fileprivate func handleInterruptableGesture(_ gesture: UIPanGestureRecognizer) {
    let offset = UIScreen.main.bounds.width - 100
    switch gesture.state {
    case .began:
        if interruptableAnimator == nil {
            interruptableAnimator = UIViewPropertyAnimator(duration: 3.0, curve: .easeOut, animations: {
                self.interruptableAnimtionView.frame = self.interruptableAnimtionView.frame.offsetBy(dx: offset, dy: 0)
            })
        }
        interruptableAnimator!.pauseAnimation()
        progressWhenInterrupted = interruptableAnimator!.fractionComplete
    case .changed:
        let translation = gesture.translation(in: self.interruptableAnimtionView)
        interruptableAnimator!.fractionComplete = (translation.x / offset) + progressWhenInterrupted

    case .ended, .cancelled:
        let timing = UICubicTimingParameters(animationCurve: .easeOut)
        interruptableAnimator!.continueAnimation(withTimingParameters: timing, durationFactor: 0)
    default:
        break
    }
}
```