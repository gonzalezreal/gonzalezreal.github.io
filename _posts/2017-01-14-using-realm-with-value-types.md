---
layout: single
title: "Using Realm with Value Types"
excerpt: >-
  We explore how to build a Data Persistence Layer on top of Realm, using struct
  based models and type-safe queries.
tags:
  - persistence
  - value types
  - realm
  - swift
date: 2017-01-14T23:35:12.783Z
---

![](/assets/images/1__CWxM8S64o55tmvBhaA5hTg.png)

In this post I’d like to explore how to build a **Data Persistence Layer** on top of **Realm**, using `struct` based models and type-safe queries.

In case you have been frozen in carbonite for the past couple of years, [Realm](https://realm.io/products/realm-mobile-database/) is a database technology built from scratch for mobile devices, and the de facto alternative to **Core Data** in the Apple world.

Like Core Data, Realm requires **subclassing** to define model objects and allows using them only on the thread which they were created. These and other requirements often add complexity and usually increase the coupling with Realm everywhere in your app.

That’s why it is usually a good idea to treat these frameworks as an implementation detail, abstracting them away under a persistence layer.

## Models

As an example, let’s create a persistence layer to store comic-book superhero characters with their publishers. We can use a couple of structs to define our models:

```swift
public struct Publisher {  
    public let identifier: Int  
    public let name: String  
}

public struct Character {  
    public let identifier: Int  
    public let name: String  
    public let realName: String  
    public let publisher: Publisher?  
}
```

We still need to create the corresponding `RealmSwift.Object` subclasses, although we will only use them as a mere transportation mechanism and hide them from the rest of the app.

```swift
final class PublisherObject: Object {  
    dynamic var identifier = 0  
    dynamic var name = ""

    override static func primaryKey() -> String? {  
        return "identifier"  
    }  
}

final class CharacterObject: Object {  
    dynamic var identifier = 0  
    dynamic var name = ""  
    dynamic var realName = ""  
    dynamic var publisher: PublisherObject?

    override static func primaryKey() -> String? {  
        return "identifier"  
    }  
}
```

## Mapping

We need a mechanism to transform our `struct` based models into their corresponding Realm objects. Let’s define a protocol for that:

```swift
public protocol Persistable {
    associatedtype ManagedObject: RealmSwift.Object

    init(managedObject: ManagedObject)
    func managedObject() -> ManagedObject  
}
```

Any type conforming to `Persistable` will be able to be serialized to and from the corresponding `RealmSwift.Object` subclass.

Here is the code that makes our `Character` model conform to `Persistable`:

```swift
extension Character: Persistable {
    public init(managedObject: CharacterObject) {  
        identifier = managedObject.identifier  
        name = managedObject.name  
        realName = managedObject.realName  
        publisher = managedObject.publisher  
            .flatMap(Publisher.init(managedObject:))  
    }

    public func managedObject() -> CharacterObject {  
        let character = CharacterObject()

        character.identifier = identifier  
        character.name = name  
        character.realName = realName  
        character.publisher = publisher?.managedObject()

        return character  
    }  
}
```

## Creation

With these tools in place, we are ready to implement the insertion methods of our persistence layer.

But first, let’s review how objects are inserted using Realm:

```swift
// Create a Publisher object
let publisher = PublisherObject()  
publisher.id = 1  
publisher.name = "Marvel"

// Get the default Realm
let realm = try! Realm()

// Add to the Realm
try! realm.write {  
    realm.add(publisher)  
}
```

There are a couple of things we can abstract here. On the one hand, we have the database itself, and on the other the operations that we can perform inside a write transaction.

Even though both concepts are provided by the `Realm` object, we are going to encapsulate them into separate types: `Container` and `WriteTransaction` respectively.

Let’s start with the `WriteTransaction` type:

```swift
public final class WriteTransaction {
  private let realm: Realm

  internal init(realm: Realm) {  
    self.realm = realm  
  }

  public func add<T: Persistable>(_ value: T, update: Bool) {  
    realm.add(value.managedObject(), update: update)  
  }  
}
```

Notice that we made the constructor `internal`, as only the `Container` will create instances of this class. Other operations we can add to the `WriteTransaction` interface are `delete` and `update`.

Implementing the `Container` is pretty straightforward:

```swift
public final class Container { 
  private let realm: Realm

  public convenience init() throws {  
    try self.init(realm: Realm())  
  }

  internal init(realm: Realm) {  
    self.realm = realm  
  }

  public func write(_ block: (WriteTransaction) throws -> Void) throws {  
    let transaction = WriteTransaction(realm: realm)  
    try realm.write {  
      try block(transaction)  
    }
  }
}
```

Now we have the minimal infrastructure to insert `Character` and `Publisher` values:

```swift
let character = Character(  
  identifier: 1455,  
  name: "Iron Man",  
  realName: "Tony Stark",  
  publisher: Publisher(identifier: 1, name: "Marvel")  
)

let container = try! Container()

try! container.write { transaction in  
    transaction.add(character)  
}
```

So far this is looking good. We are successfully persisting our `struct` based models, and there is no coupling with **Realm** on the call site.

## Partial Updates

With what we have implemented so far, if we want to update a character in the database, we have to provide the full set of values:

```swift
// Updating Iron Man's real name
let updatedCharacter = Character(  
    identifier: 1455,  
    name: "Iron Man",  
    realName: "Anthony Edward Stark",  
    publisher: Publisher(identifier: 1, name: "Marvel")  
)

try! container.write { transaction in  
    transaction.add(updatedCharacter, update: true)  
}
```

We could improve this situation by making the `Character` struct mutable, but that will still require fetching the model before updating it.

Fortunately, Realm provides a way to **partially update** objects by passing a subset of the values, along with the primary key:

```swift
try! realm.write {  
    realm.create(  
        CharacterObject.self,  
        value: [  
            "identifier": 1455,  
            "realName": "Anthony Edward Stark"
        ],  
        update: true  
    )  
}
```

How can we pass a subset of values in a type-safe manner?

We could use an `enum` with associated values to model the different properties and their values. By using that approach, a partial update will look like this:

```swift
try! container.write { transaction in  
    transaction.update(Character.self, properties: [  
        .identifier(1455),  
        .realName("Anthony Edward Stark")  
    ])  
}
```

It seems like a good idea, and it will bring auto-completion as a bonus. We will need to add a few things to make this work.

Let’s start by creating a protocol that provides the internal representation of a **property value**:

```swift
public typealias PropertyValuePair = (name: String, value: Any)

public protocol PropertyValueType {  
    var propertyValuePair: PropertyValuePair { get }  
}
```

We will use this protocol to implement `WriteTransaction.update` later.

It makes sense to add an associated type to `Persistable` for the type providing the `enum`, as it will be tightly related to the model itself.

```swift
public protocol Persistable {
  associatedtype ManagedObject: Object  
  associatedtype PropertyValue: PropertyValueType  
  ...
}
```

Now we are ready to implement the `update` method in `WriteTransaction`:

```swift
public final class WriteTransaction {  
  ...  
  public func update<T: Persistable>(_ type: T.Type, values: [T.PropertyValue]) {  
    var dictionary: [String: Any] = [:]

    values.forEach {  
      let pair = $0.propertyValuePair  
      dictionary[pair.name] = pair.value  
    }  
          
    realm.create(T.ManagedObject.self, value: dictionary, update: true)  
  }  
}
```

The implementation simply iterates over the values, querying for the internal representation and adding it to a dictionary that is passed to `Realm.create`.

Finally, we need to implement the `enum` for the `Character` property values:

```swift
extension Character {
  public enum PropertyValue: PropertyValueType {  
    case identifier(Int)  
    case realName(String)  
    ...

    public var propertyValuePair: PropertyValuePair {  
      switch self {  
      case .identifier(let id):  
        return ("identifier", id)  
      case .realName(let realName):  
        return ("realName", realName)  
        ...  
      }  
    }  
  }  
}
```

Now we have an elegant and type-safe way for partially updating objects in our Persistence Layer.

## Querying

We still lack an essential functionality in our Persistence Layer: **fetching data**.

Let’s review how data is fetched using Realm:

```swift
// All the Marvel characters sorted by name
let results = realm.objects(CharacterObject.self)  
    .filter("publisher.name == Marvel")  
    .sorted(byProperty: "name")
```

This is very similar to the problem we were facing when dealing with partial updates. We need to find a way to pass a **predicate** and a set of **sort descriptors** in a type-safe manner.

We could go crazy and implement a DSL for fully featured type-safe queries, but let’s take a more pragmatic approach and implement something simpler.

As we did with the partial updates, we will use an `enum` with associated values to express the different queries that we can apply to a particular model in our Persistence Layer.

First, we create the protocol that will provide the internal representation of a query:

```swift
public protocol QueryType {  
    var predicate: NSPredicate? { get }  
    var sortDescriptors: [SortDescriptor] { get }  
}
```

Then we add an associated type to `Persistable` that must conform to that protocol:

```swift
public protocol Persistable {  
    associatedtype ManagedObject: Object  
    associatedtype PropertyValue: PropertyValueType  
    associatedtype Query: QueryType  
    ...  
}
```

Recall that Realm will return `Results<Object>` as the result of a query. We need to create a new type that wraps those results and maps them to our `struct` based models:

```swift
public final class FetchedResults<T: Persistable> {
  internal let results: Results<T.ManagedObject>

  public var count: Int {  
    return results.count  
  }

  internal init(results: Results<T.ManagedObject>) {  
    self.results = results  
  }

  public func value(at index: Int) -> T {  
    return T(managedObject: results[index])  
  }  
}
```

Now we are ready to add a method in `Container` that returns the resulting values of a **query**:

```swift
public func values<T: Persistable> (_ type: T.Type, matching query: T.Query) -> FetchedResults<T> {  
  var results = realm.objects(T.ManagedObject.self)

  if let predicate = query.predicate {  
    results = results.filter(predicate)  
  }

  results = results.sorted(by: query.sortDescriptors)

  return FetchedResults(results: results)  
}
```

Let’s create a type for the `Character` queries and implement one to filter by publisher name:

```swift
extension Character {
  public enum Query: QueryType {  
    case publisherName(String)

    public var predicate: NSPredicate? {  
      switch self {  
      case .publisherName(let value):  
        return NSPredicate(format: "publisher.name == %@", value)  
      }  
    }

    public var sortDescriptors: [SortDescriptor] {  
      return [SortDescriptor(property: "name")]  
    }  
  }  
}
```

With all of that in place, here’s how it looks querying characters by publisher name:

```swift
let results = container.values(  
    Character.self,  
    matching: .publisherName("marvel")  
)
```

I would say that gets the job done in a very elegant way.

## Conclusion

We have successfully implemented a simple Persistence Layer using structs and decoupled the underlying persistence technology (Realm) from the rest of the app.

Please note that mapping to value types might not be the best solution if you have a complex object graph and depend on live objects. In those cases, it is better to use managed objects directly and process or transform them for presentation.

The code in this post is available [here](https://gist.github.com/gonzalezreal/9d297c68435b9ee12bb1b009afd41d9a).
{: .notice--info}
