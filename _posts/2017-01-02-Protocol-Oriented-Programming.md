---
layout: post
title:  "Protocol Oriented Programming"
date:   2017-01-02
---

# Object Oriented Programming vs Protocol Oriented Programming

Object Oriented Programming is the most widely used modern approach to programming software. C++, C#, Java and many other modern languages and frameworks follow this pattern. Swift’s predecessor, Objective-C, was an Objected Oriented Programming extension over C. Classes allow for encapsulation, access control, abstraction, namespaces, expressive syntax, and extensibility. Extensibility is mainly implemented in the form of subclassing in most OOP languages. Swift is a unique language in the sense that virtually all its types, structures, enumerations, classes, and protocols, support these features. Extensions can be applied to all types in Swift. All types support the expressive syntax that methods provide, and have the concept of `self`. Another feature in Swift is that all types support polymorphism and inheritance of behaviors in one form or another. One of the main advantages of classes in OOP, vs another pattern like procedural programming in C, is that classes allow for inheritance, which in turn allow for high code reusability. Without code reusability, most of the software that powers the products we use simply could not be built. Both seasoned iOS developers and developers coming from other languages will need a paradigm switch to effectively use Swift, and take full advantage of its performance optimizations. While Swift completely supports OOP, its standard library is built on a brand new programming paradigm called Protocol Oriented Programing. This new paradigm advocates for only using classes when absolutely necessary, and instead preferring value types like structures and enumerations. Structures are viewed as first class replacement for objects, and inheritance (even for classes) should be achieved with the use of protocol extensions.

# Shared State vs Copy-on-Write

Classes in OOP have a couple of shortcomings, the foremost being shared state. Objects are pointers to memory on the heap, so when an object is assigned to another variable, only its pointer is being copied on the stack. The object itself is not copied and remains on the heap. This is desired because making a copy is expensive. Copying an array of bytes or a string every time you need to reference it in another function or scope would make a program extremely slow and use an insane amount of RAM. While there are performance benefits for shared state, it is common in OOP languages to easily create bugs related to shared state, such as race conditions by accessing an object from multiple threads. While race conditions are solved with thread locks, this creates additional overhead, inefficiency, and complexity, as well as the potential for deadlocks. Another common bug that arises from shared state in Object Oriented Programming is unexpected mutations. Making a change to a property of an object will affect other variables that are holding on to a reference to that object. We saw this in chapter 1 with `PersonValueType` vs `PersonReferenceType`. Unexpected mutations are prevented by explicitly making a copy of the data. While this solves the problem, it creates a burden on the programmer to decide when a copy should be made, and it completely nullifies the purpose of shared state: to prevent creating unnecessary copies.

Swift solves this problem with copy-on-write. The compiler will only make a copy of the data of a value type when a mutation is made. It will also use intelligent memory to determine when buffers can be shared. This prevents unnecessary copies, but still guarantees the programmer that each variable is virtually a unique instance.

```
struct Animal {
    var type: String
    var age: Int
}

/// Unsafe structure to obtain a pointer to an instance of `Animal`.
struct Address {
    let pointer: UnsafeMutableRawPointer
    let padding1: Int64
    let padding2: Int64
    let padding3: Int64
}

// structs are very memory efficient
print("Size of Animal struct: \(MemoryLayout<Animal>.size) bytes")
print("Size of Address struct: \(MemoryLayout<Address>.size) bytes")
print("Size of Animal struct's members: \(MemoryLayout<String>.size + MemoryLayout<Int>.size) bytes")

// declare variables
let a = Animal(type: "Dog", age: 2)
var b = a
var c = b

// a, b and c point to the same buffer
print("Animal `a`:", unsafeBitCast(a, to: Address.self).pointer)
print("Animal `b`:", unsafeBitCast(b, to: Address.self).pointer)
print("Animal `c`:", unsafeBitCast(c, to: Address.self).pointer)

// mutate `b`
b.type = "Cat"
print("Mutated b:", b)

// `b` now has a new buffer
print("Animal `a`:", unsafeBitCast(a, to: Address.self).pointer)
print("Animal `b`:", unsafeBitCast(b, to: Address.self).pointer)
print("Animal `c`:", unsafeBitCast(c, to: Address.self).pointer)

// assign same value to `c` as `b`
c.type = "Cat"
print("Mutated c:", c)

// `c` now points to same buffer as `b`, because their values are equal
print("Animal `a`:", unsafeBitCast(a, to: Address.self).pointer)
print("Animal `b`:", unsafeBitCast(b, to: Address.self).pointer)
print("Animal `c`:", unsafeBitCast(c, to: Address.self).pointer)
```

