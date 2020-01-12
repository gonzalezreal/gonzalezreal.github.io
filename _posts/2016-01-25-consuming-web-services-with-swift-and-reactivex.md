---
layout: single
title: "Consuming Web Services with Swift and ReactiveX"
tags:
  - networking
  - reactive programming
  - rxswift
  - swift
date: 2016-01-25T12:13:20.629Z
---

Let’s forget [Alamofire](https://github.com/Alamofire/Alamofire) or [Moya](https://github.com/Moya/Moya) for a moment and build a web API client from scratch. In the process, we will learn how to model web API requests using an `enum`, map JSON without any third-party library and use [**RxSwift**](https://github.com/ReactiveX/RxSwift) to compose our API calls.

It is much more simple than you think!

## Borders

As an example, we are going to use the [**REST Countries API**](http://restcountries.eu) to list the countries that share borders with another given country.

![Our sample app showing the borders of Germany](/assets/images/1__aVYCt4ynN32PTcwAaWCZOA.png)

## Countries API

In order to find the borders of a given country, we need to issue two requests. First we get the country details using the country name:

```
https://restcountries.eu/rest/v1/name/Germany?fullText=true
```

```json
[
  {
    "name": "Germany",
    "borders": [
      "AUT",
      "BEL",
      "CZE",
      ...
    ],
    "nativeName": "Deutschland",
    ...
  }
]
```

As you can see, the borders are returned in an array of country codes. The second API call will get the countries for those country codes:

```
https://restcountries.eu/rest/v1/alpha?codes=AUT;BEL;CZE
```

```json
[
  {
    "name": "Austria",
    "nativeName": "Österreich",
    ...
  },
  {
    "name": "Belgium",
    "nativeName": "België",
    ...
  },
  {
    "name": "Czech Republic",
    "nativeName": "Česká republika",
    ...
  }
]
```

## Modeling the API

Let’s create a protocol that describes a REST API resource in the most generic way:

```swift
enum Method: String {
    case GET = "GET"
    ...
}

protocol Resource {
    var method: Method { get }
    var path: String { get }
    var parameters: [String: String] { get }
}
```

We could add other stuff like `body`, etc. but this should be enough for our purposes.

All of our requests will be `GET`, and a method to create an `NSURLRequest` based on a `Resource` would come handy. Let’s create a protocol extension for that:

```swift
extension Resource {
    var method: Method {
        return .GET
    }
    
    func requestWithBaseURL(baseURL: NSURL) -> NSURLRequest {
        let URL = baseURL.URLByAppendingPathComponent(path)
        
        // NSURLComponents can fail due to programming errors, so
        // prefer crashing than returning an optional
        
        guard let components = NSURLComponents(URL: URL, resolvingAgainstBaseURL: false) else {
            fatalError("Unable to create URL components from \(URL)")
        }
        
        components.queryItems = parameters.map {
            NSURLQueryItem(name: String($0), value: String($1))
        }
        
        guard let finalURL = components.URL else {
            fatalError("Unable to retrieve final URL")
        }
        
        let request = NSMutableURLRequest(URL: finalURL)
        request.HTTPMethod = method.rawValue
        
        return request
    }
}
```

Nice! [`NSURLComponents`](https://developer.apple.com/library/prerelease/ios/documentation/Foundation/Reference/NSURLComponents_class/index.html) does all the heavy-lifting and converts the parameters dictionary into a URL-escaped query.

With this foundation in place, we can model our Countries API calls using an `enum` that implements the `Resource` protocol:

```swift
enum CountriesAPI {
    case Name(name: String)
    case AlphaCodes(codes: [String])
}

extension CountriesAPI: Resource {
    var path: String {
        switch self {
        case let .Name(name):
            return "name/\(name)"
        case .AlphaCodes:
            return "alpha"
        }
    }
    
    var parameters: [String: String] {
        switch self {
        case .Name:
            return ["fullText": "true"]
        case let .AlphaCodes(codes):
            return ["codes": codes.joinWithSeparator(";")]
        }
    }
}
```

This is really cool. We have leveraged the **enum associated values** to abstract away how the parameters are laid out in the request, and hidden all the hard coded strings.

Now creating a request for our API is really simple:

```swift
let request = CountriesAPI.Name(name: "Germany").requestWithBaseURL(NSURL.countriesURL())
// request = { URL: https://restcountries.eu/rest/v1/name/Germany?fullText=true }
```

## Simple JSON Decoding

JSON decoding in Swift used to be cumbersome, leading to problems like the [Optional Pyramid of Doom](http://blog.scottlogic.com/2014/12/08/swift-optional-pyramids-of-doom.html). There are, literally, hundreds of Swift JSON libraries for that reason.

But with _Swift 1.2_ [things got much simpler](http://nshipster.com/swift-1.2/). Even though there are still some [nice libraries](https://github.com/JohnSundell/Unbox) out there, we are going to implement a simple JSON decoding solution.

First of all, let’s create a protocol for our JSON decodable types:

```swift
typealias JSONDictionary = [String: AnyObject]

protocol JSONDecodable {
    init?(dictionary: JSONDictionary)
}
```

Simple enough. Now we are going to implement some helper functions to decode types conforming to `JSONDecodable` from an array of JSON objects, a single JSON object, and an `NSData` object respectively:

```swift
func decode<T: JSONDecodable>(dictionaries: [JSONDictionary]) -> [T] {
    return dictionaries.flatMap { T(dictionary: $0) }
}

func decode<T: JSONDecodable>(dictionary: JSONDictionary) -> T? {
    return T(dictionary: dictionary)
}

func decode<T:JSONDecodable>(data: NSData) -> [T]? {
    guard let JSONObject = try? NSJSONSerialization.JSONObjectWithData(data, options: []),
        dictionaries = JSONObject as? [JSONDictionary],
        objects: [T] = decode(dictionaries) else {
            return nil
    }
    
    return objects
}
```

Notice the use of `flatMap` to remove the `nil` results of mapping the dictionary to the `JSONDecodable` type.

We are ready to create a **`Country` model** that can be decoded from JSON:

```swift
struct Country {
    let name: String
    let nativeName: String
    let borders: [String]
}

extension Country: JSONDecodable {
    init?(dictionary: JSONDictionary) {
        guard let name = dictionary["name"] as? String,
            nativeName = dictionary["nativeName"] as? String else {
                return nil
        }
        
        self.name = name
        self.nativeName = nativeName
        self.borders = dictionary["borders"] as? [String] ?? []
    }
}
```

The `name` and `nativeName` properties are mandatory and the constructor will fail if they are not present in the JSON object.

Now we can easily create a Country from a given JSON object:

```swift
let dictionary: JSONDictionary = [
    "name": "Spain",
    "borders": ["AND", "FRA", "GIB", "PRT", "MAR"],
    "nativeName": "España"
]
        
if let country: Country = decode(dictionary) {
  // ...
}
```

## The API Client

Let’s start by implementing an error type for our API Client. We need to communicate when the client got a system error or an invalid HTTP status, and also when JSON decoding failed:

```swift
enum APIClientError: ErrorType {
    case CouldNotDecodeJSON
    case BadStatus(status: Int)
    case Other(NSError)
}
```

Covering all the details of a full-featured networking API like [`NSURLSession`](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/URLLoadingSystem/Articles/UsingNSURLSession.html) in a single post is not possible. Let’s just concentrate on the most straightforward way to fetch a network resource:

```swift
let configuration = NSURLSessionConfiguration.defaultSessionConfiguration()
let session = NSURLSession(configuration: configuration)

let task = self.session.dataTaskWithRequest(request) { data, response, error in
  // Handle response
}

task.resume()
```

First we create a `configuration` object and a session based on that object. A [`configuration`](https://developer.apple.com/library/prerelease/ios/documentation/Foundation/Reference/NSURLSessionConfiguration_class/index.html) object defines the behavior and policies for a session: timeouts, caching policies, additional HTTP headers, etc.

Next we create a [data task](https://developer.apple.com/library/prerelease/ios/documentation/Foundation/Reference/NSURLSessionTask_class/index.html) providing our request and a closure that handles the data after it has been fully received. Note that, unless otherwise specified, the session will call the closure from its own private queue.

Let’s create an `APIClient` class to wrap this behavior providing a method that returns an `Observable` instead of taking a closure as a parameter:

```swift
final class APIClient {
    private let baseURL: NSURL
    private let session: NSURLSession
    
    init(baseURL: NSURL, configuration: NSURLSessionConfiguration = NSURLSessionConfiguration.defaultSessionConfiguration()) {
        self.baseURL = baseURL
        self.session = NSURLSession(configuration: configuration)
    }
    
    private func data(resource: Resource) -> Observable<NSData> {
        let request = resource.requestWithBaseURL(baseURL)
        
        return Observable.create { observer in
            let task = self.session.dataTaskWithRequest(request) { data, response, error in
                
                if let error = error {
                    observer.onError(APIClientError.Other(error))
                } else {
                    guard let HTTPResponse = response as? NSHTTPURLResponse else {
                        fatalError("Couldn't get HTTP response")
                    }
                    
                    if 200 ..< 300 ~= HTTPResponse.statusCode {
                        observer.onNext(data ?? NSData())
                        observer.onCompleted()
                    }
                    else {
                        observer.onError(APIClientError.BadStatus(status: HTTPResponse.statusCode))
                    }
                }
            }
            
            task.resume()
            
            return AnonymousDisposable {
                task.cancel()
            }
        }
    }
}
```

Let’s review the creation of `Observable<NSData>`:

*   `Observable.create()` takes a closure that will be executed every time an observer subscribes to the `Observable`. That is, each subscription will trigger a new network request. This concept is called [**Cold Observable**](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/HotAndColdObservables.md).
*   Inside the network request completion closure, we check for errors or invalid [**HTTP status codes**](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes), and notify the observer accordingly using `onError()` and `onNext()` methods.
*   The subscription closure returns an `AnonymousDisposable` object, which encapsulates the code that will be called when the subscription is torn down.

We are ready to introduce a new method that will map the data returned by the network request into our own model objects:

```swift
final class APIClient {
    
    // ...
    
    func objects<T: JSONDecodable>(resource: Resource) -> Observable<[T]> {
        return data(resource).map { data in
            guard let objects: [T] = decode(data) else {
                throw APIClientError.CouldNotDecodeJSON
            }
            
            return objects
        }
    }
}
```

Here comes the beauty of **RxSwift**, we can treat `Observable` as any other container type like `Array` or `Dictionary` and map the contents into something else. In this case, we are leveraging our JSON decoding function to map the data into an array of model objects.

Let’s finish our API client by creating an extension that provides the specific methods we need for our app:

```swift
extension APIClient {
    func countryWithName(name: String) -> Observable<Country> {
        return objects(CountriesAPI.Name(name: name)).map { $0[0] }
    }
    
    func countriesWithCodes(codes: [String]) -> Observable<[Country]> {
        return objects(CountriesAPI.AlphaCodes(codes: codes))
    }
}
```

As you can see, the infrastructure we have created pays off. It is really simple to add new methods to our API client using our core implementation.

## Chaining Requests

Recall that, in order to obtain the list of countries bordering a given country, we need to chain these two requests:

*   First we obtain the country details, including country codes of the countries bordering it.
*   Then we obtain the list of countries for those country codes.

We can use `flatMap` or `flatMapLatest` to chain network requests or any other asynchronous operation:

![flatMap in action](/assets/images/1__p__EtXgsrsOIjtLEhGMDkjQ.jpeg)

The `flatMap` operator will transform the items emitted by an `Observable` into `Observable`s, then flatten the emissions from those into a single `Observable`.

## The View Model

The view model for our application main screen needs to expose an `Observable` that sends an array of borders when the network requests complete.

We need to chain `countriesWithCodes()` after `countryWithName()` and then map the resulting countries to our `Border` type. We also have to make sure that results are delivered in the main thread, and that multiple subscriptions won’t trigger additional network requests:

```swift
typealias Border = (name: String, nativeName: String)

class BordersViewModel {
    let borders: Observable<[Border]>
    
    init(countryName: String) {
        // ...
        
        self.borders = client.countryWithName(countryName)
            // Get the countries corresponding to the alpha codes
            // specified in the `borders` property
            .flatMap { country in
                client.countriesWithCodes(country.borders)
            }
            // Catch any error and print it in the console
            .catchError { error in
                print("Error: \(error)")
                return Observable.just([])
            }
            // Transform the resulting countries into [Border]
            .map { countries in
                countries.map { (name: $0.name, nativeName: $0.nativeName) }
            }
            // Make sure events are delivered in the main thread
            .observeOn(MainScheduler.instance)
            // Make sure multiple subscriptions share the side effects
            .shareReplay(1)
    }
}
```

## The UI

Thanks to the extensions provided by **RxCocoa**, we can bind our view model `borders` property with the table view using a few lines of code:

```swift
private func setupBindings() {
    // ...
    
    viewModel.borders
        .bindTo(tableView.rx_itemsWithCellFactory) { tableView, index, border in
            let cell: BorderCell = tableView.dequeueReusableCell()
            cell.border = border
            
            return cell
        }
        .addDisposableTo(disposeBag)
}
```

Wait… Where are the `UITableViewDataSource` methods? What kind of black magic is this? This ‘magic’ is provided by the [RxDataSourceStarterKit](https://github.com/ReactiveX/RxSwift/blob/develop/RxExample/RxDataSourceStarterKit), a set of classes that implement fully functional reactive data sources for `UITableView` and `UICollectionView`, included with **RxCocoa**.

Binding to a reactive data source requires an observable whose next’s values are arrays and a closure that will receive each item and return the corresponding cell.

By the way, if you are wondering why the `dequeueReusableCell` method is not taking any identifier as a parameter, check out [this post](/2015/12/31/ios-cell-registration-reusing-with-swift-protocol-extensions-and-generics.html).

## What’s Next?

If you’re curious about how to write unit tests for what we just did, take a look at the complete sample code, which you can find [**here**](https://github.com/gonzalezreal/Borders).

The approach is to stub successful and failed requests with the help of [OHHTTPStubs](https://github.com/AliSoftware/OHHTTPStubs) to test the core [`objects` method in `APIClient`](https://github.com/gonzalezreal/Borders/blob/master/BordersTests/APIClientTests.swift#L42).

Once we have that, testing the Country API is just a matter of checking that the `Resource` protocol methods are [returning the appropriate values](https://github.com/gonzalezreal/Borders/blob/master/BordersTests/CountriesAPITests.swift#L14).