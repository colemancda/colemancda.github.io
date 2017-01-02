---
layout: post
title:  "Swift on Linux"
date:   2017-01-02
---

# LLVM and Clang

Although Swift was announced in WWDC 2014, Swift wasn’t released as open source until December 2015. Swift is built on the LLVM compiler, and is completely interoperable with C and Objective-C. The LLVM compiler is a very unique compiler and can support new languages without little to no modifications. This is a stark contrast to other compilers like GCC that must be forked to support new platforms and features. LLVM itself does not compile Swift nor C, it only accepts LLVM IR (Low Level Virtual Machine Intermediate Representation), an intermediary byte code, hence its name. Despite the use of virtual machine in its name, and only accepting byte code, LLVM uses Ahead of Time Compilation instead of Just in Time Compilation like the Java Virtual Machine (Java) and Common Language Infrastructure (C#) use. This results in LLVM supporting multiple languages and optimizations, but still being able to export machine assembly code like traditional compilers. In order to generate the intermediary byte code LLVM needs, a compiler front end is needed to compile a specific programming language into LLVM IR. 

Because Swift is interoperable with C, it requires Clang, the LLVM front end for the C family of languages. Clang, coupled with LLVM as its back end, is a full stack replacement for GCC. Clang's command-line interface is similar to and shares many flags and options with GCC. Clang is the default compiler for the PS4 SDK, and has recently replaced GCC for the Android SDK. For server-side purposes, a Debian-based Linux operating system is recommended, namely Ubuntu 14.04 LTS. You can install the dependencies for the Swift compiler with the following command:

```
sudo apt-get install clang libicu-dev git
```

Clang is required for the Swift compiler to interact with C and Objective-C, and will automatically install LLVM as a dependency. The [International Components for Unicode](http://site.icu-project.org) library is required for the Swift Standard Library to properly perform operations with unicode text in its `String` type. Finally, `git` is required for the Swift Package Manager (which we will discuss in detail later in this chapter) to fetch dependencies.

# Swift Releases

You can fetch the official releases of Swift at `swift.org`. In this book all of our code uses the Swift 3.0 release. The official releases of Swift for Ubuntu contain the Swift Compiler (an LLVM front end) and Standard Library as well as other base libraries familiar to iOS developers: Foundation, XCTest and Grand Central Dispatch. Swift also has its own dependency manager called the Swift Package Manager, which relies on `git` to fetch dependencies. You can download a copy of the Swift 3.0 compiler suite for Ubuntu 14.04 LTS with the following command:

```
wget https://swift.org/builds/swift-3.0-release/ubuntu1404/swift-3.0-RELEASE/swift-3.0-RELEASE-ubuntu14.04.tar.gz
```

Once you have the release, you must decompress it.

```
tar -zxvf swift-3.0-RELEASE-ubuntu14.04.tar.gz
```

Finally, you must add the compiler’s executables path to the current command line session path in order to use it call them by name.

```
export PATH=/path/to/swift-3.0-RELEASE-ubuntu14.04/usr/bin/:"${PATH}"
```

Note that this final step needs to be repeated every time want to use Swift in a new command line session, no matter whether you are using the `Terminal` app on Ubuntu, or remotely connecting to a server with SSH. Verify installation of Swift with the following command:

```
swift —version
```

Which should print the following:

```
Swift version 3.0.0 (swift-3.0.0-RELEASE)
Target: x86_64-unknown-linux-gnu
```

# Swift Package Manager

Now that you are all setup to using Swift on Linux, lets build an executable. We will use the Swift Package Manager to define a package. A package consists of multiple modules. A module in Swift is both a target and a namespace, and can be an executable, or a library. To build an executable just include `main.swift` in the module’s folder, and for libraries do not include such file. Let’s build a `helloworld` executable using the Swift Package Manager by creating `main.swift` in the following directory layout:

```
helloworld/sources/main.swift
```

And set `main.swift` to the following:

```
print("Hello, Swift!")
```

Now build the package with `swift build`. Make sure to execute this command while in the `helloworld` directory and not in `sources`. Now execute the built binary with `.build/debug/helloworld`, you should see the following:

```
Hello, Swift!
```

Now lets build a bit more complex setup. We will build a package that builds both an executable, and a library that the executable will link against. In addition, that library will consume a third partly library from Github, and include its set of unit tests. Create a folder called `SecureData` which will be the root folder for our Swift package. Declare the package’s contents by creating a `Package.swift` file.

```
// SecureData/Package.swift

import PackageDescription

let package = Package(
    name: "SecureData",
    targets: [
        Target(
            name: "noncegen",
            dependencies: [.Target(name: "SecureData")]
        ),
        Target(
            name: "SecureData"
        )
    ],
    dependencies: [
        .Package(url: "https://github.com/krzyzanowskim/CryptoSwift", majorVersion: 0)
    ]
)
```

In this file we are declaring a package called `SecureData`. It requires the [CryptoSwift](https://github.com/krzyzanowskim/CryptoSwift) library at `0.x.x`. This package will build a `SecureData` target (which will be a library) and a `noncegen` target (an executable). The Swift Package Manager will first fetch and compile the external dependencies, then compile `SecureData`, as it is a requirement for `noncegen`. Next lets define our `SecureData` library. We will need to create the following folders:

```
SecureData/Sources
SecureData/Sources/SecureData
```

The subfolders in `Sources` must match the names of the targets we declared in our `Package.swift` definition. Next lets create a `SecureData.swift` file that will define the main protocol we use in out library.

```
// SecureData/Sources/SecureData/SecureData.swift

import Foundation
import CryptoSwift

/// Secure Data Protocol. 
public protocol SecureData: Equatable, CustomStringConvertible {
    
    /// The data length. 
    static var length: Int { get }
    
    /// The data.
    var data: Data { get }
    
    /// Initialize with data.
    init?(data: Data)
    
    /// Initialize with random value.
    init()
}

public extension SecureData {
    
    /// Generate random data with the specified size.
    public static func random() -> Data {
        
        let bytes = CryptoSwift.AES.randomIV(Self.length)
        
        return Data(bytes: bytes)
    }
}

// MARK: - CustomStringConvertible

extension SecureData {
    
    public var description: String {
        
        return data.toHexString()
    }
}

// MARK: - Equatable

public func == <T: SecureData> (lhs: T, rhs: T) -> Bool {
    
    return lhs.data == rhs.data
}
```

Our `SecureData` protocol defines its requirements, as well as shared implementations of `==`, `Self.random()` and `self.description`. Lets define a `Nonce` type that conforms to `SecureData`.

```
// SecureData/Sources/SecureData/Nonce.swift

import Foundation

/// Cryptographic nonce
public struct Nonce: SecureData {
    
    public static let length = 16
    
    public let data: Data
    
    public init?(data: Data) {
        
        guard data.count == type(of: self).length
            else { return nil }
        
        self.data = data
    }
    
    public init() {
        
        self.data = Nonce.random()
    }
}
```

`Nonce` will contain 16 bytes of randomly generated data, and does not have to worry about how to generate that data, the `SecureData` protocol extension takes care of that. Also note that `Nonce` conforms to `Equatable` and `CustomStringConvertible` without providing its own implementation, an example of inheritance without classes with Protocol Oriented Programming. Lets build unit tests to make sure our code is working correctly. Create the following folders:

```
SecureData/Tests
SecureData/Tests/SecureDataTests
```

And add a `LinuxMain.swift` file in the `Tests` folder. This file will define the unit test suites that will run. Because `XCTest` is an Objective-C framework on OS X, it uses dynamic reflection to deduce your unit test suites and does not need this file.

```
// SecureData/Tests/LinuxMain.swift

import XCTest
@testable import SecureDataTests

XCTMain([testCase(NonceTests.allTests)])
```

Next lets implement a unit test suite with various tests. Add `NonceTests.swift` in the `SecureDataTests` subfolder of `Tests`. A package can define many test modules, in our case we only define one test module (`SecureDataTests`), with one test case (`NonceTests`).

```
// SecureData/Tests/SecureDataTests/NonceTests.swift

import XCTest
import Foundation
@testable import SecureData

final class NonceTests: XCTestCase {
    
    static var allTests = [
        ("testLength", testLength),
        ("testInitWithData", testInitWithData),
        ("testEquatable", testEquatable)
        ]
    
    func testLength() {
        
        let nonce = Nonce()
        
        XCTAssert(nonce.data.isEmpty == false, "Generated nonce data must not be empty")
        XCTAssert(nonce.data.count == Nonce.length, "Generated nonce data must equal the size of nonce data")
    }
    
    func testInitWithData() {
        
        XCTAssert(Nonce(data: Data()) == nil, "Cannot initialize nonce with empty data")
        XCTAssert(Nonce(data: Data(bytes: [0x01, 0x02, 0x03])) == nil, "Cannot initialize nonce with invalid length of data")
    }
    
    func testEquatable() {
        
        XCTAssert(Nonce() != Nonce(), "Two randomly generated nonces cannot be the same")
        
        let nonce = Nonce()
        let nonceCopy = Nonce(data: nonce.data)
        
        XCTAssert(nonce == nonceCopy, "Nonces with same data must be equal")
    }
}
```

Before running the unit test, lets implement a simple command line tool to use our library from the command line. Create a `main.swift` file in the `noncegen` folder, so the Swift package manager will compile that target as an executable and not a library.

```
// SecureData/Sources/noncegen/main.swift

import Foundation
import SecureData

let nonce = Nonce()

print(nonce)
```

Now lets run our unit tests to make sure our code is working properly.

```
swift test
```

You should output similar to the following:
```
Test Suite 'All tests' started at 03:46:54.729
Test Suite 'debug.xctest' started at 03:46:54.731
Test Suite 'NonceTests' started at 03:46:54.731
Test Case 'NonceTests.testLength' started at 03:46:54.731
Test Case 'NonceTests.testLength' passed (0.0 seconds).
Test Case 'NonceTests.testInitWithData' started at 03:46:54.732
Test Case 'NonceTests.testInitWithData' passed (0.0 seconds).
Test Case 'NonceTests.testEquatable' started at 03:46:54.732
Test Case 'NonceTests.testEquatable' passed (0.0 seconds).
Test Suite 'NonceTests' passed at 03:46:54.732
	 Executed 3 tests, with 0 failures (0 unexpected) in 0.0 (0.0) seconds
Test Suite 'debug.xctest' passed at 03:46:54.732
	 Executed 3 tests, with 0 failures (0 unexpected) in 0.0 (0.0) seconds
Test Suite 'All tests' passed at 03:46:54.732
	 Executed 3 tests, with 0 failures (0 unexpected) in 0.0 (0.0) seconds
```

Lets clean our current build artifacts and build our command line tool. You should see some randomly generated hexadecimal in your console.

```
swift build ——clean
swift build
.build/debug/noncegen
```

You can also build and run an optimized version of your code.

```
swift build ——configuration release
.build/release/noncegen
```

On OS X you can generate an Xcode project file with `swift package generate-xcodeproj`. This allows you to use Apple’s IDE for debugging and development, and not worry about manually managing which files and targets you want to compile.

# Platform differences

When developing cross-platform Swift code, be aware of the differences between Apple’s platforms (macOS, iOS, watchOS, tvOS) and Linux. For starters, the C standard library on Linux is `Glibc`, while on Apple platforms its `Darwin.C` module.

```
// Import standard C library
#if os(macOS) || os(iOS)
    import Darwin.C
#else
    import Glibc
#endif

let string = "TestString"
let cStringLength = strlen(string)

print("String length is", cStringLength)
```

Some functions from the C standard library on Darwin (the kernel used in Apple’s platforms) are not available on Linux. `arc4random_uniform()` being an example. `SwiftShims` module provides portable C functions for extended POSIX functionality.

```
 // Import standard C library
#if os(macOS) || os(iOS)
    import Darwin.C
#else
    import SwiftShims
    import Glibc
#endif

let upperBound: UInt32 = 10

let randomValue: UInt32

#if os(Linux)
randomValue = _swift_stdlib_cxx11_mt19937_uniform(upperBound)
#else
randomValue = arc4random_uniform(upperBound)
#endif

print("Random number:", randomValue)
```

Likewise, there are some C functions like `system()` that can only run on Linux.

```
#if os(Linux)
import Glibc
system("uname")
#else
print("Cannot access the command line")
#endif
```

# Foundation

The Foundation library is the base library that comes bundled with builds of Swift to complement the Standard Library. Foundation includes basic types such as `Date`, `Data`, `UUID`, `URL` `RegularExpression`, and `JSONSerialization`, which are needed for most applications but don’t belong the in the Standard Library, thus they are not a part of the language per-se. 

Here is a simple example of how you would represent requests to a server using Foundation in Swift.

```
import Foundation

enum Environment: String {
    
    case Staging = "https://staging.myserver.com/api"
    case Production = "https://myserver.com/api"
}

protocol Request: CustomStringConvertible {
    
    var environment: Environment { get set }
    
    func toRequest() -> URLRequest
}

extension Request {
    
    var description: String {
        
        let request = self.toRequest()
        
        let method = request.httpMethod!
        
        let url = request.url!.absoluteString
        
        var description = method + " " + url
        
        if let body = request.httpBody,
            let string = String(data: body, encoding: .utf8) {
            
            description += " " + string
        }
        
        return description
    }
}

/// Example GET request. 
struct GetUserProfileRequest: Request {
    
    var environment: Environment
    var username: String
    
    func toRequest() -> URLRequest {
        
        let url = URL(string: environment.rawValue + "/user/" + username)!
        
        var request = URLRequest(url: url)
        
        request.httpMethod = "GET"
        
        return request
    }
}

/// Example PUT request. 
struct UpdateUserProfile: Request {
    
    var environment: Environment
    var username: String
    var firstName: String
    var lastName: String
    var email: String
    
    enum JSONKey: String {
        
        case firstName, lastName, email
    }
    
    func toRequest() -> URLRequest {
        
        let url = URL(string: environment.rawValue + "/user/" + username)!
        
        var request = URLRequest(url: url)
        
        request.httpMethod = "PUT"
        
        // JSON
        var jsonBody = [String: String]()
        jsonBody[JSONKey.firstName.rawValue] = firstName
        jsonBody[JSONKey.lastName.rawValue] = lastName
        jsonBody[JSONKey.email.rawValue] = email
        
        request.httpBody = try! JSONSerialization.data(withJSONObject: jsonBody, options: .prettyPrinted)
        
        return request
    }
}

let environment: Environment = .Staging
let username = "johnapple1995"

let getRequest = GetUserProfileRequest(environment: environment, username: username)
print(getRequest)

let putRequest = UpdateUserProfile(environment: environment, username: username, firstName: "John", lastName: "Apple", email: "johnapple1995@icloud.com")
print(putRequest)
```

# Grand Central Dispatch

Optimizing your code for concurrency and multithreaded environments is a common practice in modern application programming, from server-side to mobile software. In modern POSIX operating systems, threads  allow concurrent processing within a process, which is absolutely vital for highly performant and scalable applications. The efficiency of creating multiple threads for concurrent operations will vary based on the amount of CPUs on the computer. For example, an image processing application optimized and coded to use two threads, will be highly performant on a dual-core CPU. On the other hand, the performance will decrease on a CPU with a single CPU, and on a CPU with 3 or 4 cores, it will not be taking advantage of the other cores on the CPU. Because of this issue, Apple created a library called `libdispatch` or Grand Central Dispatch. It abstracts away the concept of threads and deals with queues. A `DispatchQueue` represents a queue of tasks, and depending on the CPU resources available (e.g. number of cores), schedules the queue’s pending tasks for execution. A queue can be serial or concurrent. The advantage of working with GCD vs POSIX threads, is twofold: a simpler programming and concurrency model, and better performance. Grand Central Dispatch implements algorithms to decide how many threads should be created for a concurrent queue, so the programmer does not need to know how many threads to create or how to optimize a program for a certain number of CPUs.

```
import Foundation
import Dispatch

// get number of CPU cores for informational purposes
print("Running on \(sysconf(Int32(_SC_NPROCESSORS_ONLN))) CPU cores")

enum Response {
    
    case error(Error)
    case data(Data)
}

let start = Date()
var responses = [Response]()
let numberOfOperations = 100

let serialQueue = DispatchQueue(label: "Serial Queue")
let concurrentQueue = DispatchQueue(label: "Concurrent Queue", attributes: .concurrent)

let url = URL(string: "http://httpbin.org/image/png")!
let urlSession = URLSession(configuration: URLSessionConfiguration.default)

for _ in 0 ..< numberOfOperations {
    
    // perform work in background
    concurrentQueue.async {
        
        let task = urlSession.dataTask(with: url, completionHandler: { (data, urlResponse, error) in
            
            let response: Response
            
            if let error = error {
                
                response = .error(error)
                
            } else {
                
                response = .data(data!)
            }
            
            // for thread safety, we need to modify variables on a single thread / serial queue
            serialQueue.async {
                
                responses.append(response)
                
                print("Processed \(responses.count) operations")
                
                // check if last response
                guard responses.count < numberOfOperations else {
                    
                    let now = Date()
                    print("Operations took \(now.timeIntervalSince(start)) seconds")
                    exit(0)
                }
            }
        })
        task.resume()
    }
}

while true { sleep(10) } // sleep main thread forever
```