# Memory Management

Swift uses Automatic Reference Counting carried over from Objective-C. While this is much more performant and deterministic than garbage collection, objects still need to take up additional memory for class metadata such as the current reference count.

```
import SwiftShims

// define struct
struct CarStruct {
    var model: String
    var year: Int
    init(model: String, year: Int) {
        self.model = model
        self.year = year
    }
}

// define class
class CarClass {
    var model: String
    var year: Int
    init(model: String, year: Int) {
        self.model = model
        self.year = year
    }
}

let model = "Toyota Corolla"
let year = 1995
print("Object and struct's members require \(MemoryLayout<String>.size + MemoryLayout<Int>.size) bytes")

let carStruct = CarStruct(model: model, year: year)
print("Struct's data on the stack is \(MemoryLayout<CarStruct>.size) bytes")

let carClass = CarClass(model: model, year: year)
let classPointerSize = MemoryLayout<CarClass>.size
let classHeapSize = _swift_stdlib_malloc_size(unsafeBitCast(carClass, to: UnsafePointer<CChar>.self))
print("Object's pointer on the stack is \(classPointerSize) bytes")
print("Object's data on the heap is \(classHeapSize) bytes")
print("Object's total memory usage is \(classPointerSize + classHeapSize) bytes")
```

Objects also take longer to initialize because they consume more memory, and are dynamically allocated on the stack. Not to mention various calls to Swift’s runtime. Automatic Reference Counting also makes objects slightly less performant compared to structs because the compiler needs to insert static calls to `swift_retain` and `swift_release`.

```
// Import standard C library
#if os(macOS) || os(iOS)
    import Darwin.C
#else
    import Glibc
#endif

/// Get seconds since UNIX epoch
func timeOfDay() -> Double {
    var timeStamp = timeval()
    gettimeofday(&timeStamp, nil)
    let secondsSince1970 = Double(timeStamp.tv_sec)
    let microseconds = Double(timeStamp.tv_usec) / 1_000_000.0
    return secondsSince1970 + microseconds
}

protocol Date {
    var secondsSince1970: Double { get set }
    init()
    init(secondsSince1970: Double)
}

struct DateStruct: Date {
    var secondsSince1970: Double
    init() {
        self.secondsSince1970 = timeOfDay()
    }
    init(secondsSince1970: Double) {
        self.secondsSince1970 = secondsSince1970
    }
}

class DateClass: Date {
    var secondsSince1970: Double
    static var deallocationCount = 0
    required init() {
        self.secondsSince1970 = timeOfDay()
    }
    required init(secondsSince1970: Double) {
        self.secondsSince1970 = secondsSince1970
    }
    deinit {
        DateClass.deallocationCount += 1
    }
}

class DateSubclass: DateClass {
    required init() {
        // perform subclass stuff
        let _ = timeOfDay()
        super.init()
    }
    required init(secondsSince1970: Double) {
        // perform subclass stuff
        let _ = timeOfDay()
        super.init(secondsSince1970: secondsSince1970)
    }
}

func test<T: Date>(_ dateType: T.Type) {
    
    let size = 10_000
    let start = timeOfDay()
    for _ in 0 ..< size {
        let _ = dateType.init() // test initializer
    }
    let end = timeOfDay()
    print("\(dateType) init test: \(end - start)")
}

test(DateStruct.self)
test(DateClass.self)
test(DateSubclass.self)

print("swift_release was called \(DateClass.deallocationCount) times")
```

In addition to the performance overhead in function calls when initializing and deallocating objects, reference types cannot take advantage of reusing variables and buffers like value types can. By reusing a variable, you can drastically reduce the amount of memory that needs to be allocated (and later cleaned up) for your data. Since creating a class always allocates memory on the heap, reusing a variable only recycles the bytes needed for the pointer to the data on the heap (a pointer is typically one byte). Since structs are always stored on the stack, the amount of memory than can be consumed is determined at compile time, and the entire buffer allocated for the struct instance can be completely reused.

