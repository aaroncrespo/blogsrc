---
layout: post
title: "Towards Effective Unit Tests in Swift (and iOS in General)"
date: 2015-10-28 09:45
comments: true
categories: quality
---

I often pose a question to fellow engineers: “Given a Magic Engineer Wand™ what’s one thing you would improve about how apps get built at your company?”. Usually the answers I get boil down to something related to poorly tested code or code missing tests. Often they’ll respond with “I wish we had more unit tests”, “I wish we had better unit tests”, or “I wish it was easier for me to write unit tests with my codebase”, “Slow releases due to regressions”. Crashes suck and one good measure of the health of your code is your test coverage and test suite. Hopefully in this post I can show you some strategies to decouple your code and your tests, making it easier to write healthier apps.

Let’s start with some guiding principles and end with a couple pitfalls to watch out for.

# Single Use

A class should only have a single well defined responsibility. A responsibility, for our sake, can be thought of as requirement where if changed would require code changes in the class. A Render class that initially rendered content to a screen but now is required to render to an image would represent a change in responsibility. An exercise you can practice is listing responsibilities of a class frequent use of the word “and” is a good sign that class is violating this principle.

Take this example:

```swift
struct ArticleService {
    private var cache = NSCache()
    private var client = APIClient(authentication: [])
    private var network = NSURLSession.sharedSession()

    func getLatestArticles(completion: (response: Response) -> Void) {
        let request = APIClient.request("articleListID")
        let cachedResponse = cache.objectForKey(request.url)
        if let cachedResponse = cache.objectForKey(request.url) where !cachedResponse.stale {
            return completion(cachedResponse)
        }
        APIClient(request: request, completion: completion)
    }
}

class ArticleServiceTests: XCTestCase {
    func testArticleService() {
        ArticleService().clearCache()

        //test no network
        testHelper.disableNetwork()

        ArticleService().getLatestArticles {
            XCTAssertNil($0.content)
            XCTAssertNotNil($0.error)
            noNetworkExpectation.fulfill()
        }

        testHelper.enableNetwork()

        ArticleService().getLatestArticles {
            XCTAssertNotNil($0.content)
            XCTAssertNil($0.error)
            normalExpectation.fulfill()
        }
        //test caching
        let mockAPIRequest = Mock(add: APIClient.fetchContent)

        ArticleService().getLatestArticles {
            XCTAssertNotNil($0.content)
            XCTAssertNil($0.error)
            XCTAssertFalse(mockAPIRequest.fullfilled)
            Mock(remove: APIClient.fetchContent)
            cachedExpectation.fulfill()
        }
    }
}
```
It would be a reasonable request for ArticleService’s caching strategy to change. We can also see some tight coupling to cache state in our unit tests. Some of the tests are not unit tests but closer to integration tests. Additionally ArticleService requests content over the network and caches it. Caching in and of itself is a pretty well defined responsibility so let’s improve the code.

```swift
struct ArticleService {
    private var APIClient = ContentAPI(authentication: authData)
    private var network = NSURLSession.sharedSession()
    private var cache: RequestCache

    init(cache: RequestCache) {
        self.cache = cache
    }
    func getLatestArticles() -> Response {
        if cache.responseForRequest(request) {
            completion(cachedResponse)
        } else {
            APIClient(request: request, completion: completion)
        }
    }
}

class APIClientTests: XCTestCase {
    func testArticleQuery() {
        testHelper.disableNetwork()
        APIClient().getLatestArticles {
            XCTAssertNotNil($0.content)
            XCTAssertNil($0.error)
            noNetworkExpectation.fulfill()
        }
        testHelper.enableNetwork()
        //test no network
        testHelper.disableNetwork()
        APIClient().getLatestArticles {
            XCTAssertNotNil($0.content)
            XCTAssertNil($0.error)
            normalExpectation.fulfill()
        }
    }
}

class RequestCacheTests: XCTestCase {
    func testArticleCache() {
        XCTAssertNil(RequestCache().responseForRequest(request))
        RequestCache().cacheResponse(response, forRequest: request)
        XCTAssertNotNil(RequestCache().responseForRequest(request))
        RequestCache().clearCache()
    }
}

class ArticleServiceTests: XCTestCase {
    func testArticleService() {
        ArticleService().getLatestArticles {
            XCTAssertNotNil($0.content)
            XCTAssertNil($0.error)
            normalExpectation.fulfill()
        }
    }
}
```

Small changes in the application’s requirements should only have small effects on the code (but try not to predict the future). Practice keeping classes small and narrowly defined. Compose responsibilities don’t build them out.

