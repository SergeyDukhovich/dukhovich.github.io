---
title: "Writing custom extensions"
date: 2019-08-02 12:00:00 +0300
categories: rxswift
toc: true
toc_sticky: true
---

One of RxSwift pros is that its code usually written in the same way. And you "write once, read many times":

* there is an Observable sequence;
* there might be or might not transformation operators;
* then we subscribe to its next events;
* then we do some memory managment.

```swift
observableString
  .subscribe(onNext: { [weak self] str in
    self?.label.text = str
  })
  .disposed(by: disposeBag)
```

Or more stylish, convient RxSwift/RxCocoa way:

```swift
observableString
  .bind(to: label.rx.text)
  .disposed(by: disposeBag)
``` 

Even bind is just a short version of subscribe method, it helps reducing number of closures in code and reducing number of lines in code. And my opinion is, that bind version is quite more readable.

RxCocoa library contains a lot of extensions, like `UITextField+Rx.swift`. These extensions usually provide both Observable and Observer behavior for inner properties, like `text` for `UITextField`.




 covering most used properties in UIKit classes
