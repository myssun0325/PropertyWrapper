# Property Wrapper

## 사용조건

전역변수나 computed property에는 적용불가

Swift 5.1 이상에서 도입

```swift
// example
@propertyWrapper
struct TwelveOrLess {
	private var number = 0
	var wrappedValue: Int {
		get { number }
		set { number = min(newValue, 12) }
	}
}

struct SmallRectangle {
    @TwelveOrLess var height: Int
    @TwelveOrLess var width: Int
}

var rectangle = SmallRectangle()
print(rectangle.height)
// Prints "0"

rectangle.height = 10
print(rectangle.height)
// Prints "10"

rectangle.height = 24
print(rectangle.height)
// Prints "12"
```

## Synthesizes

프로퍼티에 wrapper를 적용하면, 컴파일러가 아래 2가지 코드를 자동으로 생성해준다.

1. wrapper에 대한 저장소를 제공하는 코드
2. wrapper를 통해 프로퍼티에 접근할 수 있는 코드

(wrapper는 `wrapped value`를 저장해야하는 책임이 있기 때문에, 그것에 대한 코드 생성은 없음)

When you apply a wrapper to a property, the compiler synthesizes code that provides storage for the wrapper and code that provides access to the property through the wrapper. (The property wrapper is responsible for storing the wrapped value, so there’s no synthesized code for that.) You could write code that uses the behavior of a property wrapper, without taking advantage of the special attribute syntax. 