# Avoid Editing: Extend
At the root of it we want to write unit tests and systems that are not tightly coupled. We have described how having multiple responsibilities or concerns makes testing and refactoring challenging. Eventually some systems in our app have to interoperate. How can we limit the effects when a requirement changes on consumers of the updated systems? Modularity/Extensibility is one way to help accomplish this. Pieces that can plug in interchangeably will require minimal changes for consumers when they share a narrowly defined interface.

Building off the previous example of changing the cache scheme..

```swift
struct ArticleService {
    private var APIClient = ContentAPI(authentication: authData)
    private var network = NSURLSession.sharedSession()
    private var cache: RequestCache

    init(cache: RequestCache) {
        self.cache = cache
    }

    func getLatestArticles() -> Response {
        if cache.responseForRequest(request) {
            completion(cachedResponse)
        } else {
            APIClient(request: request, completion: completion)
        }
    }
}
```
A caching system generally shares a set of common functions so a requirement to change this service’s cache scheme could look like:

```swift
/// caches requests in memory
class RequestCache {
    // methods to cache content
}
/// caches requests persistantly
class PersistantRequestCache: RequestCache {
    // methods to cache content
    // implimentation extends behavior
}

ArticleService(cache: RequestCache())
ArticleService(cache: PersistantRequestCache())

```
In this case modularity is enabled by a common interface inherited from the superclass. Alternately a protocol can be used to provide the common interface if subclassing is not appropriate.

# Contracts
Modularity is great but sometimes we can’t achieve it merely by subclassing and composing classes. We can design with contracts in mind. These contracts can be based on system requirements, real world requirements or API requirements.

Once established a subclass should not violate the contracts of a parent type. Correctness of the system should not be affected by changing for a more specific type. Consider this as a way to provide a good user experience for people you are sharing the code base with. An engineer should not find any surprises when an API that accepts a UIView parameter is given a UIButton.

```swift
class Rect {
    var height: Int = 0
    var width: Int = 0
}

class Square: Rect {
    override var height: Int {
        get { return self.height }
        set { self.width = newValue
            self.height = newValue }
    }
    override var width: Int {
        get { return self.width }
        set { self.width = newValue
              self.height = newValue }
    }
}

class View {
    var bounds = Rect()
}

class SquareView: View {
    // Swift's type system actually helps us here..
    // setter or getter would violate co-variant type rules enforced by swiftc.
    override var bounds = Square()
}

func getView() -> {
    return SquareView()
}
// If I am vendored a Rect but the underlying implimentation was a Square Im going to be "surprized" by this height and width behavior
let view: View = getView()
view.bounds.height = 20
view.bounds.height = 10
```

# Divide Large Interfaces
It’s easier to grow code than shrink code but if you follow the previous guides ending up with large interfaces becomes a challenge. Small well defined modular classes and systems naturally have an API with a small surface area for their consumers.

If you are faced with a large interface step one to improving the situation would be splitting it apart along concerns/responsibilities. This starts you on the road where you can begin extracting the behaviors into tighter defined classes. As an aside this is one of the reasons many people end up UIViewControllers that are a challenge to unit test. All they do is manage view Infrastructure, but managing is exposed as a very wide API that might require many calls to hook into. Decoupling code by moving logic into extensions or protocols and extracting those unit into subcomponents step by step will improve the codebase.

# Invert Dependencies
A high level module should avoid depending on concrete implementations of its sub components. These dependencies should be managed via abstraction. A ViewController does not need to care about implementation details of fetching from a database. This has a synergy with extensibility depending on abstractions (protocols) instead of concrete implementations (classes) allows any class to extend a consumer simply by exposing the same interface. By coupling to abstractions implementation details are more isolated. If you buy into this principle heavily your unit tests requires minimal mocking.

```swift
// Before
class CommicService: FavoriteAbleServiceProtocol, FetchingServiceProtocol {
    override var cache: SearchableCache {
        return SearchableCache()
    }
}
class GalleryService: FavoriteAbleServiceProtocol, FetchingServiceProtocol {

}
class ArticleService: FavoriteAbleServiceProtocol, LiveServiceProtocol, FetchingServiceProtocol {

}
class HomePageViewController {
    var articleService = ArticleService()
    var comicService = ComicService()
    var galleryService = GalleryleService()
    func viewDidLoad() {
        //todo
        draw(articleService.getLatest.content)
        draw(comicService.getLatest.content)
        draw(galleryService.getLatest.content)
    }

    func beginLiveContent(contentID: String) {
        articleService.observe(observer: self, contentID: contentID) {
            self.updateTicker($0)
        }

    }
    func draw(content: [AnyObject]?) { }
    func updateTicker(content: [AnyObject]?) { }
}

// After
protocol Service: class {}

class CommicService: Service, FavoriteAbleServiceProtocol, FetchingServiceProtocol {
    override var cache: SearchableCache {
        return SearchableCache()
    }
}
class GalleryService: Service, FavoriteAbleServiceProtocol, FetchingServiceProtocol {

}
class ArticleService: Service, FavoriteAbleServiceProtocol, LiveServiceProtocol, FetchingServiceProtocol {

}

class HomePageViewController {
    var service: FetchingServiceProtocol?
    func viewDidLoad() {
        //todo
        for service in services {
            draw(services.latest)
        }
    }

    func beginLiveContent(contentID: String) {
        for case let service as LiveServiceProtocol in services {
            service.observe(observer: self, contentID: contentID) {
                self.updateTicker($0)
            }
        }
    }
    func draw(content: [AnyObject]?) { }
    func updateTicker(content: [AnyObject]?) { }
}
```

