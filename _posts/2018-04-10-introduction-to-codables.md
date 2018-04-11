---
layout: post
title:  An Introduction to Swift Codables
date:   2018-04-10_13:24:58
description: An introduction to Swift's Codable protcol.
---

<!--  Outline

> _All examples assume Swift 4.1 and Xcode 9.3._

1. An Introduction to Swift’s `Codable` Protocol
	- Brief primer on manual JSON parsing on iOS.
	- High level overview of protocols and how to use them.
	- Manually implementing Decodable/Encodable conformance.
	- You get a lot for free.
	- Decoding/Encoding strategies.
	- Classes vs. Structs
		- Pass by reference vs value types
		- Deterministic property initialization
	- 	`Codable` Network Requests

-->

## Introduction

In almost every application, developers need to interact with data that exists in some intermediary format, whether that be in the form of JSON received from a network request or a property list read from disk, and convert it to a concrete type within the code base. Parsing this intermediary data and mapping to a type can be tedious and error prone. In response, numerous third party solutions have arisen over the years attempting to fill this need though none have gained a foothold large enough to be considered the "standard".

At WWDC 2017, Apple introduced `Codable` to the Swift Standard Library: a simple protocol that solves the problem of converting data between interchange formats (e.g. JSON or XML) and types within our code. This protocol is customizable, easy to use, and (most importantly) a first party solution. To top it all off, Apple has even include pre-built implementations for two of the most popular interchange formats: JSON and Property Lists.

## Vocabulary

- Encode: The process of converting a type to an interchange format.
- Decode: The process of converting an interchange to a type.

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

Before we dive into `Codable`, let's take a brief look at the work required to _manually_ decode a `Person` object. We'll be using only what `Foundation` provides us to avoid pulling any third-party frameworks into the discussion.

After making a network request, the basic process looks like this:  

- In the response, use `JSONSerialization` to deserialize raw data to a `[String: AnyObject]` dictionary.
- Map the key-value pairs in the dictionary to a `Person` object's properties.

Here's the basic logic of the response:

```
// personService is a simple API client that retrieves people from SWAPI
personService.get(personId: 1) { data in
    
    guard let data = data else {
        // handle missing data...
        return
    }

    guard let json = try? JSONSerialization.jsonObject(with: data, options: []) as? [String: AnyObject] else {
        // handle failed deserialization...
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

This looks simple enough but, as you can imagine, this get gets tedious and error prone really fast. The main areas where things can go wrong include:

- handling the raw strings used to parse values from the JSON
- mapping raw values to specific types (e.g. `Date` or `URL`)
- properly handling missing or malformed values
- scaling when dealing with large or complex models








