```
import SwiftShims

struct EmployeeStruct {
    let identifier: Int
    let yearOfBirth: Int
    let name: String
    init(identifier: Int, yearOfBirth: Int, name: String) {
        self.identifier = identifier
        self.yearOfBirth = yearOfBirth
        self.name = name
    }
}

class EmployeeClass {
    let identifier: Int
    let yearOfBirth: Int
    let name: String
    init(identifier: Int, yearOfBirth: Int, name: String) {
        self.identifier = identifier
        self.yearOfBirth = yearOfBirth
        self.name = name
    }
}

let creationCount = 1000

do {
    
    var classLoopMemoryUsage = 0
    
    for _ in 0 ..< creationCount {
        
        let variable = EmployeeClass(identifier: 1, yearOfBirth: 1995, name: "John Miller")
        
        let classPointerSize = MemoryLayout<EmployeeClass>.size
        let classHeapSize = _swift_stdlib_malloc_size(unsafeBitCast(variable, to: UnsafePointer<CChar>.self))
        classLoopMemoryUsage += classPointerSize + classHeapSize
    }
    
    print("Used \(classLoopMemoryUsage) bytes without reusing variables for class")
}

do {
    
    var structLoopMemoryUsage = 0
    
    for _ in 0 ..< creationCount {
        
        let variable = EmployeeStruct(identifier: 1, yearOfBirth: 1995, name: "John Miller")
        
        structLoopMemoryUsage += MemoryLayout<EmployeeStruct>.size(ofValue: variable)
    }
    
    print("Used \(structLoopMemoryUsage) bytes without reusing variables for struct")
}

do {
    
    var classLoopMemoryUsage = 0
    
    var variable: EmployeeClass!
    
    for _ in 0 ..< creationCount {
        
        variable = EmployeeClass(identifier: 1, yearOfBirth: 1995, name: "John Miller")
        
        let classPointerSize = MemoryLayout<EmployeeClass>.size
        let classHeapSize = _swift_stdlib_malloc_size(unsafeBitCast(variable, to: UnsafePointer<CChar>.self))
        classLoopMemoryUsage += classHeapSize
    }

    let classPointerSize = MemoryLayout<EmployeeClass>.size
    classLoopMemoryUsage += classPointerSize
    
    print("Used \(classLoopMemoryUsage) bytes reusing variable for class")
}

do {
    
    let structLoopMemoryUsage = MemoryLayout<EmployeeStruct>.size
    
    var variable: EmployeeStruct!
    
    for _ in 0 ..< creationCount {
        
        variable = EmployeeStruct(identifier: 1, yearOfBirth: 1995, name: "John Miller")
    }
    
    print("Used \(structLoopMemoryUsage) bytes reusing variable for struct")
}
```

# Inheritance

Inheritance is achieved through protocol extensions in POP. With classes, you are forced to choose one superclass, cannot retroactively model, and inherit more memory usage from the superclass’ stored properties. Initialization chaining (e.g. calling `super.init()`) is also an additional burden on the developer, and adds to the initialization time. Protocol extensions allow types to inherit behavior from multiple protocols, and including method implementations. Since there is no class hierarchy, calling methods or computed properties in protocol extensions are as fast as calling a C function.

```
protocol Person {
    var identifier: Int { get }
    var firstName: String { get }
    var lastName: String { get }
    var fullName: String { get }
}

extension Person {
    var fullName: String {
        return firstName + " " + lastName
    }
}

struct Attendee: Person {
    var identifier: Int
    var firstName: String
    var lastName: String
}

struct Speaker: Person {
    var identifier: Int
    var firstName: String
    var lastName: String
    var event: String
}

let people: [Person] = [Speaker(identifier: 1, firstName: "John", lastName: "Miller", event: "How to use Swift on the Server"),
              Attendee(identifier: 2, firstName: "Mark", lastName: "Alvarado"),
              Attendee(identifier: 3, firstName: "James", lastName: "Smith")]

print("List of people in this meeting:")
people.forEach { print($0.fullName) }
```