# A Pitfall
For certain patterns singletons make it easy to compartmentalize parts of your app in a nice abstraction. Only one copy of your app is ever running at a time so the pattern is useful and sometimes safe for sharing state between parts of your app with a tightly defined API into a shared state. However, the flow of state through an app is one of your more valuable type of tests to write and singletons can rob you of easily maintaining an isolated model of that flow. Apple sample code the starting point for many an iOS developer sometimes makes it hard by often not including unit tests or not discussing how consumers of their singletons should go about unit testing. Many Cocoa API are singleton-like in their behavior and heavily used: NSUserDefaults, NSNotificationCenter, UIApplication.

Using some of the above principles above we can alleviate these issues. If we design a dependency to an abstract interface it’s straightforward to avoid polluting the global state of other consumers by passing in dependencies that are not shared (inversion, contracts, and extension).

# Challenges
That’s all great but some of these constructors are starting to become huge:

```swift
PHBViewController(service: PHBService(),
      stateMachine: PHBRuleMachine.currentState(),
      analytics: PHBAnalytics())
```
“That doesn’t really scale or it’s annoying to type all that” one might say. To that I say up to a point “true”, and past a certain point you might want help to setup your different dependencies between your environments. Luckily most apps don’t need this right away. These are techniques you can work in as needed. They are also some good problems to have because your app or your team is growing. Avoid architecting forever but be on the looking for areas where you decouple things. Most teams budget some amount of effort to a refactor step right?

This is also where judgment starts to come into play you have many options with how you architect for testability just realize and be conscious of the fact that given these choices most people fall on the wrong side there is no Magic Engineer Wand™.

Swift specifically has some extra tools to help manage this as well.

Default parameter values allow you to provide over rideable default dependencies

```swift
class PHBViewController{
    convenience init(service: PHBService = PHBRemoteService(),
        stateMachine: PHBRuleMachine = PHBRuleMachine().currentState,
        analytics: PHBAnalytics? = PHBLazyAnalyticService()) {

    }
}

PHBViewController(service: PHBService(),
    stateMachine: PHBRuleMachine.currentState(),
    analytics: PHBAnalytics())

PHBViewController(service: PHBTestMockService(),
    stateMachine: PHBTestRuleMachine.appSuspendedInFlow(),
    analytics: nil)

PHBViewController(service: PHBTestMockService())

PHBViewController()
```
Additionally lazy closure properties have a nice side effect that can aid us in managing dependencies without exposing the internal workings of our classes:

```swift
struct Foo {
    private let dependency: Int = 1
    lazy var dependant: Int =  {
        return self.dependency * 6
    }()
}

var foo = Foo()
print(foo.dependency)  // 1
foo.dependant = 2
print(foo.dependant)   // 2

```
Here we can see that the behavior of the dependent structure can be changed without touching the source of its dependent behavior (extend versus edit). The result of the closure is not recalculated so new dependencies may need to be set as needed.

# Summary
My ulterior motive for this post was to describe SOLID principles in the framework of unit testing architecture. More generally following the principles leads to easier to maintain change and extend code without throwing out piles of unit tests every release. Also threw in some bonus pitfalls that can make isolating and intuiting flow of state in each of your unit test cases.

<pre class="language-swift">Single Use :      	        S   Single responsibility principle
Avoid Editing: Extend :   	O   Open/closed principle
Contracts :       		L   Liskov substitution principle
Divide Large Interfaces :       I   Interface segregation principle
Invert Dependencies :     	D   Dependency inversion principle
</pre>

## Additional Reading:

* https://github.com/andymatuschak/refactor-the-mega-controller
* https://developer.apple.com/sample-code/wwdc/2015/?q=Advanced%20User%20Interfaces
* http://typhoonframework.org
* https://en.wikipedia.org/wiki/SOLID_(object-oriented_design)
* Uncle Bob
