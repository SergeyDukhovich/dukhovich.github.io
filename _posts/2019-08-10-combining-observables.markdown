---
title: "Combining Observables"
date: 2019-08-10 12:00:00 +0300
categories: rxswift
toc: true
toc_sticky: true
---

Quite often we have two or more Observable sources and want to get single Observable applying specific behavior that fits our use case. In this post I'll try to cover some operators which are usually used for this purpose.

## Merge

You can combine the output of multiple Observables so that they act like a single Observable, by using the `merge` operator. Observable sources must have the same type, and this type will be used in result Observable.

* Reaction on `completed`: If 2 of 3 Observable sources terminate by emitting `completed` event, result Observable continues working.
* Reaction on `error`: If any of Observable sources terminates by emitting `error` event, result observable terminates too. 

When you receive events on the result Observable, you don't know from which source it came.

![merge marble diagram](http://dukhovich.net/assets/images/articles/16/merge.png)

And its usage:

```swift
let arrayOfObservable: [Observable<String>]
  = [first, second, third]

Observable.merge(arrayOfObservable)
  .subscribe(onNext: { str in
    print(str)
  })
  .disposed(by: disposeBag)
```

Potential problem might appear when you merge  a lot of Observable sources which represent URL requests, they will be executed simultaneously.

## CombineLatest

When an item is emitted by either of two Observables, combine the latest item emitted by each Observable via a specified function and emit items based on the results of this function. Source Observables might have different types.

* Reaction on `completed`: If 2 of 3 Observable sources terminate by emitting `completed` event, result Observable continues working. But if these Observables haven't emitted any `next` event before, result Observable continues as never operator.
* Reaction on `error`: If any of Observable sources terminates by emitting `error` event, result observable terminates too. 

When you receive events on the result Observable, you could recognize from which source it came by array or tuple index.

![combine latest marble diagram](http://dukhovich.net/assets/images/articles/16/combinelatest.png)

```swift
Observable
  .combineLatest(arrayOfObservable) {
    return $0.reduce("", { $0 + " \($1)" })
  }
  .subscribe(onNext: { str in
    print(str)
  })
  .disposed(by: disposeBag)

Observable
  .combineLatest(first, second, third)
  .subscribe(onNext: { (s1, s2, s3) in
    print("\(s1) \(s2) \(s3)")
  })
  .disposed(by: disposeBag)

Observable.combineLatest(arrayOfObservable)
  .subscribe(onNext: { array in
    print(array.reduce("", { $0 + " \($1)" }))
  })
  .disposed(by: disposeBag)
```

There are two options using this operator: you could use up to 8 Observable sources, separated by comma or you could pass an array of Observable sources. 

Also, you could use `resultSelector` and transform tuple or array immediately, or pass them as is.

Common problem is when one of Observable sources doesn't emit any event, result Observable misbehaves.

## WithLatestFrom

This operator works only with 2 Observable sources. When the first source emits an event result Observable reemits it with latest event of second Observable if it had events before.

* Reaction on `error`: If any of Observable sources terminates by emitting `error` event, result observable terminates too. 

* Reaction on `completed`: If 1st Observable source terminate by emitting `completed` event, result Observable completes too.

![with latest from marble diagram](http://dukhovich.net/assets/images/articles/16/withlatestfrom1.png)

* Reaction on `completed`: If 2nd Observable source terminate by emitting `completed` event, result Observable continues working.

![with latest from marble diagram](http://dukhovich.net/assets/images/articles/16/withlatestfrom2.png)

Common problem here is similar to `combineLatest` operator. If 2nd Observable hadn't emitted any event before 1nd Observable has, this next event wouldn't be reemitted by result Observable. On both marble diagrams there is no `1*` event.

## StartWith

And the problems, mentioned in the description of the last 2 operators, are solved by the following operator, which emits a specified sequence of items before beginning to emit the items from the source Observable.

![start with marble diagram](http://dukhovich.net/assets/images/articles/16/startwith.png)

## Zip

Combine the emissions of multiple Observables together via a specified function and emit single items for each combination based on the results of this function.

![zip marble diagram](http://dukhovich.net/assets/images/articles/16/zip.png)

## Concat

Concatenates all observable sequences in the given collection, as long as the previous observable sequence terminated successfully. Normal termination is required, cause if some sequence doesn't emit `completed` the following won't start. 

![concat marble diagram](http://dukhovich.net/assets/images/articles/16/concat.png)

I've found this operator useful, when I had to write quite a lot of URL requests, wrapped in Observable.

## FlatMap

The `flatMap` operator usually is used in a bit different context, but it suits for chaining of Observable sequences too. Let's say we want to chain API calls:

* At first, get list of restaurants;
* Then for the first restaurant get list of the reviews;
* Then for the first review get user details.

```swift
api
  .restaurantList()
  .flatMap { [weak self] restaurants -> Observable<[Review]> in
    guard let self = self,
      let id = restaurants.first?.id
      else { return .empty() }
    return self.api.restaurantReviews(by: id)
  }
  .flatMap { [weak self] reviews -> Observable<User> in
    guard let self = self,
      let id = reviews.first?.user.id
      else { return .empty() }
    return self.api.user(by: id)
  }
  .subscribe(onNext: { user in
    print(user)
  })
  .disposed(by: disposeBag)
```

## Amb

Given two or more source Observables, emit all of the items from only the first of these Observables to emit an item or notification.

To be honest, I've never used this operator on practice.

![amb marble diagram](http://dukhovich.net/assets/images/articles/16/amb.png)