---
title: "Writing custom extensions"
date: 2019-08-02 12:00:00 +0300
categories: rxswift
toc: true
toc_sticky: true
---

## What an extension is?

Technically, `rx` extensions for `SomeClass` is an extension, written in the following way (replacing `someObservable()` to something meaningful in your context):
 
```swift
extension Reactive where Base: SomeClass {
  func someObservable() -> Observable<Int> {
    return Observable<Int>...
  }
}
```

This extension alone doesn't give you any benefits (except `SomeClass` inherits from `NSObject`). Before you could access to `.rx.someObservable()` on the instance, you have to write one more extension:

```swift
extension SomeClass: ReactiveCompatible {}
```

There are no limitations for return type in the `Reactive` extension. But at least for me it wouldn't make much sense, if I had written here non-reactive methods or properties. Each time I write `.rx.` at the end of some instance, I usually expect to see an autocompletion with something reactive. I think that a good practice is to place into extension an Observable or an Observer, or both.

### UITextField

RxCocoa library contains a lot of extensions, like `UITextField+Rx.swift`. These extensions usually provide Observable or Observer or both behavior for useful properties, like `text`, which might be used in the following way:

```swift
//as Observable
loginField.rx.text.orEmpty
  .bind(to: viewModel.loginRelay)
  .disposed(by: disposeBag)
	
//as Observer
viewModel.loginRelay
  .bind(to: loginField.rx.text)
  .disposed(by: disposeBag)
```

All you need to do is just write `.rx.` and see methods and properties available which fit your needs.

Using extension helps us writing code in convenient and recognizable rx-way. In the introduction post [what RxSwift is]({% post_url 2019-04-26-rxswift-introduction %}) I wrote that it is one of the pros of RxSwift.

## Extensions are optional

Of course, you don't have to use these extensions, if you don't like. But I highly recommend you at least to look through properties and methods covered in RxCocoa for UIKit. 

Assume you are beginner developer and haven't heard about RxCocoa or you don't want to import this library along with RxSwift.

In this case the example for `loginField` will be re-written:

```swift
override func viewDidLoad() {
  super.viewDidLoad()
  loginField.addTarget(self,
	                   action: #selector(textChanged(sender:)),
	                   for: [.allEditingEvents, .valueChanged])
	
  //as Observer
  viewModel.loginRelay
    .subscribe(onNext: { [weak self] text in
      self?.loginField.text = text
    })
    .disposed(by: disposeBag)
}

//as Observable
@objc private func textChanged(sender: Any) {
  if let textField = sender as? UITextField,
    textField == loginField,
    let text = loginField.text {
      viewModel.loginRelay.accept(text)
  }
}
```

It's quite a bit more code. Observable part is replaced with target-action pattern and code related to the logic is separated into `viewDidLoad` and `textChanged(sender:)` methods. Observer part is replaced with regular setter inside subscription closure.

## RxCocoa extensions

If you check RxCocoa's sources out, you'll find, that all extensions return 3 types:

* Binder;
* [ControlEvent]({% post_url 2019-05-17-traits %});
* [ControlProperty]({% post_url 2019-05-17-traits %});

Investigating what the types above are will help you to decide which one you should use when you need your own extension for UI class. 

### Binder (Observer only)

Probably, the easiest way to have an Observer for some property or method. Look, how easy it was to declare `isAnimating` for `UIActivityIndicatorView`:

```swift
extension Reactive where Base: UIActivityIndicatorView {
  public var isAnimating: Binder<Bool> {
    return Binder(self.base) { activityIndicator, active in
      if active {
        activityIndicator.startAnimating()
      } else {
        activityIndicator.stopAnimating()
      }
    }
  }
}
```

Or `isEnabled` for `UIAlertAction`:

```swift
extension Reactive where Base: UIAlertAction {
  public var isEnabled: Binder<Bool> {
    return Binder(self.base) { alertAction, value in
      alertAction.isEnabled = value
    }
  }
}
```

It's pretty easy, isn't it?

And there is its description:

* Can't bind errors (in debug builds binding of errors causes `fatalError` in release builds errors are being logged)
* Ensures binding is performed on a specific scheduler (by default on main scheduler)
* `Binder` doesn't retain target and in case target is released, element isn't bound.

### ControlEvent (Observable only)

When you **use** `ControlEvent` from extensions written in RxCocoa you may enjoy all its benefits, which are:

* It never fails;
* It doesn’t send any initial value on subscription;
* It `Complete`s the sequence when the control deallocates;
* It never errors out;
* It delivers events on `MainScheduler.instance`.

But when you **write** your own `ControlEvent`, you **must guarantee**, that all benefits(requirements) enumerated above are satisfied. Because if they aren't, you could potentially break someone’s code.