All types in Swift can inherit methods and computed properties by just declaring conformance to a protocol. Methods declared in protocol extensions provide a default implementation, and the ability to override like classes do. Protocol extensions can also be static and non-overridable by providing methods in an extension that do not belong to the protocol.

```
/// Protocol Oriented Programming View
protocol View {
    /// Will always need to be implemented.
    var frame: (x: Double, y: Double, width: Double, height: Double) { get }
    /// Protocol extension provides default implementation, but can be overriden.
    var userInteractionEnabled: Bool { get }
}

extension View {
    /// Default implementation for protocol requirement.
    var userInteractionEnabled: Bool {
        return true
    }
    /// Protocol extension that cannot be overriden.
    func handleTouch() {
        if userInteractionEnabled {
            print("\(self) was touched")
        }
    }
}

struct Button: View {
    // `frame` must always be provided
    var frame: (x: Double, y: Double, width: Double, height: Double)
    // `handleTouch()` and `userInteractionEnabled` are already implemented.
}

struct Label: View {
    // `frame` must always be provided
    var frame: (x: Double, y: Double, width: Double, height: Double)
    // `handleTouch()` is already implemented and cannot be overriden,
    // but the default implementation of `userInteractionEnabled` can be overriden
    var userInteractionEnabled: Bool {
        return false
    }
}

// Protocol Oriented Programming is great for unit testing
let button = Button(frame: (0, 0, 10, 10))
let label = Label(frame: (0, 10, 20, 20))

button.handleTouch()
label.handleTouch() // will not print anything
```

Abstract classes are a useful tool in a developer’s toolbox, but can sometimes lead to bugs. Mainly, not overriding a required method, or overriding a method that should not be overridden are a common issues. The concept of abstract classes translates very nicely into Protocol Oriented Programming and is much safer than in Object Oriented Programming.

```
class AbstractClass {
    /// an internal value that should always be true
    var consistencyCheck: Bool
    func mustOverride() {
        fatalError("Need to override")
    }
    init() {
        consistencyCheck = true
    }
}

class ConcreteClass: AbstractClass {
    override func mustOverride() {
        // super.mustOverride() // calling this will compile, but crash at runtime
        print("Overriden abstract class method")
    }
    override init() {
        super.init()
        consistencyCheck = false // developer did not read documentation, or value is undocumented
    }
}

protocol AbstractProtocol {
    // since no protocol extension provides an implementation, the compiler will force us to implement the method
    func mustImplement()
}

struct ImplementedProtocol: AbstractProtocol {
    func mustImplement() {
        // no super to accidentally call and crash application
        print("Safe POP implementation")
    }
}

let concreteClass = ConcreteClass()
concreteClass.mustOverride()
if concreteClass.consistencyCheck == false {
    print("Warning: an internal consistency check failed for \(concreteClass)")
}

// let abstractClass = AbstractClass()
// abstractClass.mustOverride() // will crash

let implementedProtocol = ImplementedProtocol()
implementedProtocol.mustImplement()

// let abstractProtocol = AbstractProtocol() // will not compile, which is safer for the developer
```

Classes can also lose type relationships when a subclass tries to override a method. This is something protocols prevent.

```
class ComparableClass {
    func compare(_ other: ComparableClass) -> Bool {
        fatalError("Must implement \(#function)")
    }
}

class NumberClass: ComparableClass {
    var value: Int = 0
    override func compare(_ other: ComparableClass) -> Bool {
        return self.value < (other as! NumberClass).value
    }
}

class TextClass: ComparableClass {
    var text: String = ""
    override func compare(_ other: ComparableClass) -> Bool {
        return self.text.utf8.count < (other as! TextClass).text.utf8.count
    }
}

let numberClass1 = NumberClass()
numberClass1.value = 10
let numberClass2 = NumberClass()
numberClass2.value = 20
numberClass1.compare(numberClass2)

let textClass1 = TextClass()
textClass1.text = "Hello World"
let textClass2 = TextClass()
textClass2.text = "Classes rule"
textClass1.compare(textClass2)

// Will compile, but will crash
// error: Could not cast value of type 'NumberClass' to 'TextClass'.
// textClass1.compare(numberClass1)

protocol ComparableProtocol {
    func compare(_ other: Self) -> Bool
}

struct NumberStruct {
    var value: Int
    func compare(_ other: NumberStruct) -> Bool {
        return self.value < other.value
    }
}

struct TextStruct {
    var text: String
    func compare(_ other: TextStruct) -> Bool {
        return self.text.utf8.count < other.text.utf8.count
    }
}

let numberStruct1 = NumberStruct(value: 10)
let numberStruct2 = NumberStruct(value: 20)
numberStruct1.compare(numberStruct2)

let textStruct1 = TextStruct(text: "Hello World")
let textStruct2 = TextStruct(text: "Protocols rule")
textStruct1.compare(textStruct2)

// Will not compile thanks to Swift's type safety
// error: cannot convert value of type 'TextStruct' to expected argument type 'NumberStruct'
// numberStruct1.compare(textStruct1)
```

