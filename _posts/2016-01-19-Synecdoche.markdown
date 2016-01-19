---
layout: post
title:  "A closure tale"
date:   2016-01-19 18:51:07
categories: ios
---


**a:** speaking of closures, they messed it up  
**d:** who? why  
&nbsp;&nbsp;&nbsp;&nbsp;not sure I follow  
**a:** the name I mean, it's misleading  
**d:** you don't like closures?  
**a:** see [closure wiki:][closure]
> In programming languages, closures (also lexical closures or function closures) are a technique for implementing lexically scoped name binding in languages with first-class functions.  

``` objc
function startAt(x)
    function incrementBy(y)
        return x + y
    return incrementBy
```

**a:** this is a closure from a functional programming perspective  
&nbsp;&nbsp;&nbsp;&nbsp;while in Swift every anonymous function is called a closure  
&nbsp;&nbsp;&nbsp;&nbsp;surely you can use one to make a closure, but  
> The term closure is often mistakenly used to mean anonymous function.

**d:** ok, got it  
**a:** sweet  
**d:** for a broad idea thing they came up with a name that describes something more specific  
&nbsp;&nbsp;&nbsp;&nbsp;you know how it's called?  
&nbsp;&nbsp;&nbsp;&nbsp;this figure of speach?  
**a:** ?  
**d:** a synecdoche  
&nbsp;&nbsp;&nbsp;&nbsp;[Synecdoche, San Francisco][synecdoche] (9/10)  

[synecdoche]: http://www.imdb.com/title/tt0383028/
[closure]: https://en.wikipedia.org/wiki/Closure_(computer_programming)