Hopefully, for UIControl simple target-action wrapper is already written - `func controlEvent(_:)`. All you need is just passing an array of interested `UIControlEvents`:

```swift
extension Reactive where Base: UIButton {
  public var tap: ControlEvent<Void> {
    return controlEvent(.touchUpInside)
  }
}
```

In some cases, ControlEvent might be initialized with `source` Observable, that meets all requirements above. As an example, `itemSelected` for `UICollectionView` is written in this way. The source observable is built using [DelegateProxy]({% post_url 2019-07-26-delegates %}):

```swift
public var itemSelected: ControlEvent<IndexPath> {
  let source = delegate.methodInvoked(#selector(UICollectionViewDelegate.collectionView(_:didSelectItemAt:)))
    .map { a in
      return try castOrThrow(IndexPath.self, a[1])
    }  
  return ControlEvent(events: source)
}
```

### ControlProperty (Observer and Observable)

ControlProperty comes with power of both, Observer and Observable types. I'll remind you its benefits:

* It never fails;
* `shareReplay(1)` behavior;
* It's stateful, upon subscription (calling subscribe) last element is immediately replayed if it was produced;
* It will `Complete` sequence on control being deallocated
* It never errors out;
* It delivers events on `MainScheduler.instance`

Sequence of values only represents initial control value and user-initiated value changes. Programmatic value changes won't be reported.

The same as for ControlEvent, when you are wrapping UIControl, you might use target-action wrapper available `func controlProperty(editingEvents:getter:setter:)`.

`value` extension for `UISlider` is the following:

```swift
extension Reactive where Base: UISlider {
  public var value: ControlProperty<Float> {
    return base.rx.controlPropertyWithDefaultEvents(
      getter: { slider in
        slider.value
      }, setter: { slider, value in
        slider.value = value
      }
    )
  }
}
```

In some cases `ControlProperty` might be initialized using `init(values:, valueSink:)`.

## Custom UIControl extension

Subclasses of [UIControl](https://developer.apple.com/documentation/uikit/uicontrol) may vary. But if your subclass is written in the convenient way, somewhere in the implementation you'll have `sendActions(for: .valueChanged)`, or `sendActions(for: . touchUpInside)`. Good samples for custom UIControls are written on raywenderlich - [Knob Control](https://www.raywenderlich.com/5294-how-to-make-a-custom-control-tutorial-a-reusable-knob) and [Custom Slider](https://www.raywenderlich.com/2297-how-to-make-a-custom-control-tutorial-a-reusable-slider). Also, there are a lot of custom controls on GitHub.

### Use ControlProperty

If your custom UIControl relies on some value, and uses event `.valueChanged`, it will be easy to wrap this value with `ControlProperty`. 

And don't worry if your (or third party) implementation doesn't use events at all, you still able to instantiate ControlProperty with source Observable, but read its requirement twice! You may start with `BehaviorRelay` as the source.

### Use ControlEvent

If your custom UIControl doesn't store any value and `.valueChanged` event is used, it will be easy to wrap this event with `ControlEvent`. 

Remember, you are able to instantiate ControlEvent with source Observable, if your control doesn't send any event. You may start with `PublishRelay` as the source.

## For UI but not UIControl cases

In this case you might ask yourself what behavior you are expected to have? Observable, Observer or both. An answer to this question will give you the right type - `ControlEvent`, `Binder` or `ControlProperty`.

And don't abuse `BehaviorRelay` or `PublishRelay`. If your implementation relies on delegates you should construct Observable source using `DelegateProxy` and pass it to `ControlProperty` or `ControlEvent` initializer.

## Community extension

Reactive extensions exist not only for UIKit. And we are not limited only by `ControlEvent`, `Binder`, `ControlProperty`. For your extension you could use raw Observable, or any [trait]({% post_url 2019-05-17-traits %}) that perfectly describes behavior of your extension.

Let's take a look what types are used in some popular extensions written by community.

### RxAlamofire

Here we could find 3 classes wrapped with reactive:

* URLSession;
* SessionManager;
* DataRequest;
* Request.

The methods from these extensions return raw Observable. There are a few entries of `Observable.create` to wrap asynchronous calls into reactive world.

### RxGesture

Because gestures are about UI, it is written using ControlEvent and ControlProperty

### RxKeyboard

In RxKeyboard `Driver` trait is used. It's suitable for UI too.

One `frame` `Driver<CGRect>` is backed by BehaviorRelay, other are just transforms of the `frame`.

### RxRealm

There is a basic `ObserverType` is defined - `RealmObserver<Element>` and it helps writing reactive code.