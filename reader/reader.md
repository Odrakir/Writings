# Environment injection with Readers

Ok, imagine one of those sticker books for kids in which there are some stickers the kid can place in different backgrounds. For example, he can take the cow sticker and place it in a picture of a field, with mountains and a river flowing. Or he can place the cow in an experimental lab, or in space, or in an circus… the book offers many different landscapes to choose from.

The result is a scene in which the cow is chewing cud in a field, in an circus, or in space, although there might not be much grass to chew there…

[PICTURE OF A STICKER BOOK]

The kid first takes the sticker from the sticker’s page in the book, then decides where to place it (most times in following a totally random logic) and finally puts it in a specific environment.

Then he can take another sticker, of a cow doing something else (jumping, sleeping…) or of a totally different character doing whatever. You get the point.

## Injecting context

Ok, back in the world of adults, where you don’t play with stickers any more (unless you have kids). Imagine you are building an app, and at some point you get some data from a repository. It could be from any source, network or local, it doesn’t matter.

Lets say you are building a social app and you need to get a user for a specific email. You’ll have a method with the following signature:

```
func getUser(email:String) -> User
```

In this example, following our sticker book analogy, the sticker is the email and the result is the User object. But where is the actual landscape?

The landscape is the context where all this happens, in the sticker book it’s the circus, or the field. In our little example it might be the specific repository we are accessing to get the data: A web service, a fully fledged persistence layer or a test mock (where there might not be any grass to chew). 

For our app, let’s call this landscape, the service.

So, how do we let the previous function know which service to use, meaning, where should it get its data from? In other words, how can we inject the service? The direct solution might be just passing another parameter to the function.

```
func getUser(email:String, service:Service) -> User {
  return service.findUser(withEmail:email)
}
```

Now the service access the repository to return a user for the email we are passing. For the moment we are ignoring all kinds of errors. This is great because we can now replace the Service by anything else that conforms the same protocol for testing purposes, for example.

While there’s nothing wrong with this approach, it means that now every function of that kind has to include the service parameter. Adding a default value to the service parameter might look simpler, you can again use the previous function signature and the service will be the default one. But then you are making the dependency on the service implicit and harder to see for API users.

There’s also another problem, which might be a little harder to see. With that syntax, the caller of the function will have to pass the userId and the service, in other words, it’ll have to know that information when making the call. Two pieces of information that are, at first sight, totally unrelated: the id of the user and the service to get that user from. We are mixing two totally different responsibilities for the caller of this function: business logic and persistence or network code.

## An alternative. The Reader

You are a good programmer, so you want to separate responsibilities. You believe in the “S” in SOLID, so a class must have only one reason to change. We need to improve that code.

What if there was another way?. What if we could remove the service parameter from the function, and have a result value that allows for partial application. In a way, we first get the sticker, and then, whenever we are in the appropriate page of the sticker book and we are ready, we place it in the chosen landscape.

What we are going to do is have that function return some kind of structure, that can later accept the environment to compute the actual result of the operation. That something is a Reader and the new function signature would be:

```
func getUser(email:String) -> Reader<Service, User>
```

The Reader is generic over two types: The environment and the actual result we want to get. And as the Reader is just a function from want to the other, it'll be implemented like so:

```
public struct Reader<E, A> {

    let f : (E) -> A

    init( _ f:@escaping (E) -> A) {
        self.f = f
    }

    public func run(_ e:E) -> A {
        return f(e)
    }
}
```

As you can see a Reader is just a struct that holds a function from one generic type (the environment) to another (the result). We are only adding a method 'run' to actually run the function.

Having all that in place, the usage is pretty simple

```
func getUser(email:String) -> Reader<Service, User> {
  return Reader { service in
    return service.findUser(withEmail:email)
  }
}

let service = Service()
let user:User = getUser(email:"user_email@gmail.com").run(service)
```

With this approach we don't directly return the result from the service, but we are returning a Reader of a function from the Service to the User. So now 'getUser' gives us a function (wrapped by a Reader) we can run at any time, by passing it the Service.

For our use of readers in this example we can create an alias to simplify the syntax a little bit. We can think of the Reader from Service to anything as an action we want to run in our Service.

