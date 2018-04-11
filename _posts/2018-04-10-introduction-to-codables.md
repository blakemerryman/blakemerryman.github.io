---
layout: post
title:  An Introduction to Swift Codables
date:   2018-04-10_13:24:58
description: An introduction to Swift's Codable protcol.
---

In almost every application, developers need to interact with data that exists in some intermediary format, whether that be in the form of JSON received from a network request or a property list read from disk, and convert it to a concrete type within the code base. Parsing this intermediary data and mapping to a type can be tedious and error prone. In response, numerous third party solutions have arisen over the years attempting to fill this need though none have gained a foothold large enough to be considered the "standard".

At WWDC 2017, Apple introduced `Codable` to the Swift Standard Library: a simple protocol that solves the problem of converting data between interchange formats (e.g. JSON or XML) and types within our code. This protocol is customizable, easy to use, and (most importantly) a first party solution. To top it all off, Apple has even include pre-built implementations for two of the most popular interchange formats: JSON and Property Lists.

This article will focus on JSON due to its ubiquity but most of the concepts discussed would be applicable for other interchange formats, such as property lists.

## Vocabulary

- _Encode_: The process of converting a type to an interchange format (e.g. Model type to JSON).
- _Decode_: The process of converting an interchange format to a type (e.g. JSON to Model type).

## Motivating Example

