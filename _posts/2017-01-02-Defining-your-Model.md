---
layout: post
title:  "Defining your Model"
date:   2017-01-02
---

The model layer is the core component of your application. While serialization technologies (e.g. JSON, XML), transport protocols (e.g. HTTP, WebSockets) and even databases (e.g. SQL, NoSQL, MySQL, MongoBD) can be changed in your application, your model is your business logic and as such is less likely to change in a well-defined application. In this chapter you will learn different techniques with examples for properly defining your model, with the goal of high performance, code reusability, and low memory footprint.

# Attributes

In a model entity, values like text, numbers, links and data are attributes. In Swift, Protocol Oriented Programming improves the programming model with value types, as well as lower memory footprint for value types and static dispatch of methods. We can further optimize our model by using the right type of value in certain situations. Lets start by representing a user in Swift.

```
let AdminUserType = "admin"
let SpeakerUserType = "speaker"
let AttendeeUserType = "attendee"

/// A user.
public struct Person {
    
    public let identifier: String
    
    public let created: String
    
    public var firstName: String
    
    public var lastName: String
    
    public var userType: String
}

let user = Person(identifier: "32A8ACDD-C74A-4FDC-84C1-13F139E783D0", created: "\(Date())", firstName: "John", lastName: "Apple", userType: AdminUserType)

print(user)
```

In this example, the `firstName` and `lastName` values are variable strings, while `identifier` is a constant UUID string and `created` is the date when the user was created. We should see the following output for your user.

```
Person(identifier: "32A8ACDD-C74A-4FDC-84C1-13F139E783D0", created: "2016-10-23 07:16:04 +0000", firstName: "John", lastName: "Apple", userType: "admin")
```

This description is very informative and we can see the user has valid values in this case. However if our identifiers are UUIDs, it is still possible to provide an invalid or empty string, the same case being for `date`. The `userType` attribute is even more constrained and only accepts three possible values. Another issue is memory footprint. For this simple `Person` struct, we are allocating 5 string buffers on the heap. In this case the `userType` property is using a singleton string buffer because we assigned it from a static string. But if we were to created this entity and populate its values from the JSON in a HTTP request, or values in our database, the string would be dynamically allocated on the heap and always point to a different buffer, despite having the same value. Lets refactor this model to make is safer and use less memory.

```
import Foundation

public enum UserType: String {
    
    case admin
    case speaker
    case attendee
}

/// A user.
public struct Person {
    
    public let identifier: UUID
    
    public let created: Date
    
    public var firstName: String
    
    public var lastName: String
    
    public var userType: UserType
}

let identifier = UUID(uuidString: "32A8ACDD-C74A-4FDC-84C1-13F139E783D0")!

let user = Person(identifier: identifier, created: Date(), firstName: "John", lastName: "Apple", userType: .admin)

print(user)

print("Admin string value: \(UserType.admin.rawValue)")
print("Admin is \(unsafeBitCast(UserType.admin, to: UInt8.self))")
print("Speaker is \(unsafeBitCast(UserType.speaker, to: UInt8.self))")
print("Attendee is \(unsafeBitCast(UserType.attendee, to: UInt8.self))")
```

Running this code will show the following output:

```
Person(identifier: 32A8ACDD-C74A-4FDC-84C1-13F139E783D0, created: 2016-10-23 07:16:04 +0000, firstName: "John", lastName: "Apple", userType: UserType.admin)
Admin string value: admin
Admin is 0
Speaker is 1
Attendee is 2
```

In this revised version, the user represents the same data as before. However, `identifier` is a 128 bit `UUID` which ensures we use less memory than its string counterpart, as well as using memory on the stack instead of a string buffer on the heap. Using a `UUID` instead of a `String` provides more safety without any validation overhead, and ensures we can only create a `Person` by providing a valid `UUID`. We also use less memory by making `created` a `Date`, which makes it a `Double` value on the stack, instead of a string on the heap. For the `UserType` attribute, we created an enumeration which will enforce that only the three posible string values can be provided when creating a `Person`. In addition to the input validation, when an instance of `Person` is copied, only a single byte is being copied for the `UserType` attribute. While `UserType` represents a string value, it is internally stored as a byte (`UserType.admin` is `0x00`, ``UserType.speaker` is `0x01` and `UserType.attendee` is `0x02`). Keep this in mind when defining your model. Using the correct types can improve application performance, and require less code for input validation.