```
typealias Action<A> = Reader<Env, A>

func getUser(email:String) -> Action<User> {
  return Action { service in
    return service.findUser(withEmail:email)
  }
}

let getUserAction = getUser(email:"user_email@gmail.com")

// ... Somewhere else

let service = Service()
let user:User = getUserAction.run(service)
```

## One more step: The monad

This is all fine, but I might seem a little overkill just to pass a dependency. Everything up to this point could be achieved by using curried functions. But with the Reader we can go further. The Reader is actually a Monad, and as such it can implement map and flatMap.

```
extension Reader
{
    public func map<B>(_ g:@escaping (A) -> B) -> Reader<E, B> {
        return Reader<E, B> { a in g(self.f(a)) }
    }

    public func flatMap<B>(_ g:@escaping (A) -> Reader<E, B>) -> Reader<E, B> {
        return Reader<E, B>{ e in g(self.f(e)).f(e) }
    }
}
```

You can take a look at the implementations, but all you need to know is they work pretty much like the versions for Optional or Array (also monads) work in Swift.

On a side note: The Reader Monad is called like that because it "reads" from a shared environment, sometimes it's also called Environmental Monad, which to me is a much more clear name, although less cool and less common.

Usage of map is pretty straight forward

```
let getNameLength = getUser(email:"user_email@gmail.com")
  .map({ $0.name.characters.count })
```

Flatmap will allow us to chain readers easily. For example, to get User objects that are friends with a User with a specific email.

```
func getUserFriends (user:User) -> Action<[User]> {
    return Action { ctx in
        return service.findFriends(forUser: user)
    }
}

// We can compose the two Readers into a new one:
func getFriendsForUser(withEmail email:String) -> Action<[User]> {
    return getUser(name: email)
        .flatMap(getUserFriends)        
}

/* Just to clarify, the last function is equivalent to:
func getFriendsForUser(withEmail email:String) -> Action<[User]> {
    return getUser(name: email)
        .flatMap({ (user) -> Reader<Env, [User]> in
          return getUserFriends(user: user)
        })
}*/
```

Now let's try to go the extra mile and get the ages of all the friends of a user with a specific email. With flatMap that should be pretty easy, right?

```
func getUserAge(user:User) -> Action<Int> {
    return Action { service in
        return service.getAge(forUser: user)
    }
}

func getFriendsAgesForUser(withEmail email:String) -> Action<[Int]> {
    return getUser(name: email)
        .flatMap(getUserFriends)
        .flatMap(getUserAge)
        //Oops! "Cannot convert value of type '(User) -> Reader<Env, Int>' to expected argument type '([User]) -> Reader<Env, [Int]>'"
}
```

What's going on there? Simple: We are getting an array of users from getUserFriends, and getUserAge needs a single User. What can we do? We could manually write the flatMap closure and return a reader which function maps over the users array asking for their age.

```
func getFriendsAgesForUser(withEmail email:String) -> Action<[Int]> {
    return getUser(name: email)
        .flatMap(getUserFriends)
        .flatMap{ (users) -> Reader<Env, [Int]> in
            return Action { service in
                return users.map { user in
                    getUserAge(user: user).run(service)
                }
            }
        }
}
```

This works, but now it makes our code look ugly as we were aiming to have every step on our new action in just a single flatMap line. We can generalize what that flatMap does and extract the implementation to a function. If you think about it, what we are actually doing is just turning a function from User to a Reader<Service, Int>, to a function from [User] to Reader<Service, [Int]>. We can write a pure function that does just that.

```
func multiple<E,A,B>(_ f:@escaping (A) -> Reader<E, B>) -> ([A]) -> Reader<E, [B]> {
    return { v in
        return Reader { service in
            return v.flatMap {
                f($0).run(service)
            }
        }
    }
}

//And then our code is clean again
func getFriendsAgesForUser(withEmail email:String) -> Action<[Int]> {
    return getUser(name: email)
        .flatMap(getUserFriends)
        .flatMap(multiple(getUserAge))
}
```

## Conclusion

This is getting a little long, so we are going to leave it here. But the basic idea is, the Reader (being a Monad), can be easily extended to compose it with other readers. That's something that wouldn't be so easy if we were to use a similar solution with curried functions or something else instead of readers.

Next up I'll talk about making all this asynchronous and a specific challenge we'll find along the way, which we we'll solve with the help of Monad Transformers.
