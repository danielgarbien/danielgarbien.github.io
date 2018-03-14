---
layout: post
title:  "Downloading and caching images with NSURLSession/NSURLCache"
categories: blog
excerpt: "Native solution that's simple and robust"
tags: [Swift, Networking, Caching]
image:
feature:
date:   2016-08-18 10:39:07
modified:
share: true
---

# Presenting images from the internet is hard

Download process is triggered by user. It has to be done asynchronously. It may be cancelled anytime. Finally for the best user experience images have to be cached. And caching is not trivial.

To lift that burden from their shoulders developers tend to reach for third party libraries like [SDWebImage][sdwebimage], [Alamofire][alamofire] or lately [Kingfisher][kingfisher]. It’s sure tempting to forget about all the above consideration and just

``` swift
imageView.setImageWithURL(imageURL)
```

...but is it the only sane way?

# Go native!

`NSURLSession` comes with caching abilities of `NSURLCache`. It will efficiently cache HTTP responses (accordingly to their Cache-Control headers) including those with images or any other media! Satisfaction guaranteed by Apple. No need to reinvent the wheel.

In fact, one can make use of `NSURLCache` even when responses lack Cache-Control headers. Missing headers may be injected in response in `NSURLSessionDelegate` method

```swift
func URLSession(session: URLSession, dataTask: URLSessionDataTask, willCacheResponse proposedResponse: CachedURLResponse, completionHandler: (CachedURLResponse?) -> Void)
```

# Implementation

_(March, 2018) Updated code samples and the demo project to use Swift 4._

Here's a [gist][gist] with:

* `SessionDelegate` - implementation of `NSURLSessionDelegate` able to inject Cache-Control headers
* `Synchronizer` - `NSURLSession` client that internally uses SessionDelegate to control caching

`Synchronizer` ia able to load resources represented with `Resource` protocol. `Resource` requirementes are very simple: provide `NSURLRequest` and interpret the outcome:

```swift
protocol Resource {
func request() -> URLRequest

associatedtype ParsedObject
var parse: (Data) throws -> ParsedObject { get }
}
```

`Synchronizer` defines it's result type...

```swift
enum SynchronizerResult<Result> {
case Success(Result)
case NoData
case Error(ErrorType)
}
```

...and provides a loading method:

```swift
typealias CancelLoading = () -> Void

func loadResource<R: Resource, Object where R.ParsedObject == Object>
(resource: R, completion: SynchronizerResult<Object> -> ()) -> CancelLoading
```

In order to support images `ImageResource` is introduced. As simple as that:

```swift
struct ImageResource {
let imageURL: URL
}
extension ImageResource: Resource {
func request() -> NSURLRequest {
return URLRequest(URL: imageURL)
}
var parse: (Data) throws -> UIImage? {
return { data in
UIImage(data: data)
}
}
}
```

# Usage

```swift
let MB = 1024 * 1024
let day: TimeInterval = 24 * 60 * 60

// create caching image synchronizer
let imageCache = URLCache(memoryCapacity: 100 * MB, diskCapacity: 100 * MB, diskPath: "images")
let imageSynchronizer = Synchronizer(cacheTime: day, URLCache: imageCache)

// load image into image view
let cancelationBlock = imageSynchronizer.loadResource(ImageResource(URL: imageURL)) { (object) in
if case .Success(let image) = object {
imageView.image = image
}
}

// cancel loading when image view goes off screen
cancelationBlock()
```

So where is my `setImageWithURL` method?

Well, there is none yet. If you ain't scared of singletons you can easily come up with an extension on `UIImageView` that uses shared instance of image synchronizer. You could also take care of cancelling there when imageView goes off screen or internet connection is bad.

# Conslusion

The point is you cut loose a third party dependency. The solution presented here may be used next to or combined with your current networking layer. You poses full control over it. You are in charge now.

# Demo

See it working in the project (Swift 4): [code][github]

<figure>
<img src="/images/avatars.png" alt="image">
</figure>

[github]: https://github.com/danielgarbien/PagedFeed
[gistSessionDelegate]: https://gist.github.com/danielgarbien/8e904b07c07110a502b3116576afaa64
[kingfisher]: https://github.com/onevcat/Kingfisher
[sdwebimage]: https://github.com/rs/SDWebImage
[alamofire]: https://github.com/Alamofire/Alamofire
[gist]: https://gist.github.com/danielgarbien/8e904b07c07110a502b3116576afaa64#file-sessiondelegate-swift