# Values types with shared state

`String`, `Array`, and `Dictionary` are essential building blocks for writing software in Swift. Like most of the standard library, they are structures, and not classes. At first glance this should be impossible because a structure’s size must be known at compile-time, and these types can hold a variable length of data. Defining these types as value types guarantees a virtual copy every time they are assigned to a new constant or variable, or when they are passed to a function or method. But in actuality, making a copy every time would have a huge impact on memory footprint and performance. Because of this, these types are defined as reference types, with value semantics. They are able to take full advantage of the copy-on-write behavior of value types, as well as the shared state behavior of reference types (classes).

```
public typealias Byte = UInt8

// Mixing reference and value types
public struct Image {
    
    internal var storage = Storage()
    
    private mutating func ensureUnique() {
        if !isKnownUniquelyReferenced(&storage) {
            storage = storage.copy
        }
    }
    
    public var count: Int {
        return self.storage.count
    }
    
    public subscript (index: Int) -> Byte {
        return storage.pointer![index]
    }
    
    public mutating func append(_ byte: Byte) {
        ensureUnique()
        storage.append(byte)
    }
}

extension Image {
    
    class Storage {
        var count: Int
        var pointer: UnsafeMutablePointer<Byte>?
        /// Initializes with a copy of the provided data.
        init(count: Int = 0, pointer: UnsafeMutablePointer<Byte>? = nil) {
            self.count = count
            guard let pointer = pointer else {
                self.pointer = nil
                return
            }
            self.pointer = UnsafeMutablePointer<Byte>.allocate(capacity: count)
            self.pointer?.initialize(from: pointer, count: count)
            print("Created new buffer \(self.pointer!)")
        }
        deinit {
            pointer?.deinitialize(count: count)
            pointer?.deallocate(capacity: count)
            print("Buffer \(pointer!) was deallocated")
        }
        var copy: Storage {
            print("Buffer did change")
            return Storage(count: count, pointer: pointer)
        }
        func append(_ byte: Byte) {
            if let pointer = self.pointer {
                (pointer + count).initialize(to: byte)
            } else {
                pointer = UnsafeMutablePointer<Byte>.allocate(capacity: 1)
                print("Created new buffer \(self.pointer!)")
            }
            
            count += 1
        }
    }
}

func testImage() {
    
    /// create psuedo image data
    var image = Image()
    print("Inserting first byte: 0")
    image.append(0x00) // first byte, buffer is internally created
    var originalBuffer = image.storage.pointer
    print("Appending remaining bytes")
    for index in 0 ..< 99 {
        image.append(0x01)
        print("Appended byte \(index + 1):", image[index + 1])
    }
    
    print("Total bytes: \(image.count)")
    
    print("Buffer did change:", originalBuffer != image.storage.pointer)
    
    // Assign value type to another variable. Will not make a copy of the buffer
    var image2 = image
    print("Shared buffer:", image.storage.pointer == image2.storage.pointer)
    
    // Mutating either variable will copy the buffer, only if there is more than one reference to it.
    // In this case there are 2: `image` and `image2` both hold a strong reference to the buffer.
    print("Will mutate original image")
    image.append(0x02)
    
    print("Shared buffer after mutation:", image.storage.pointer == image2.storage.pointer)
}

print("Will run test")
testImage()
print("Did run test")
```