---
layout: post
title:  "unowned(unsafe) var delegate"
date:   2016-07-11 19:39:07
categories: ios
---

During last [CocoaHeads][cocoaheads] meeting in Cracow I did a small live debugging session with a crowd. Together we debugged an interesting crash that's a side effect on some Foundation implementation details and has roots in a frightening unowned(unsafe) reference type.

## Setup

Test application is a classic master-detail. On the left there is a table view that lists few people. When a person is selected from the master list Detail screen is presented on the right. Detail contains some basic info + another table view. All table views are populated by fetched results controllers. From Detail you can delete a person, which will cause an automatic update in the Master list.

<img width="600" alt="app" src="https://cloud.githubusercontent.com/assets/3668771/16743340/1b174f7a-47ac-11e6-878e-350c38943836.png">

## EXC_BAD_ACCESS

If you get a [demo project][github], get back to Initial commit, run on iPad Pro simulator and try to delete a person you'll hit EXC_BAD_ACCESS in AppDelegate.  A message was sent to an already-freed oject. (I have checked that it's 100% reproducible on iPad Pro simulator and iPad mini, though it might not happen on other devices.)

Let's see what message was it.

<img width="600" alt="msgsend" src="https://cloud.githubusercontent.com/assets/3668771/16744685/78d54710-47b2-11e6-9267-99459b598999.png">

"controllerWillChangeContent:" is a method in NSFetchedResultsControllerDelegate. It looks like an instance of NSFetchedResultsController outlived it's unowned(unsafe) delegate and tried to send it a message - thus causing a crash.

That's surprising since both FRC and it's delegate are held by DetailViewController. When the view controller is removed, FRC should be gone as well. Or did we make a reference cycle somewhere? 

Let's find out with Instruments. When we look at lifecycle of a problematic FRC we can spot an unbalanced retain that's causing an issue.

<img width="600" alt="foundationretain" src="https://cloud.githubusercontent.com/assets/3668771/16745689/461f7ef8-47b7-11e6-9049-69b6e523f505.png">

It turns out fetched results controllers are shortly retained by Notification Center for the time of posting Core Data related notifications. This retain is usually followed by an almost immediate release. If (in this short timeframe) FRC is to be released by the last object holding it, the actual deallocation is postponed until it's released by Notification Center. It means that for a moment we lose control over memory management. In this case it led to a crash. 

## Solution

Simply niling out FRC's delegate in DetailsViewController deinit would solve the issue, but we would have to remember about if in every place NSFetchedResultsController is used. Instead I came up with SafeFetchedResultsController. It overrides delegate property to back it up with weak property. 

When we replace NSFetchedResultsController with SafeFetchedResultsController objects objc_msgSend that before was causing a crash now simply does nothing by sending a message to nil object.

``` swift   
class SafeFetchedResultsController: NSFetchedResultsController {
    
    private weak var safeDelegate: NSFetchedResultsControllerDelegate?
    
    override var delegate: NSFetchedResultsControllerDelegate? {
        get {
            return safeDelegate
        }
        set {
            safeDelegate = newValue
            super.delegate = newValue
        }
    }
}
```   

Line "super.delegate = newValue" might look like not needed here, but actually it is crucial. Super implementation of delegate setter builds a collection of selectors delegate responds to (by testing it with respondsToSelector(:)). If super setter was never called than a collection of valid selectors is empty and no messages are sent to delegate from NSFetchedResultsController.

## unowned(unsafe)

Why unowned(unsafe) delegates are so common in Foundation API? The reason lies in how memory management is handled in Objective-C. There is a centralised list of weak references. It means that loading and destroying weak objects require acquiring locks to avoid race conditions. It makes using weak costly. 

In Swift, on the other hand, there is no centralised list. Instead each object has it's own weak counter. Locks are not needed and performance is greatly improved. Thanks to that using SafeFetchedResultsController makes no harm to app speed.

More on weak references in Swift in a great post by [Mike Ash][mikeash].

## What's next

Safe delegate pattern may be also used in other classes like UIScrollView or UIWebView, if needed.

Get demo project [here][github]. 

[cocoaheads]: http://www.meetup.com/CocoaHeads-Krakow/
[github]: https://github.com/danielgarbien/FetchedResultsCrash
[mikeash]: https://www.mikeash.com/pyblog/friday-qa-2015-12-11-swift-weak-references.html
