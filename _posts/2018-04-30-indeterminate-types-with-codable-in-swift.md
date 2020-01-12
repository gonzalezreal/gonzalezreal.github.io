---
layout: single
title: "Indeterminate Types with Codable in Swift"
excerpt: We explore two different techniques to handle indeterminate types in a JSON response.
tags:
  - codable
  - enum
  - type erasure
  - solid
  - swift
date: 2018-04-30T20:51:51.488Z
---

![](/assets/images/1__kObb4dzj0arPDMtTF__kFbA.jpeg)

A companion playground for this post is available [here](https://github.com/gonzalezreal/IndeterminateTypesWithCodable).
{: .notice--info}

Time flies. Swift 4.0 was released back in September 2017, and we have been enjoying the `Codable` protocol for a while. And yet, we still have some ground to cover.

Recall that the `Codable` protocol makes it super-easy to encode and decode values conforming to it. Oh, and it comes with **Property List** and **JSON** support. Remember the countless JSON decoding libraries before Swift 4?

In most cases, when you declare a type that adopts `Codable` the compiler does most of the work and **synthesizes conformance**. You may also need to specify a `CodingKeys` enumeration if the JSON keys do not match the property names.

However, there are some scenarios in which it is necessary to implement `Codable` manually. Having a JSON with objects whose type is determined by the value of a `"type"` key is one of those scenarios.

Consider a hypothetical _Messaging API_ with support for different kinds of attachments: image, audio, etc.

```json
{
  "from": "Guille",
  "text": "Look what I just found!",
  "attachments": [
    {
      "type": "image",
      "payload": {
        "url": "http://via.placeholder.com/640x480",
        "width": 640,
        "height": 480
      }
    },
    {
      "type": "audio",
      "payload": {
        "title": "Never Gonna Give You Up",
        "url": "https://audio.com/NeverGonnaGiveYouUp.mp3",
        "shouldAutoplay": true
      }
    }
  ]
}
```

Because Swift is a strongly typed language, we must implement a type for each kind of attachment.

```swift
struct ImageAttachment: Codable {
  let url: URL
  let width: Int
  let height: Int
}

struct AudioAttachment: Codable {
  let title: String
  let url: URL
  let shouldAutoplay: Bool
}
```

As you can see is quite simple, until you have to implement the `Message` type.

```swift
struct Message: Codable {
  let from: String
  let text: String
  let attachments: [???]
}
```

To complete the implementation of `Message`, we must first create an `Attachment` type that can hold `ImageAttachment` or `AudioAttachment` values.

There are several ways of doing this. We are going to explore **two different approaches**, each with its pros and cons.

## Using an Enum with Associated Values

Swift has a language feature that is perfect for this situation. Enumerations can store associated values of any given type.

```swift
enum Attachment {
  case image(ImageAttachment)
  case audio(AudioAttachment)
  case unsupported
}
```

Notice that we are adding `unsupported` in case our _Messaging API_ decides to support new attachment types that we don't know how to handle.

The compiler won’t be able to synthesize conformance to `Codable` in this case, but it is not difficult to do it ourselves.

For the `Decodable` part of `Codable`, we must create a `CodingKeys` enumeration and implement `init(from: Decoder)`.

```swift
extension Attachment: Codable {
  private enum CodingKeys: String, CodingKey {
    case type
    case payload
  }
  
  init(from decoder: Decoder) throws {
    let container = try decoder.container(keyedBy: CodingKeys.self)
    let type = try container.decode(String.self, forKey: .type)
    
    switch type {
    case "image":
      let payload = try container.decode(ImageAttachment.self, forKey: .payload)
      self = .image(payload)
    case "audio":
      let payload = try container.decode(AudioAttachment.self, forKey: .payload)
      self = .audio(payload)
    default:
      self = .unsupported
    }
  }
  ...
}
```

For the `Encodable` part of `Codable`, we have to implement `encode(to: Encoder)` by switching on `self` and letting the associated value encode itself.

```swift
func encode(to encoder: Encoder) throws {
  var container = encoder.container(keyedBy: CodingKeys.self)
  switch self {
  case .image(let attachment):
    try container.encode("image", forKey: .type)
    try container.encode(attachment, forKey: .payload)
  case .audio(let attachment):
    try container.encode("audio", forKey: .type)
    try container.encode(attachment, forKey: .payload)
  case .unsupported:
    let context = EncodingError.Context(codingPath: [], debugDescription: "Invalid attachment.")
    throw EncodingError.invalidValue(self, context)
  }
}
```

Finally, we need to specify the `attachments` type in `Message`.

```swift
struct Message: Codable {
  let from: String
  let text: String
  let attachments: [Attachment]
}
```

Inspecting the attachments of any message is a simple task.

```swift
let decoder = JSONDecoder()
let message = try decoder.decode(Message.self, from: json)

for attachment in message.attachments {
  switch attachment {
  case .image(let image):
    // do something with the image
  case .audio(let audio):
    // do something with the audio
  case .unsupported:
    // ignore unsupported attachments
  }
}
```

To support a new type of attachment, we would need to perform the following tasks:

1.  Implement a type for the new attachment.
2.  Add a new case to the `Attachment` enum and update its `Codable` implementation.

The second point is where the major drawback of this implementation lies. The **Open/Closed Principle** states:

> Software entities should be open for extension, but closed for modification.

We should find a way to implement `Attachment` so that **it does not need modification when we have to support a new type of attachment**.

## Back to Any

We have no choice but to use `Any` to store attachments if we want to avoid any future modifications.

```swift
struct Attachment {
  let type: String
  let payload: Any?

  private enum CodingKeys: String, CodingKey {
    case type
    case payload
  }
  ...
}
```

It may look like a step back, but bear with me, it’s not that bad.

The `Codable` protocol was not made to work with `Any`. We need a way to register attachment types.

```swift
Attachment.register(ImageAttachment.self, for: "image")
Attachment.register(AudioAttachment.self, for: "audio")
```

The implementation of `register` should store the logic uniformly to decode and encode the given attachment type, keyed by the type name in the JSON. Recall how we decode an attachment:

```swift
try container.decode(ImageAttachment.self, forKey: .payload)
```

How the heck do we store that logic uniformly in a `Dictionary`?

## Type Erasure to the Rescue

To store the decoding and encoding logic uniformly, we have to erase type information. Let’s define the signature for our encoding and decoding closures.

```swift
typealias AttachmentDecoder = (KeyedDecodingContainer<CodingKeys>) throws -> Any
typealias AttachmentEncoder = (Any, inout KeyedEncodingContainer<CodingKeys>) throws -> Void
```

The `AttachmentDecoder` closure takes a container and returns the result of decoding its contents.

The `AttachmentEncoder` closure takes the attachment payload and encodes it using the given container.

Now we need two private static properties to store the decoders and the encoders respectively.

```swift
private static var decoders: [String: AttachmentDecoder] = [:]
private static var encoders: [String: AttachmentEncoder] = [:]
```

Finally, we can implement our `register` method to store the decoding and encoding closures for a given attachment type.

```swift
static func register<A: Codable>(_ type: A.Type, for typeName: String) {
  decoders[typeName] = { container in
    try container.decode(A.self, forKey: .payload)
  }
  encoders[typeName] = { payload, container in
    try container.encode(payload as! A, forKey: .payload)
  }
}
```

## A Better Codable Implementation

Now that we have a method to register attachment types, implementing `Codable` is as easy as relying on the `decoders` and `encoders` static properties.

To decode an `Attachment`, we access the value of the `type` property and use it to find the corresponding decoder. Then, we use that decoder to decode the payload.

```swift
init(from decoder: Decoder) throws {
  let container = try decoder.container(keyedBy: CodingKeys.self)
  type = try container.decode(String.self, forKey: .type)

  if let decode = Attachment.decoders[type] {
    payload = try decode(container)
  } else {
    payload = nil
  }
}
```

To encode an `Attachment`, we first encode its type and then find the corresponding encoder, which is then used to encode the payload.

```swift
func encode(to encoder: Encoder) throws {
  var container = encoder.container(keyedBy: CodingKeys.self)
  try container.encode(type, forKey: .type)
  if let payload = self.payload {
    guard let encode = Attachment.encoders[type] else {
      let context = EncodingError.Context(codingPath: [], debugDescription: "Invalid attachment: \(type).")
      throw EncodingError.invalidValue(self, context)
    }
    try encode(payload, &container)
  } else {
    try container.encodeNil(forKey: .payload)
  }
}
```

## Example Use

It may surprise you, but inspecting the attachments of a message is not very different from how we did it with the **enumeration** solution.

It is important that we register the supported attachments before we do any decoding or encoding.

```swift
Attachment.register(ImageAttachment.self, for: "image")
Attachment.register(AudioAttachment.self, for: "audio")
```

As we can use a `switch` statement for type casting, the rest is more or less the same.

```swift
let decoder = JSONDecoder()
let message = try decoder.decode(Message.self, from: json)

for attachment in message.attachments {
  switch attachment.payload {
  case let image as ImageAttachment:
    // do something with the image
  case let audio as AudioAttachment:
    // do something with the audio
  default:
    // ignore unsupported attachments
  }
}
```

To support a new attachment, we simply need to **implement a new type for the attachment and register it**. There is no need to modify the implementation of `Attachment`.

## Conclusion

Enumerations with associated values are one of my favorite Swift language features. However, I don’t think they are well suited for this use case.

Using `Any` for the `Attachment` payload and implementing a registration mechanism for attachment types is resilient to modifications while maintaining a similar development experience.