- 다른 블로그 글
    
    [https://sarunw.com/posts/what-is-property-wrappers-in-swift/](https://sarunw.com/posts/what-is-property-wrappers-in-swift/)
    
    프로퍼티에 property wrapper를 달게되면 
    
    1. It **makes that property computed** (generate a getter/setter).
    2. It **introduces a stored property** whose **type is the wrapper type**.
    3. Initialize the stored property in one of **[three ways](https://sarunw.com/posts/what-is-property-wrappers-in-swift/#how-to-initialize-property-wrapper)**.

코드로 보기

```swift
@propertyWrapper // 1
struct Truncate {
    private var _value: String = ""
    
    var wrappedValue: String { // 2
        set {
            _value = truncate(string: newValue)
        }
        
        get {
            return _value
        }
    }
    
    private func truncate(string: String) -> String {
        if string.count > 10 {
            return string.prefix(10) + "..."
        } else {
            return string
        }
    }
}

struct BlogTeaser {
	@Truncate var body: String
}

// compiler generated like
struct BlogTeaser {
	priavte var _body = Truncate() // wrapper type의 init()
	var body: String { // wrappedValue에 접근할 수 있는 computed property를 생성해줌
		get { return _body.wrappedValue }
		set { _body.wrappedValue = newValue }
	}
}

struct BlogTeaser {
	@Truncate var body: String
	func foo() {
		print(_body) // You can access _body of type Truncate
	}
}

```

## ⭐⭐ 초기값 (Setting Initial Values for Wrapped Properties)⭐⭐

TwelveOrLess의 정의에서 number에 초기값 셋팅 (wrappedValue는 이 초기값을 씀)

문제점: 다른 초기값을 쓸 수 없다.

```swift

// TwelveOrLess의 정의에서 number에 정의된 값으로 세팅
// 대신이렇게되면 다른 초기값을 지정할 수 없다.
@propertyWrapper
struct TwelveOrLess {
    private var number = 0
    var wrappedValue: Int {
        get { number }
        set { number = min(newValue, 12) }
    }
}

struct SmallRectangle {
    @TwelveOrLess var height: Int
    @TwelveOrLess var width: Int
		// 이런거 안됨.
		//@TwelveOrLess var width: Int = 5
}

var rectangle = SmallRectangle() // height, width에 다른 값을 줄 수 없음
```

→ 초기값 커스터마이징을 위해 이니셜라이저를 추가해주기

```swift
@propertyWrapper
struct SmallNumber {
    private var maximum: Int
    private var number: Int
    
    var wrappedValue: Int {
        get { number }
        set { number = min(newValue, maximum) }
    }
    
    init() {
        maximum = 12
        number = 0
    }
    
    init(wrappedValue: Int) {
        maximum = 12
        number = min(wrappedValue, maximum)
    }
    
    init(wrappedValue: Int, maximum: Int) {
        self.maximum = maximum
        number = min(wrappedValue, maximum)
    }
}
```

- `init()` 사용
    - SmallNumber의 인스턴스는 SmallNumber()를 통해 인스턴스화 됐음.
    
    ```swift
    struct ZeroRectangle {
        @SmallNumber var height: Int
        @SmallNumber var width: Int
    }
    
    var zeroRectangle = ZeroRectangle()
    print(zeroRectangle.height, zeroRectangle.width)
    // 0, 0
    ```
    
- `init(wrappedValue:)` 사용 ⭐⭐⭐
    - 프로퍼티래퍼의 프로퍼티에 직접 default value를 주게 되면 init(wrappedValue:) 이니셜라이즈를 호출하게 된다.
    
    ```swift
    struct UnitRectangle {
        @SmallNumber var height: Int = 1 // ⭐️⭐️⭐️
        @SmallNumber var width: Int = 1 // ⭐️⭐️⭐️
    }
    
    var unitRectangle = UnitRectangle()
    print(unitRectangle.height, unitRectangle.width)
    // 1, 1
    
    // 이런것도 됨
    
    struct UnitRectangle {
    	  @SmallNumber(wrappedValue: 1) var height: Int
        @SmallNumber(wrappedValue: 1) var width: Int
    }
    ```
    
- `init(wrappedValue:maximum:)` 사용 → 가장 일반적인 방법
    
    ```swift
    struct NarrowRectangle {
        @SmallNumber(wrappedValue: 2, maximum: 5) var height: Int
        @SmallNumber(wrappedValue: 3, maximum: 4) var width: Int
    }
    var narrowRectangle = NarrowRectangle()
    print(narrowRectangle.height, narrowRectangle.width)
    // 2, 3
    narrowRectangle.height = 100
    narrowRectangle.width = 100
    print(narrowRectangle.height, narrowRectangle.width)
    // 5, 4
    ```
    
- 섞는 것도 됨.
    
    ```swift
    
    struct MixedRectangle {
    		// wrappedValue는 initialValue로 init(wrappedValue: 1)를 호출한다.
        @SmallNumber var height: Int = 1
    		// SmallNumber(wrappedValue: 2, maximum: 9) 호출
        @SmallNumber(maximum: 9) var width: Int = 2
    }
    
    var mixedRectangle = MixedRectangle()
    print(mixedRectangle.height)
    
    mixedRectangle.height = 20
    print(mixedRectangle.height)
    ```
    

## Property Wrappr의 projectedValue

`projectedValue`를 정의하면 추가적인 기능을 제공함.

wrappedValue를 프로퍼티래퍼를 붙여준 변수를 통해 접근할 수 있었다면, projectedValue는 $(달러사인)으로 접근할 수 있다. → 프로퍼티래퍼로 선언된 변수에 그냥 접근하면 프로퍼티래퍼, 달러사인으로 접근하면 projectedValue

만약에 property wrapper로 정의한 타입에 projectedValue를 정의하지 않았는데, 사용하는 곳에서 접근하면 `has no member` 컴파일 에러가 뜬다.

다른 타입에서 projected value에 접근할 때, property getter or an instance method `self.` 를 생략할 수 있음. 

```swift

@propertyWrapper
struct SmallNumber {
    private var number: Int
    private(set) var projectedValue: Bool

    var wrappedValue: Int {
        get { return number }
        set {
            if newValue > 12 {
                number = 12
                projectedValue = true
            } else {
                number = newValue
                projectedValue = false
            }
        }
    }

    init() {
        self.number = 0
        self.projectedValue = false
    }
}
struct SomeStructure {
    @SmallNumber var someNumber: Int
}
var someStructure = SomeStructure()

someStructure.someNumber = 4
print(someStructure.$someNumber)
// Prints "false"

someStructure.someNumber = 55
print(someStructure.$someNumber)
// Prints "true"


enum Size {
    case small, large
}

struct SizedRectangle {
    @SmallNumber var height: Int
    @SmallNumber var width: Int
    
    mutating func resize(to size: Size) -> Bool {
        switch size {
        case .small:
            height = 10
            width = 20
        case .large:
            height = 100
            width = 100
        }
        return $height || $width //self. 생략
    }
}

var sizedRectangle = SizedRectangle()
print(sizedRectangle.resize(to: .small))
print(sizedRectangle.resize(to: .large))
```

## 장점 - 재사용성, 중복제거, 가독성 향상

- **공통된 동작을 한 곳에서 정의하고 재사용할 수 있는 장점**
- 코드 중복 제거 - 여러 프로퍼티에 중복으로 적용해야하는 코드들의 중복을 제거 할 수 있다.
    - ex. trim하는 소스의 중복 제거
        
        ```swift
        // property wrapper를 사용하지 않는 경우
        struct Person {
            var name: String {
                didSet {
                    name = name.trimmingCharacters(in: .whitespacesAndNewlines)
                }
            }
            
            var address: String {
                didSet {
                    address = address.trimmingCharacters(in: .whitespacesAndNewlines)
                }
            }
            
            init(name: String, address: String) {
                self.name = name
                self.address = address
                
                self.name = self.name.trimmingCharacters(in: .whitespacesAndNewlines)
                self.address = self.address.trimmingCharacters(in: .whitespacesAndNewlines)
            }
        }
        
        let person = Person(name: " John ", address: " 123 Main Street ")
        print(person.name)    // "John" 출력
        print(person.address) // "123 Main Street" 출력
        
        // property wrapper 적용
        @propertyWrapper
        struct Trimmed {
            private(set) var value: String = ""
            
            var wrappedValue: String {
                get { value }
                set { value = newValue.trimmingCharacters(in: .whitespacesAndNewlines) }
            }
            
            init(wrappedValue: String) {
                self.wrappedValue = wrappedValue
            }
        }
        
        struct Person {
            @Trimmed
            var name: String
            
            @Trimmed
            var address: String
        }
        
        let person = Person(name: " John ", address: " 123 Main Street ")
        print(person.name)    // "John" 출력
        print(person.address) // "123 Main Street" 출력
        ```
        
- 가독성 향상: getter, setter마다 해줘야할 것을 PropertyWrapper안으로 가져가게된다. 위 장점의 순기능

## 단점 or 주의 - 추가 오버헤드, 제한된 커스터마이즈, 초기화 순서, 호환성

- 추가 오버헤드 - getter와 setter에 추가 동작이 수행되므로 실행 시간이 늘어날 수 있다. 많은 양의 데이터에 property wrapper를 적용하면 성능 측면을 고려해야한다.
- 제한된 커스터마이즈 - 추가 동작을 정의하는데 유용하지만, 이 동작을 변경하거나 확장하는 것은 제한적일 수 있다. (와닿진 않는군… 어떤 경우가 있을까?)
- 초기화 속성 순서에 주의할 것. property wrapper가 적용된 프로퍼티는 초기화 될 때 Property Wrapper의 초기화 과정을 거친다. 따라서 객체 내의 다른 프로퍼티에 의존을 하는 경우 초기화 순서에 영향을 받을 수 있으므로 주의해야 한다.

참고

---

## Advanced

[Property wrappers in Swift | Swift by Sundell](https://www.swiftbysundell.com/articles/property-wrappers-in-swift/)

## Swift Language Guide 공식문서

[https://docs.swift.org/swift-book/LanguageGuide/Properties.html#ID617](https://docs.swift.org/swift-book/LanguageGuide/Properties.html#ID617)

[What is a Property Wrapper in Swift | Sarunw](https://sarunw.com/posts/what-is-property-wrappers-in-swift/#how-to-initialize-property-wrapper)

- ChatGPT: “프로퍼티에 추가적인 동작을 캡슐화하고, 구조적이고 선언적인 문법을 제공하는 기능.”
