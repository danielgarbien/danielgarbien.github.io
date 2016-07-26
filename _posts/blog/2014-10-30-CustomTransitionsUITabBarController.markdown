---
layout: post
title:  "Custom transitions in UITabBarController"
date:   2014-10-30 12:23:07
categories: blog
tags: [Objective-C, Animations]
---

## The idea

Building your own container from scratch to support custom animations takes a lot of nerve. It would also be a pity to disregard transitioning API available from iOS 7. Still, in order to use the (not so) new API you have a choice to go with your own custom containment that supports it ([painfull][objc_issue]) or go easy on yourself and use one of framework's containments.
Scenario I've faced required retaining all of child view controllers. My choice was UITabBarController.

## Implementation

Classes involved:

* UITabBarController : UIViewController

* TabBarControllerDelegate : NSObject \<UITabBarControllerDelegate\>

Initialized with instance of Animator. Depending on transition's nature returns proper animators: the one it was initialized with and (if interactive) an instance of UIPercentDrivenInteractiveTransition.

* TabBarTransitionController : NSObject

Initialized with UITabBarController and TabBarControllerDelegate instances. The former must be the tab bar controller's delegate.
Transition controller adds a gesture recognizer to tabBarController's view and handles it to change tabs and drive interactive transition. Reference to the instance of TabBarControllerDelegate is needed to decide on transition's nature and drive it if interactive.

* Animator : NSObject \<UIViewControllerAnimatedTransitioning\>

Contains animation code. It also introduces new protocol and sends a message to its delegate when a transition ends. 
Unfortunatelly there is no easy way to be notified by UITabBarController that a tab was just changed (selectedIndex property is not KVO compliant).

## Secure the container's consistency

A tab change might be triggered from multiple actions (pan gesture and tap on a tab bar) therefore we need to emphesize on keeping UITabBarController's state consistent. E.g. you shouldn't be able to select yet another index when there is a transition in progress. We can't simply set userInteractionEnabled to NO because 1) blocking all interactions might be excessive and more importantly 2) it won't block interactions started before changing this property. For this reason I've introduced a category on UITabBarController:

``` objc
@interface UITabBarController (SafeIndexSelection)

- (void)beginSelectingIndexSafely:(NSUInteger)selectedIndex;
- (void)beginSelectingViewControllerSafely:(UIViewController *)selectedViewController;
- (void)endSelectingSafely;

@end
```

Begin and end methods should be used pairwise as changing tabs in between is blocked.

## Tips and tricks

Transition's start is not quite synchronous with setting selectedIndex. E.g. with gesture driven transition it is possible that very short gesture ends before the transition has even started. Consider the following scenario:

1. In GestureStarted : set new selected index on tab bar controller (starts the transioning mechanism, asking delegate for animator etc.).
2. In GestureChanged : update interactive transition ratio (in fact transition hasn't been started yet). 
3. In GestureEnded : finish interactive transition.
4. ...finally transition has been started by the framework.

It's rare but possible for very short gestures. At this point the transition will never be finished. This leaves view hierarchy in inconsistent state. As a workaround you may pass additional info to the interactive animator, example in a demo project.

## Demo

Demo project on [github][demo_project].

[objc_issue]: http://www.objc.io/issue-12/custom-container-view-controller-transitions.html
[demo_project]: https://github.com/danielgarbien/TabBarTransitioning
