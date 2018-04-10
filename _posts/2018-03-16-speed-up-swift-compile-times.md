---
layout: post
title:  Speed Up Swift Compile Times
date:   2018-03-16 10:09:59
description: A few ways to debug and help speed up Swift compile times.
---


### Debugging with BuildTimeAnalyzer.app

https://github.com/RobertGummesson/BuildTimeAnalyzer-for-Xcode

### Code Optimized for Slow Compile Times

It takes 1600+ ms to compile the following code:
```
/// Registers our user with our various analytics libraries.
func registerUser(_ user: User) {
	
	AwesomeAnalytics.shared.setIdentity("\(user.id)")
    
    CoolAnalytics.shared.setUserId("\(user.id)")
    CoolAnalytics.shared.setUserProperties([
        "email": user.email ?? "",
        "name": user.name,
        "created": user.createdAt?.iso8601() ?? "",
        "gender": user.gender?.rawValue ?? "",
        "training_frequency": user.trainingFrequency ?? ""
    ])

    WOWAnalytics.shared.identify("\(user.id)", traits: [
        "email": user.email ?? "",
        "name": user.name,
        "created": user.createdAt?.iso8601() ?? "",
        "gender": user.gender?.rawValue ?? "",
        "training_frequency": user.trainingFrequency ?? ""
        ])

    SearchAnalytics.shared.identifyUser(user)
    
    CrashAnalytics.shared.setUserIdentifier("\(user.id)")
}
```

Being more explicit with the structure & reducing duplication reduces our compile time to ~25ms:
```
/// Registers our user with our various analytics libraries.
class func registerUser(_ user: User) {

	let userId = String(user.id)
    let email = user.email ?? ""
    let name = user.name ?? ""
    let createdAt = user.createdAt?.iso8601() ?? ""
    let gender = user.gender?.rawValue ?? ""
    let frequency = user.trainingFrequency ?? ""

    let traits = [
        "email": email,
        "name": name,
        "created": createdAt,
        "gender": gender,
        "training_frequency": frequency
    ]

    AwesomeAnalytics.shared.setIdentity(userId)
    CoolAnalytics.shared.setUserId(userId)
    CoolAnalytics.shared.setUserProperties(traits)
    WOWAnalytics.shared.identify(userId, traits: traits)
    SearchAnalytics.shared.identifyUser(user)    
    CrashAnalytics.shared.setUserIdentifier(userId)
}
```


### Hidden Swift Whole Module Optimization Setting

Create a user-defined setting titled `SWIFT_WHOLE_MODULE_OPTIMIZATION` and set it to `YES`. This is a hidden Swift compiler settings.

My Dev Machine:
- MacBook Pro (15-inch, 2017)
- Processor: 3.1 GHz Intel Core i7
- Memory: 16 GB 2133 MHz LPDDR3
- Storage: Flash Storage

| branch | Build Time (mm:ss) |
|-|-:|
| `master` | 02:44 |
| `with_swmo_on` | 01:34 |


Colleague with iMac Pro:
store_beta: 1:04
Blake_fast: 0:46

Colleague with older MacBook Pro:
before: 7:13
after: 3:20