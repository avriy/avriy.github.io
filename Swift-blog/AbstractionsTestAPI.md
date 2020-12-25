# Make common abstractions testable

During development of a project you might find some abstractions so usefull, you start reusing them throughout the codebase. In this article we are going to learn a trick how to make tests API supporting these abstractions.

Imagine you have a storage abstraction like this:
```swift
/// Abstract value storage of `Codable` types.
class ValueStorage<Value: Codable> {
    func save(_ value: Value, completion: @escaping (Error?) -> Void) {
        // ...
    }
}
```

`ValueStorage` is used to store downloaded data in `URLDownloader`:
```swift
class URLDownloader<Value: Codable> {
    private let valueStorage: ValueStorage<Value>
    init(valueStorage: ValueStorage<Value>) {
        self.valueStorage = valueStorage
    }

    /// Downloads value for `url` and saves it.
    func download(at url: URL, completion: @escaping (Error?) -> Void) {
        // ...
        // Delegate saving downloaded value to `valueStorage`
        valueStorage.save(value, completion: completion)
    }
}
```
`URLDownloader` is what we'd like to test. You may start with the test like:
```swift
class URLDownloaderTest: XCTestCase {
    let valueStorage = ValueStorage<String>()
    lazy var urlDownloader = URLDownloader(valueStorage: valueStorage)

    func testDownloading() {
        let valueDownloaded = expectation(description: "Value downloaded")
        urlDownloader.download(at: url) { error in
            // Finished downloaded.
            if error == nil {
                valueDownloaded.fulfill()
            }
        }
        wait(for: [valueDownloaded], timeout: 1)
    }
}
```
But how do you know the value was saved? Maybe `URLDownloader` called completion before delegating work to `valueStorage` or it got the wrong value.

## Fake test API

We can introduce a new type of `ValueStorage` designed for tests with a set of expectation utilities.

```swift
class FakeValueStorage<Value: Codable>: ValueStorage {
    private var didSaveValueHandler: (Value) -> Void = { _ in }

    func valueSavedExpectation(matching predicate: @escaping (Value) -> Void) -> XCTestExpectation {
        let expectation = XCTestExpectation(description: "Value is saved matching")
        didSaveValueHandler = { [previousHandler = didSaveValueHander, weak expectation] value in
            // Defering callback to a previous version of the handler makes the chaining work not 
            // matter what happens later.
            defer {
                previousHandler(value)
            }
            if predicate(value) {
                // Expectation is not retained in the closure as usual test doesn't need it as you create bunch of expectations at the beginning of the test and wait on them at the end.
                expectation?.fulfill()
            }
        }
        return expectation
    }

    override func save(_ value: Value, completion: @escaping (Error?) -> Void) {
        super.save(value) { [weak self] error in
            completion(error)
            if error == nil {
                self?.didSaveValueHandler(value)
            }
        }
    }
}
```

## Improved test
```swift
class URLDownloaderTest: XCTestCase {
    let valueStorage = FakeValueStorage<String>()
    lazy var urlDownloader = URLDownloader(valueStorage: valueStorage)

    func testDownloading() {
        let downloadedValueSaved = valueStorage.valueSavedExpectation { 
            $0.contains("Lorem Ipsum")
        }
        urlDownloader.download(at: url) { error in
            XCTAssertNil(error)
        }
        wait(for: [downloadedValueSaved], timeout: 1)
    }
}
```
Which remains simple and has much stronger assertion.

## FAQ

*Should I be writing such a big chunk of code for each class?*

No, only for the abscructions used havily inside your code.

*Why using matching instead of Equatability*

Equatability check is a special case of for a matcher, 