To illustrate the topics in this article, we will be using a person object received from the [Star Wars API's `/people` endpoint](https://swapi.co/api/people/1/):

```
{
	"name": "Luke Skywalker",
	"url": "https://swapi.co/api/people/1/",
	"created": "2014-12-09T13:50:51.644000Z",
	"birth_year": "112BBY",

	// abbreviated for clarity

}
```

```
class Person: NSObject {
	let name: String
	let url: URL
	let created: Date
	let birthYear: String

	// abbreviated for clarity

	init(name: String, url: URL, created: Date, birthYear: String) {
		self.name = name
		self.url = url
		self.created = created
		self.birthYear = birthYear

		// abbreviated for clarity

		super.init()
	}
}
```

## The Old Way

Before we dive into what `Codable` gets us, let's take a brief look at the work required to _manually_ decode and encode a `Person` object. In order to avoid pulling in third-party libraries, we will only use what Foundation makes available to us.

For our example, we'll be making a simple network request to get a person from SWAPI. After making a network request, the basic decoding process looks like this:

- Use `JSONSerialization` to deserialize raw data to a JSON object.
- Cast this JSON object to a dictionary (e.g. `[String: AnyObject]`).
- Map the key-value pairs in the dictionary to a `Person`'s properties.
- Handle any errors along the way.

Here's the basic logic of the response:

```
// personService is a simple API client that retrieves people from SWAPI
personService.get(personId: 1) { data in
    
    guard let data = data else {
        // handle missing data...
        return
    }

    guard let jsonObject = try? JSONSerialization.jsonObject(with: data, options: []) else {
        // handle failed deserialization...
        return
    }

    guard let json = jsonObject as? [String: AnyObject] else {
        // handle receiving incorrect JSON format...
        return
    }
    
    guard let person = Person(fromJSON: json) else {
    	// handle failed initialization...
    	return
    }
    
    // We now have a person object!
}
```

Here's a closer look at the JSON convenience initializer for `Person`:

```
extension Person {

    /// Convenience initializer used to create `Person` from a JSON dictionary.
    convenience init?(fromJSON dictionary: [String: AnyObject]) {

        // Parse "name" from the JSON.
        guard let name = dictionary["name"] as? String else {
            return nil
        }

        // Parse "url" from the JSON.
        guard let urlString = dictionary["url"] as? String else {
            return nil
        }

        // Convert "url" string to a URL.
        guard let url = URL(string: urlString) else {
            return nil
        }

        // Parse "created" from the JSON.
        guard let createdString = dictionary["created"] as? String else {
            return nil
        }

        // Convert "created" string to a Date.
        guard let created = DateFormatter.iso8601Full.date(from: createdString) else {
            return nil
        }

        // Parse "birth_year" from the JSON.
        guard let birthYear = dictionary["birth_year"] as? String else {
            return nil
        }

        // Call the designated initializer.
        self.init(name: name, url: url, created: created, birthYear: birthYear)
    }

}
```

Let's imaging that we also want the ability to encode a `Person` back to data. Here's what it looks like to manually encode a `Person` back to `Data`:

```
let personDictionary = person.toDictionary()

guard let personData = try? JSONSerialization.data(withJSONObject: personDictionary, options: []) else {
    // handle failure to convert back to data
    return
}

// We now have a person as raw data!
```

Here's a closer look at the dictionary constructor for `Person`:

```
extension Person {

    func toDictionary() -> [String: AnyObject] {
        return [
            "name": self.name as AnyObject,
            "url": self.url.absoluteString as AnyObject,
            "created": DateFormatter.iso8601Full.string(from: self.created) as AnyObject,
            "birth_year": self.birthYear as AnyObject
        ]
    }

}
```

This all seems simple enough but, as you can imagine, it becomes tedious and error prone really fast. The main areas where things can go wrong include:

- working with the raw JSON key strings
- handling non-primitive types in a type-safe manner (e.g. `Date` or `URL`)
- dealing with missing or malformed values
- scaling when encoding/decoding large or complex models

We could expend the energy and fix all of these problem areas but that would take a lot of time and effort. Luckily, Apple has done all of that work for us.

## Enter Codable

Swift 4 introduced `Codable`, a simple protocol that set out to solve the problem of converting data between external formats and types in our code. Taking a closer look, we can see that it's actually a typealias combining two other protocols: `Decodable` and `Encodable`:

```
typealias Codable = Decodable & Encodable
```

The `Decodable` protocol defines what it takes to convert from an external format to a type in our code.

```
/// A type that can decode itself from an external representation.
protocol Decodable {
	init(from: Decoder) throws
}
```

The `Encodable` protocol defines what it takes to convert from a type in our code to an external format.

```
/// A type that can encode itself to an external representation.
protocol Encodable {
  encode(to: Encoder) throws
}
```

Any class, struct, or enum can declare itself as `Codable`. Apple already did the work to conform primitive Swift types, like `String`, `Int`, & `Float`. They even went the extra mile to conform several key Foundation types as well (e.g `Date`, `Data`, and `URL`). Container types like `Array`, `Dictionary`, and `Optional` are also `Codable` so long as all contained elements are `Codable`.

## Conforming to Codable

Conforming our own types to `Codable` is extremely easy in most cases. Using our `Person` example, conformance is as easy as simply _declaring_ our type as `Codable`:

```
class Person: NSObject, Codable {

	// abbreviated for clarity

}
```

The benefit of a first party solution is clear once you realize what the compiler is doing for us here. Let's take a look at the code that is auto-generated for us after declaring ourselves as `Codable`:

```
class Person: NSObject, Codable {

	...

    // All of the following is generated for us by the compiler:

    /// Enumerates the list of keys expected in an external representation.
	enum CodingKeys: String, CodingKey {
        case name
        case url
        case created
        case birthYear
    }

    /// Initializer that handles decoding a type.
    required init(from decoder: Decoder) throws {

        // Grab the container that holds the values keyed by the coding keys...
        let container = try decoder.container(keyedBy: CodingKeys.self)

        // Decode the type-safe values...
        name = try container.decode(String.self, forKey: .name)
        url = try container.decode(URL.self, forKey: .url)
        created = try container.decode(Date.self, forKey: .created)
        birthYear = try container.decode(String.self, forKey: .birthYear)

    }

    /// Method that handles encoding a type.
    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(name, forKey: .name)
        try container.encode(url, forKey: .url)
        try container.encode(created, forKey: .created)
        try container.encode(birthYear, forKey: .birthYear)
    }

}
```

Going back to our network request, here's what decoding looks like in action:

```
personService.get(personId: 1) { data in

    guard let data = data else {
        // handle missing data...
        return
    }

    // Swift's built-in JSON Decoder

    let decoder = JSONDecoder()
    decoder.dateDecodingStrategy = .formatted(.iso8601Full)
    decoder.keyDecodingStrategy = .convertFromSnakeCase

    // Decode a Person...
    do { 
	   	let person = try decoder.decode(Person.self, from: data)
    	// We now have a person object!
    }
    catch {
    	// handle decoding failures...
    }
}
```

Note that the JSONDecoder has a few "decoding strategy" configuration points. These allow us to offload some of the error-prone work to `JSONDecoder`. 

In the above example, we use `dateDecodingStrategy` to specify that we expect dates to come down in the ISO-8601 format. Swift's default ISO-8601 formatter does not support the full spec so we have to specify our own custom date formatter.

In addition, we use `keyDecodingStrategy` to handle converting the JSON's `snake_case` keys (naming convention on the web) to `camelCase` (naming convention on iOS). This is useful because the raw value of the `CodingKeys` must match the JSON key exactly. This is new in Swift 4.1; in the past, we solved this problem by overriding the `CodingKeys` raw values.

We can also customize how data (using `dataEncodingStrategy`) and floats (using `nonConformingFloatEncodingStrategy`) are decoded.

Similarly, let's look at what encoding a `Person` back to data looks like:

```
// Swift's built-in JSON Encoder
let encoder = JSONEncoder()
encoder.dateEncodingStrategy = .formatted(.iso8601Full)
encoder.keyEncodingStrategy = .convertToSnakeCase

do {
    let personData = try? encoder.encode(person)
    // We now have a person as raw data!
}
catch {
    // handle encoding failures...
}
```

Note the encoding strategies that correspond to the decoding strategies discussed earlier.

















