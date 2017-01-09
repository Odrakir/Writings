# Techniques for a Functional Dependency Injection in Swift
Yeah, I know, I couldn’t find any more jargon to put in the title, but if you bear with me, we may be able to find something useful in this article.

Some days ago I was working on a new way to architect my Swift apps based on the concepts of the Redux for Javascript. Suddenly I found myself wanting to use many free functions for some tasks (more information in a future article). Those free functions used some dependencies and I needed a way to get those dependencies inside those functions, that is, to inject those dependencies.

I did a little research about how could I do that, and this is what I got. But before I forget I want to thank [@jdortiz](https://twitter.com/jdortiz) for the tech review of this text.

# Dependency Injection
If you are reading this you probably already know what dependency injection is, why it is important and why you should already be using it in your code. Nevertheless let’s do a quick summary.

According to the accurate and usually hard to grasp definition from [Wikipedia](https://en.wikipedia.org/wiki/Dependency_injection):

> In software engineering, dependency injection is a software design pattern that implements inversion of control for resolving dependencies. A dependency is an [object](https://en.wikipedia.org/wiki/Object_(computer_science)) that can be used (a service). An injection is the passing of a dependency to a dependent object (a client) that would use it.

In plain English that translates to passing stuff objects need, to them. That means if, for example, an object has a property that holds an instance of a service, we don’t want that object to instantiate the service, we want it to receive an instance of the service from outside that object.

This will encourage decoupling as the object in question won't have to know how to construct its dependencies and thus, won't necessarily need to know what specific services it is using. Given that you are using abstract rather than concrete types for the dependencies, the object in question will just know the dependencies implement a specific interface (or conform to a specific protocol in Swift language).

This will, in consequence, make testing easier, as we will be able to easily replace any dependency by a mock object which implements that interface.

The most basic way of dependency injection, then, is just passing an object's dependencies in its constructor. This way we guarantee those dependencies are in place when we start using our object.

```swift
class MyClass
{
  private let dependency:MyDependency

  init(dependency:MyDependency)
  {
    self.dependency = dependency
  }
}
```
From this basic technique, you can complicate it as much as you want to the point of using factories or even using a whole framework to manage dependency injection. The gist of it still being: an object has to get its dependencies from somewhere else.

But you already know all that. Right?

# Functional it is

Up to this point everything should be familiar to you. But what if I told you that, when Wikipedia talks about an 'object' it isn't just talking about an instance of a class. Lets follow the hyperlink in the definition of Dependency Injection and see what an 'object' is according to Wikipedia

> In computer science, an object can be a variable, a data structure, or a *function* or a method, and as such, is a location in memory having a value and possibly referenced by an identifier.

The 'function' part is what I'm interested in. Nowadays functional is all the cool kids talk about. And lately, with all the Swift features it makes more sense every day to leave some of those old OOP paradigms behind and do the functional dance.

That’s why I wanted to write about how we should approach dependency injection in a functional way. What if the ‘object’ you are trying to pass dependencies to is not an instance of a class, but a function? Not a method either, a free function which doesn’t have access to any instance variables you can use to hold references to those injected properties.

A pure function in a sense that it just has parameters and a return value, it doesn't perform any side effects and it can't access any global variables, and in consequence it always gives the same results for the same input values.

Following there are some of the techniques I tried and a brief explanation of each of them. Which one is the best will probably, to some extent, depend on your needs.

### About the examples

The code as it is will probably make no sense in a real application. We are, apparently, working with a network API that returns its results synchronously. This is just for the sake of simplicity, in real life you should use a method (such as Observables) to get those results in a asynchronous manner.

These are the structs and classes we will use in the following examples.

```swift
typealias JSONDictionary = [String:AnyObject]

struct User
{
    var userId:String
    var name:String

    init?(json:JSONDictionary)
    {
        guard let userId = json["userName"] as? String,
            let name = json["userName"] as? String else { return nil }

        self.userId = userId
        self.name = name
    }
}

protocol Service {
    func fetchUser(userId:String) -> JSONDictionary
}

class WebService:Service {
    func fetchUser(userId: String) -> JSONDictionary {
        //We would make the actual network request here and return it in a async way, but that's stuff for another article
        //If you want to see a great way to build Network Requests take a look at: https://talk.objc.io/episodes/S01E01-networking
        return ["userId":"0001" as AnyObject,"userName":"Ricardo" as AnyObject]
    }
}

class TestService:Service {
    func fetchUser(userId: String) -> JSONDictionary {
        return [:] //Whatever
    }
}
```

## ServiceLocator

The first way to get a free function to have access to its dependencies disqualifies itself for several reasons, but I think its worth a thought anyway.

The idea behind the Service Locator pattern is to have a central structure in which dependencies get registered and can be queried for any time later. The idea is that those dependencies will implement an interface and by registering one or another we can change dependencies easily even at runtime.

There are many ways you can implement the Service Locator, but the usage will be more or less the same.

```swift
//We register a specific instance (a closure that builds it, actually) for the Service protocol
ServiceLocator.sharedLocator.register(factory: { WebService() as Service })

func loadUser(userId:String) -> User?
{
    //We get the previously registered instance
    let service:Service = ServiceLocator.sharedLocator.inject()
    let json = service.fetchUser(userId: userId)
    return User(json: json)
}

//The call doesn't have to provide that dependency
let user = loadUser(userId: "USERID")
```

This all looks fine at first, we are injecting the dependencies, the object that receives them doesn't know how to build them and it doesn't even know which specific object it's getting. It provides separation of concerns.

But this also goes against the behavior we were aiming at regarding the use of pure functions. We are getting global access to an instance so we can't guarantee no side effects are happening.

Also we are making the dependency implicit. There's nothing on the function signature that tells us it needs a dependency. The user of the function, somehow, has to know it needs that dependency and also how to register it. That could also be a big source of headaches.

In a way we are just displacing the dependency to a dependency on the Service Locator itself, which we are tightly coupled to.

Where does it make sense to use Service Locator? Well, maybe in really [cross cutting concerns](https://en.wikipedia.org/wiki/Cross-cutting_concern), like logging. We might want to be able to replace the standard logging output anytime, but it doesn't really make sense to pass it around, as it'll pollute the API, make the code harder to read and it really doesn't have any advantages.

We could use Service Locator for that, but really we should find a better way. Let's move on.

## Bastard Injection

The name is usually attributed to a kind of constructor injection in OOP, but I think this works in a very similar way with functions, and I liked the name, so...

The basic idea is to have two function signatures, one which requires a dependency as one of its parameters, and one which doesn't and uses a default object for that dependency. In Swift that can be easily done with a default value for that argument.

```swift
func loadUser(userId:String, service:Service = WebService()) -> User?
{
  let json = service.fetchUser(userId:userId)
  return User(json:json)
}

//The we can make the call without the dependency, and then use the default WebService
let user = loadUser(userId:"USERID")
//or use another instance (that conforms to the required protocol) for the dependency
let test_user = loadUser(userId:"USERID", service:TestService())
```

The problem with this approach is basically that, even if you are using a protocol as the type of the dependency, we are coupling the function with the default implementation of the dependency. So you'll have to be careful with that, it'll make more sense if the dependency is something local to the function, meaning something from the same module or layer of the application.

It won't be a good idea to use this kind of injection if the dependency is something external to the function, as you will have to carry around that dependency wherever you wanted to use that function. So code reuse will be harder. Imagine trying to reuse some class in another project but being forced to also import some other dependencies that have nothing to do with your current usage of that class, just because they are set as the default dependency.

Also this will get complicated if the injected dependency has other dependencies itself or any other parameters that need to be passed to the constructor.

In the example above it may be ok to use this kind of dependency injection, maybe it's not the best approach, but it is passable, as the load function will always require a service to load it from. So we can have a clean function signature and at the same time it'll be easy to replace that dependency for something else if necessary.

But in any case, deciding if it is ok to couple the function with the default dependency is a hard call. In general, the less the code is coupled, the better. Also, this is not a very "functional" approach, so let's look for something else.

## Currying

Currying is a technique consisting in breaking a function that takes multiple arguments into multiple functions that each take one of those arguments and are evaluated sequentially.

If you've been following Swift 3.0 news, you probably heard something about currying getting deprecated. Fortunately. it is not deprecated, is just one of its syntaxes which is being [removed](https://www.google.es/webhp?sourceid=chrome-instant&ion=1&espv=2&ie=UTF-8#q=swift%203%20currying).

Currying will still work like this

```swift
func addNumbers (_ a:Int) -> (Int) -> Int
{
    return { (_ b:Int) -> Int in
        return a + b
    }
}

let addThree = addNumbers(3)
let added = addThree(5)
```

As you can see the function `addNumbers` returns another function with the signature `(Int) -> Int`. This second function will sum its argument with the argument passed to addNumbers. This way we can create a function that always sums 3 to whatever argument we pass to it.

We are using two Ints in that example, but those arguments can be anything, they don't have to be of the same type. The great part is we'll have access to both of them when the inner function is executed. See how that can help us inject a dependency?

```swift
func loadUser(userId:String) -> (Service) -> User?
{
    return { service in
        let json = service.fetchUser(userId:userId)
        return User(json: json)
    }
}

//We get the function that will load an specific user
let loadUserAction = loadUser(userId: "0001")

//Sometime later we can inject the service to get the user
user = loadUserAction(WebService())
```

Maybe in the code above it is not easy to see the advantages of this approach. But imagine having the invocation to `loadUser` in the view controller, for example, and the dependency to the service in another class. The view controller won't have any knowledge of the actual service doing the request. And that is exactly what we aim for, decoupling.

### On parameter order

As a side note, this example surprised a friend of mine (who wants to remain anonymous) when I showed it to him. As he would do it swapping the userId and Service parameters. So first we get a function for a specific Service and then we call that function with some userId. The signature of the function being `func loadUser(service:Service) -> (String) -> User?`.
This distinction is important as it'll make the next technique easier to understand if you get this right.

So, we have two possible signatures, both of them having similar implementations as the one described before.

```swift
func loadUser(userId:String) -> (Service) -> User?
//or
func loadUser(service:Service) -> (String) -> User?
```

The first one can be used as

```swift
let loadUserAction = loadUser(userId:"0001")
//then
let user = loadUserAction(WebService())
```

Which can be read as, first create an action to get a specific user, and then run that action with this environment (the WebService).

The second one can be use like this

```swift
let loadUserAction = loadUser(service:WebService())
//then
let user = loadUserAction("0001")
```

And can be read as, first create an action that will be run in a specific environment, and then execute that action for a particular userId.

Both approaches are valid, using one or the other will ultimately depend on the usage you are giving to that function. In my case, it makes sense to generate those functions as complete actions and dispatching them to a store where the service will be injected and the function executed, but it's a matter of personal preference.

Also, the first approach will help you better understand the following technique.

Is this functional enough for you? Well, we still can go another step farther into the functional world.

## Reader Monad

Boom! The "M" word in a dependency injection article. Well, that "functional" in the title should have given you a hint. But, anyway, this is not an article about monads, if you want to learn about them in Swift you should read [this](http://www.mokacoding.com/blog/functor-applicative-monads-in-pictures/).

For our purpose, let's just think of monads as wrappers that contain some kind of value, together with some methods defined on it. The methods to be precise, are map and flatMap. Let's see what they do.

One of the most popular monads in Swift is the Optional type. As you know, an Optional is just an enum which contains a value and implements the two methods needed to be considered a Monad.

You can take a look at the signatures of map and flatMap in Optional to see what they do. Optional defines the type of the value it wraps as Wrapped.

```swift
func map<U>(transform: (Wrapped) throws -> U) -> U?
func flatMap<U>(transform: (Wrapped) throws -> U?) -> U?
```

The "map" method gets a function from Wrapped to another type and returns an Optional of the second type (that is, the second type wrapped). The "flatMap" method gets a function from Wrapped to an Optional of another type and returns an Optional of the second type.

So with map we can transform the wrapped value and with flatMap we can do the same even if the function that transforms that value returns an Optional itself. Both functions result in another Optional, which make them easy to chain.

If we were to build our own Optional type, it would look similar to this (let's call it "Maybe" in order to not collide with Swift types):

```swift
enum Maybe<T>
{
    case Some(T)
    case None

    init () {
        self = .None
    }

    init(value:T) {
        self = .Some(value)
    }

    func map<U>(_ f: (T) -> U) -> Maybe<U> {
        switch self {
        case .Some(let x): return .Some(f(x))
        case .None: return .None
        }
    }

    func flatMap<U>(_ f: (T) -> Maybe<U>) -> Maybe<U> {
        switch self {
        case .Some(let wrapped): return f(wrapped)
        case .None: return .None
        }
    }
}
```

This type of construct happens to be very useful, allowing us to chain transformations to any Optional, using either map or flatMap when needed.

```swift
let maybeInt = Maybe<Int>(value: 5)
let maybeNot = Maybe<Int>()

//We can chain map operators
let result = maybeInt
    .map{$0 * 3}
    .map(String.init) //Swift short hand syntax allows us to just send the method name

//result is .Some("15")

//If we have a function which itself returns a Maybe Monad...
func half(value:Int) -> Maybe<Int>
{
    if value % 2 == 0 {
        return Maybe<Int>(value: value/2)
    }

    return Maybe<Int>()
}

//We can chain those calls as well
let result2 = Maybe<Int>(value:16)
    .flatMap(half)
    .flatMap(half)

// result2 is .Some(4)

let result3 = result2
    .flatMap(half)
    .map{ $0 + 1 }
    .flatMap(half)

// result3 is .None
```

Ok, now that we know, at least in an informal though very practical way, what a Monad is, let's talk about the Reader Monad.

The Reader Monad is just like the Optional type, but the wrapped value is a function and the transformations change the return value of that function. It also offers a method to get the result of that function given an input parameter (that is, to run the function).

It's called the Reader Monad because it reads values from a shared environment. It's also sometimes called Environmental Monad.

```swift
class Reader<E, A> {

    let g: (E) -> A

    //closure as parameters are non-scaping by default in Swift 3, read this: https://github.com/apple/swift-evolution/blob/master/proposals/0103-make-noescape-default.md
    init(_ g: @escaping (E) -> A) {
        self.g = g
    }
    func run(_ e: E) -> A {
        return g(e)
    }
    func map<B>(_ f: @escaping (A) -> B) -> Reader<E, B> {
        return Reader<E, B>{ e in f(self.g(e)) }
    }
    func flatMap<B>(_ f: @escaping (A) -> Reader<E, B>) -> Reader<E, B> {
        return Reader<E, B>{ e in f(self.g(e)).g(e) }
    }
}
```

Comparing the code for the Reader Monad together with the one of the Optional, is easy to see the similarities. I hope this makes the Reader Monad a little less mysterious now.

This is how we will use it with the API example we are working with.

```swift
func loadUser(userId:String) -> Reader<Service, User?>
{
    return Reader { service in
        let json =  service.fetchUser(userId: userId)
        return User(json: json)
    }
}

//We get the function that will load an specific user
let loadUserAction = loadUser(userId: "0001")

//Sometime later we can inject the service to get the user
user = loadUserAction.run(WebService())
```

As with the Currying example, the real advantage of this comes when you have the creation of the 'loadUser' function and the injection of the dependency in different modules.

We can even, as with the Optional type, use map or flatMap to transform values or chain calls.

```swift
//Imagine we had another method in our service to fetch some user friends
extension Service
{
    func fetchFriends(userId:String) -> [JSONDictionary] {
        return [[:]]
    }
}

//We add a free function that calls that method using the Reader Monad approach
func loadFriends(userId:String) -> Reader<Service, [User]>
{
    return Reader { service in
        let jsonArray = service.fetchFriends(userId: userId)
        return jsonArray.flatMap(User.init)
    }
}

//And then we can chain calls to get first the user and then its friends. And that way create a Reader that gets a user's friends
let userFriends = loadUser_4(userId: "0001")
    .map { $0!.userId }
    .flatMap(loadFriends)

let friends = userFriends.run(WebService())

//How about getting multiple users in one function call?
let getManyUsers = ["1", "2", "3", "4"].map(loadUser_5).map{ $0.run(WebService()) }

```

Look mum, I'm using monads! As you can see, this is a highly functional approach. It's not too complicated, and it has great advantages in the usage of map and flatMap. But it'll probably fit best in a code base that it's already buying the functional product. It may be a little hard to understand for new comers if you present them with this out of the blue in a very OOP app.

You may think this is not a big change with respect to the currying technique, but, oh boy!, the Reader Monad can take us much farther. We can define more methods for our monads that will allow us to make some powerful operations just by combining simple blocks. For example we can use zip to combine two Reader Monads

```swift
extension Reader
{
    //We define the new zip method on Reader. It'll take another Reader and a transform function and return a new Reader
    func zip<B, U>(_ other:Reader<E,B>, f:@escaping (A,B)->U) -> Reader<E, U>
    {
        return self.flatMap { a in
            other.map { b in
                return f(a, b)
            }
        }
    }
}

//We then create a new Reader by combining two of the previous Readers
let loadUserAndFriendsAction = loadUser_5(userId: "0001").zip(loadFriends(userId: "0001")) { ($0, $1) }

//Finally we run the new Reader
let (user2, friends) = loadUserAndFriendsAction.run(WebService())
```

There are more opportunities to improve your code once you are using monads, but we'll talk about that in a future article.

# Conclusion

I hope this was a good review of the different techniques at our disposal when approaching dependency injection in a functional environment. Other than the first two, which have very limited recommended applications, the decision between using currying or the Reader Monad is very personal and it will ultimately depend on the project and the team you are working with.

Currying is very simple to understand, and while the Reader Monad doesn't add too much complexity, it might seem a little bit daunting at the first. But on the other hand, if used properly it'll add lots of functional programming resources to your code. But that is probably material for another article.
