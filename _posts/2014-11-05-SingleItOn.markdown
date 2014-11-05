---
layout: post
title:  "Turning class into singleton with a category"
date:   2014-11-05 12:23:07
categories: ios
---

## The idea

How to turn a class into singleton when you don't have access to implementation? It's possible with a category.
Since it's very specific to the third party framework and an unfortunate architectural detail I'll skip why and explain how.

## Implementation

Firstly, the class has to provide sharedInstance alike method in it's public API, which creates an instance the first time it's used and returns happily once created instance ever after. This one's easy and well known:

``` objc
#pragma mark - Public Class Methods

+ (instancetype)sharedInstance
{
    static dispatch_once_t once;
    static id _sharedInstance = nil;
    dispatch_once(&once, ^{
        _sharedInstance = [[self alloc] init];
    });
    return _sharedInstance;
}
```

Secondly, it needs to be ensured that only one single instance is ever created. In order to do that we have to override (if subclassing) or swizzle +allocWithZone: method and return a shared instance from there. In fact we should do that with every kind of singletons, but it's especially crucial if we have no control over it's creation (that was my case).
Remember about changing sharedInstance implentation to use correct version of alloc. Here's the entire implementation for the category:

``` objc
#pragma mark - Public Class Methods

+ (instancetype)sharedInstance
{
    static dispatch_once_t once;
    static id _sharedInstance = nil;
    dispatch_once(&once, ^{
        _sharedInstance = [[self singleton_allocWithZone:nil] init];
    });
    return _sharedInstance;
}

#pragma mark - Overridden Class Methods

+ (void)load
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        
        Class class = [self class];
        
        SEL originalSelector = @selector(allocWithZone:);
        SEL swizzledSelector = @selector(singleton_allocWithZone:);
        
        class = object_getClass((id)class);
        
        Method originalMethod = class_getClassMethod(class, originalSelector);
        Method swizzledMethod = class_getClassMethod(class, swizzledSelector);
        BOOL didAddMethod = class_addMethod(class, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod));
        if (didAddMethod) {
            class_replaceMethod(class, swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

#pragma mark - Swizzle Methods

+ (id)singleton_allocWithZone:(NSZone *)zone
{
    return [self sharedInstance];
}
```

Not so much of a code, right?

## What's next

Turn it into a gist or a macro (something like [Matt Gallagher did][cocoa_with_love]) and use wisely.
Good luck!

[cocoa_with_love]: http://www.cocoawithlove.com/2008/11/singletons-appdelegates-and-top-level.html
