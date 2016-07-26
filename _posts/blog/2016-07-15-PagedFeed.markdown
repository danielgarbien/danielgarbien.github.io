---
layout: post
title:  "Paged feed with UICollectionView by example"
categories: blog
excerpt:
tags: [Swift, Architecture]
image:
  feature:
date:   2016-07-15 00:39:07
modified: 
share: true
---

A simple variation on feed with pagination in iOS app. If you'll pardon ascetic aesthetics, it's just a proof of concept, shall you find some clever goodies in the [code][github]. Details below.

(App let you search GitHub users.)

<img width="487" alt="appshot" src="https://cloud.githubusercontent.com/assets/3668771/16858457/3ff64878-4a27-11e6-96d6-df54bc34c733.png">

## Layers

Project is split into 5 groups: Model, 3 actual layers and App. 

<img width="300" alt="screen shot 2016-07-15 at 02 04 10" src="https://cloud.githubusercontent.com/assets/3668771/16859610/8c3ccdf2-4a30-11e6-92a6-9c7486a2028c.png">

Model is super simple and has no business logic to it. It's designed to reflect data. Period. As such it flows from Top layer down to be visualized in UI. (Currently it's unidirectional, there are no write actions.)

Next, we have 3 layers. No layer is aware of any layer below (Network Layer doesn't know about Data Access Layer etc.). On the other hand each layer only knows and talks to a layer that's directly above him (Controllers talk to Data Access Layer, but have no idea about networking).

App layer glues them together.

This kind of design gives very solid division of responsibilities.

## Network Layer - Resource

As defined with a protocol a single Resource object:

* knows where to get it from (path) 
* knows how to get it (parameters)
* can interprete results (parse)

This way all the code sensitive to backend changes lays in one place.

``` swift
protocol Resource {
    var path: String? { get }
    var parameters: [String: AnyObject] { get }
    
    associatedtype ParsedObject
    var parse: (NSData) throws -> ParsedObject { get }
}
```

Extension serves you with a practical URL request creator.

``` swift
extension Resource {
    func requestWithBaseURL(baseURL: NSURL) -> NSURLRequest {
        let URL = baseURL.URLByAppendingPathComponent(path ?? "")
        let components = NSURLComponents(URL: URL, resolvingAgainstBaseURL: false)!
        
        components.queryItems = parameters.map { key, value -> NSURLQueryItem in
            NSURLQueryItem(name: String(key), value: String(value))
        }
        return NSURLRequest(URL: components.URL!)
    }
}
```

In this form resources are very easy to be used by the Synchronizer.

## Network Layer - Caching

One hour long caching is achieved with NSURLCache. It might look like a no go when URL responses come with no Cache-Control headers (which is a case in GitHub API). The solution was to implement NSURLSessionDelegate that injects Cache-Control headers into responses headers before they are cached.

``` swift
func URLSession(session: NSURLSession, dataTask: NSURLSessionDataTask, willCacheResponse proposedResponse: NSCachedURLResponse, completionHandler: (NSCachedURLResponse?) -> Void) {
    switch proposedResponse.response {
    case let response as NSHTTPURLResponse:
        var headers = response.allHeaderFields as! [String: String]
        headers["Cache-Control"] = "max-age=\(cacheTime)"
        
        let modifiedResponse = NSHTTPURLResponse(URL: response.URL!,
                                                 statusCode: response.statusCode,
                                                 HTTPVersion: "HTTP/1.1",
                                                 headerFields: headers)
        let modifiedCachedResponse = NSCachedURLResponse(response: modifiedResponse!,
                                                         data: proposedResponse.data,
                                                         userInfo: proposedResponse.userInfo,
                                                         storagePolicy: proposedResponse.storagePolicy)
        completionHandler(modifiedCachedResponse)
    default:
        completionHandler(proposedResponse)
    }
}
```

Get a gist [here][gistSessionDelegate].

## Data Access Layer

Abstracts network away from Controllers. Additionally it goes all the way to meet pagination. To that end it introduces:

* PagedResource - extends Resource protocol to represent a certain page with a specific size.  
* FeedResult<Page> - a result type of asynchronous completion blocks. When successful, alongside Page result (e.g. array of Users), it contains a handy block to call for a next page.

``` swift
enum FeedResult<Page> {
    case Success(page: Page, nextPage: LoadPageBlock)
    case FeedEnd
    case Error(error: ErrorType, retry: LoadPageBlock)
    
    typealias LoadPageBlock = (completion: FeedResult<Page> -> Void) -> Void
}
```

* LoadingFeedStateMachine<Page> - once initialized and launched it walks you from page 1. to the last one. Sample usage:

``` swift
// Initialize
lazy var loadingStateMachine: LoadingFeedStateMachine<User> = LoadingFeedStateMachine(stateDidChange: self.handleLoadingStateChange)
// Launch
loadingStateMachine.startFeed { completion in
    dataAccess.usersWithQuery(
        "dan",
        sort: .BestMatch,
        pageSize: 30,
        completion: completion)
}
// Handle changes
private func handleLoadingStateChange(state: LoadingFeedState<[User]>) {
    switch state {
    case .Loading(true): resetCollection() // initial load
    case .Succeed(let users): updateCollectionByAppendingUsers(users)
    default: break
    }
}
// Trigger next load (when user scrolls to bottom)
loadingStateMachine.next()
```

Functional, isn't it?

## Controllers

* Column layout implementation works very well with vertically scrolled content of different height items.
* There is a Simple Collection View Data Source generic over Object and Cell types. ([gist][gistDataSource])
* Items collection stack (view controller + data source) is generic over Item type. User model could be replaced with, let's say, Respository at any time.

With this much of genericness classes with actual business logic are very concise and to the point (vide 40 line UsersCollectionViewController). It doesn't have to stop here. For instance Items collection stack could be made generic over Cell class. But this is another, never-ending story...

## App

Project available on [github][github].

[github]: https://github.com/danielgarbien/PagedFeed
[gistSessionDelegate]: https://gist.github.com/danielgarbien/8e904b07c07110a502b3116576afaa64
[gistDataSource]: https://gist.github.com/danielgarbien/b6ff053974d3c2acdcc9d224ff259cc5
