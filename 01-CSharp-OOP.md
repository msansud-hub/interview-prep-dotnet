# C#, OOP and SOLID

## 1. What are the four pillars of OOP?

**Answer:** Encapsulation protects an object's state behind an interface; abstraction exposes what users need and hides implementation; inheritance derives a type from a base type; polymorphism lets different implementations respond to the same contract. For example, `IPaymentProcessor` abstracts payment processing; different processors implement it; private state and validation provide encapsulation. Prefer composition over inheritance when there is no genuine "is-a" relationship.

**Example:**
```csharp
IAnimal animal = new Dog();
Console.WriteLine(animal.Speak());                  // polymorphism: Dog.Speak()

interface IAnimal // abstraction
{
    string Speak();
}

class Animal // base class
{
    protected string Name = "Pet";
}

class Dog : Animal, IAnimal // inheritance
{
    private int age = 2;                            // encapsulation
    public string Speak() => $"{Name}, age {age}: woof";
}
```

## 2. Interface versus abstract class?

**Answer:** An interface defines a contract that multiple unrelated types can implement. Modern C# interfaces can also have default implementations. An abstract class can hold instance state, constructors, and shared implementation; a class can inherit only one base class but implement multiple interfaces. Use an interface for a capability; use an abstract class when closely related types genuinely share state or implementation.

**Example:**

```csharp
interface ILogger
{
    void Log(string message);
}

abstract class Vehicle
{
    public int Speed { get; set; }
}
```

The interface defines a capability; the abstract class provides shared state.

### Follow-up: Why use an interface if I already have a class?

**Answer:** An interface describes what a caller needs without requiring a particular implementation. This helps when there are genuinely different implementations or when an external dependency needs a test substitute.

```csharp
interface INotifier
{
    void Send(string message);
}

class EmailNotifier : INotifier
{
    public void Send(string message) => Console.WriteLine(message);
}
```

An order service can receive `INotifier` without knowing how email is sent. Avoid creating an interface for every class when no useful boundary exists.

### Follow-up: Composition versus inheritance: what is the practical difference?

**Answer:** Inheritance says one type **is a kind of** another type. Composition says an object **uses or has** another object. Composition often makes it easier to replace a behavior without building deep class hierarchies.

```csharp
class OrderService
{
    private readonly INotifier notifier;

    public OrderService(INotifier notifier) => this.notifier = notifier;
    public void Notify() => notifier.Send("Order received");
}
```

The order service **uses** a notifier; it does not need to inherit from `EmailNotifier`.

## 3. What is polymorphism? Overloading versus overriding?

**Answer:** Overloading means methods have the same name but different parameter lists; the compiler chooses an overload. Overriding replaces a base `virtual` or `abstract` implementation in a derived class; runtime dispatch selects the implementation for the actual object. A return type alone cannot distinguish overloads.

**Example:**
```csharp
Animal a = new Dog();
Console.WriteLine(a.Speak()); // woof

class Animal
{
    public virtual string Speak() => "?";
}

class Dog : Animal
{
    public override string Speak() => "woof";
}

class Printer
{
    public void Print(int x) { }
    public void Print(string x) { }
}
```

### Follow-up: What does `virtual` mean?

**Answer:** It says that a derived class **may provide a different implementation** of a base-class member. `override` provides that new implementation. The actual object's type determines which override runs.

```csharp
Animal animal = new Dog();
Console.WriteLine(animal.Sound());

class Animal
{
    public virtual string Sound() => "Some sound";
}

class Dog : Animal
{
    public override string Sound() => "Woof";
}
```

**Output:** `Woof`.

**Why?** The variable is declared as `Animal`, but the actual object is a `Dog`. Because `Sound` is virtual, the `Dog` implementation runs.

### Follow-up: What happens if the base method is not virtual?

**Answer:** A derived class cannot override a non-virtual method. It can hide it with `new`, but hiding is different from overriding: the method selected depends on the variable's declared type.

```csharp
Animal animal = new Dog();
Dog dog = new Dog();

Console.WriteLine(animal.Sound()); // Some sound
Console.WriteLine(dog.Sound());    // Woof

class Animal
{
    public string Sound() => "Some sound";
}

class Dog : Animal
{
    public new string Sound() => "Woof";
}
```

Use `override` when runtime polymorphism is intended. `new` deliberately hides the inherited member and can be confusing if used accidentally.

### Follow-up: How are `virtual` and `abstract` different?

**Answer:** A `virtual` method has a base implementation and can be overridden. An `abstract` method has no implementation in the abstract base class and must be implemented by a concrete derived class. An abstract class cannot be instantiated directly.

```csharp
abstract class Animal
{
    public abstract string Sound();
    public virtual string Sleep() => "Sleeping";
}

class Dog : Animal
{
    public override string Sound() => "Woof";
}
```

`Dog` must implement `Sound`, but it can use the inherited `Sleep` implementation. It may also override `Sleep` if needed.

### Follow-up: Can an overridden method use the base implementation?

**Answer:** Yes. `base.Method()` calls the inherited implementation. Add derived behavior before or after it when the base contract permits.

```csharp
class Animal
{
    public virtual string Describe() => "Animal";
}

class Dog : Animal
{
    public override string Describe()
    {
        return base.Describe() + ": Dog";
    }
}
```

`new Dog().Describe()` returns `Animal: Dog`.

### Follow-up: Is method overloading the same as overriding?

**Answer:** No. **Overloading** uses the same method name with different parameter lists. The compiler selects an overload based on the call's compile-time information. **Overriding** changes a virtual base implementation and uses runtime dispatch.

```csharp
static string Show(int value) => "Number";
static string Show(string value) => "Text";

Console.WriteLine(Show(5));       // Number
Console.WriteLine(Show("five"));  // Text
```

The return type alone cannot distinguish overloads.

## 4. Value type versus reference type?

**Answer:** A value-type variable holds its value; assignment copies the value. A reference-type variable holds a reference to an object; assignment copies the reference, so both variables can point to the same object. Storage location is not simply "stack versus heap": it depends on context. A struct containing a reference still copies that reference, not the referenced object.

**Example:**

```csharp
int a = 1;
int b = a;
b = 2;
// a is still 1: b is a separate copy.

var x = new List<int>();
var y = x;
y.Add(2);
// x.Count is 1: x and y refer to the same list.
```

### Follow-up: If a method receives a reference type, can it change the caller's object?

**Answer:** Yes. By default, a copy of the reference is passed, and both references point to the same object. The method can change the object. But assigning a **new object** to its local parameter does not change the caller's variable unless the parameter is passed with `ref`.

```csharp
var names = new List<string> { "Ana" };
Change(names);
Console.WriteLine(names[0]); // Sam

static void Change(List<string> list)
{
    list[0] = "Sam";
    list = new List<string> { "Kim" };
}
```

The element change is visible to the caller. Reassigning `list` only changes the local parameter, so the caller still has the list containing `Sam`.

## 5. What are boxing and unboxing?

**Answer:** Boxing converts a value type to `object` or an implemented interface, typically allocating an object and copying the value. Unboxing extracts the value, requiring the correct underlying type. Generics such as `List<int>` avoid repeated boxing compared with non-generic collections.

**Example:**

```csharp
object boxed = 42;         // Boxing
int value = (int)boxed;    // Unboxing
```

Casting this boxed `int` directly to `long` fails.

### Follow-up: What is boxing, and can it change the original value?

**Answer:** Boxing copies a value type into an object. Changing the original value later does not change the boxed copy.

```csharp
int number = 5;
object boxed = number;
number = 10;

Console.WriteLine((int)boxed); // 5
```

Unboxing requires the correct underlying type: `(long)boxed` fails because the box contains an `int`; convert after unboxing instead: `(long)(int)boxed`.

## 6. `string` versus `StringBuilder`?

**Answer:** Strings are immutable; operations that change content produce new strings. `StringBuilder` is useful for repeated or dynamically sized concatenation, such as a loop. A few simple concatenations do not automatically justify it.

**Example:**

```csharp
var builder = new System.Text.StringBuilder();
builder.Append("Hello");
builder.Append(" world");

string result = builder.ToString(); // "Hello world"
```

## 7. What do `ref`, `out`, and `in` mean?

**Answer:** They pass an argument by reference. `ref` requires initialization before the call and allows modification. `out` does not require the caller to initialize it; the called method must assign it. `in` provides a read-only reference, useful for avoiding copies of larger value types when appropriate.

**Example:**
```csharp
static void Increment(ref int n) => n++;
static void SetValue(out int n) => n = 5;
static int Read(in int n) => n;
int value = 1; Increment(ref value); // value is 2
SetValue(out int result);            // result is 5
```

## 8. `const`, `readonly`, and `static`?

**Answer:** `const` is a compile-time constant; consuming assemblies can embed its value. A `readonly` field is assigned at initialization or in a constructor, so it can vary by instance. `static` belongs to the type, not an instance. A `static readonly` field can be initialized at runtime.

**Example:**

```csharp
class Settings
{
    public const int MaxRetries = 3;
    public readonly Guid Id = Guid.NewGuid();
    public static readonly DateTime Started = DateTime.UtcNow;
}
```

`MaxRetries` is a compile-time constant; each instance has its own `Id`; `Started` belongs to the type.

## 9. `==` versus `Equals`?

**Answer:** Behavior depends on the type and operator overloads. Reference types often use reference equality by default, while strings, records, and many value types implement value equality. If you implement equality for a custom type, keep `Equals`, `GetHashCode`, and equality operators consistent. Mutable fields used as dictionary keys can cause lookup problems if changed.

**Example:**

```csharp
var first = new List<int>();
var second = new List<int>();

Console.WriteLine(first == second);       // False: different objects
Console.WriteLine("cat".Equals("cat"));   // True: equal values
```

### Follow-up: Why does equality matter for a `Dictionary` key?

**Answer:** A dictionary uses a key's hash code and equality to find entries. If two keys are considered equal, they must have equal hash codes. Avoid changing fields used for equality while the object is a key, or lookups can fail.

```csharp
var scores = new Dictionary<string, int>();
scores["Ana"] = 90;

Console.WriteLine(scores["Ana"]); // 90
```

`string` supplies consistent value equality and hashing. When implementing custom equality, implement `Equals` and `GetHashCode` consistently.

## 10. What is a record?

**Answer:** A record is a C# type designed for data-centric models with generated value-based equality and useful compiler-generated members. `record class` is a reference type; `record struct` is a value type. `with` creates a shallow copy with selected properties changed; referenced mutable objects are not deep-copied.

**Example:**

```csharp
var first = new Person("Ana");
var second = first with { Name = "Sam" };
// first.Name is "Ana"; second.Name is "Sam".

record Person(string Name);
```

## 11. Explain delegates, events and `Func`/`Action`.

**Answer:** A delegate is a type-safe reference to a method. `Func<T, TResult>` returns a value; `Action<T>` does not. An event restricts who can invoke the delegate: subscribers can add or remove handlers, but the declaring type controls raising it. Unsubscribe long-lived event handlers when needed to prevent retaining subscribers.

**Example:**

```csharp
Func<int, int> doubleIt = n => n * 2;
Action<string> print = text => Console.WriteLine(text);

int result = doubleIt(3); // 6
print("Hello");           // Prints Hello
```

A publisher can declare an event as `public event Action? Changed;` and invoke it inside the declaring class.

### Follow-up: What is a delegate, in simple words?

**Answer:** A delegate is a type that describes a method's **input and return type**. A delegate variable can hold a reference to a compatible method. You can then call that method through the variable or pass it to another method.

```csharp
static int Double(int number) => number * 2;

Func<int, int> operation = Double;
Console.WriteLine(operation(5));
```

**Output:** `10`.

**What happened?** `Func<int, int>` represents a method that takes an `int` and returns an `int`. `operation` refers to `Double`; calling `operation(5)` calls that method.

### Follow-up: Why would I pass a method to another method?

**Answer:** It lets the caller choose part of the behavior. The receiving method can keep the common steps while different callers provide different operations.

```csharp
static int Apply(int number, Func<int, int> operation)
{
    return operation(number);
}

Console.WriteLine(Apply(5, n => n * 2)); // 10
Console.WriteLine(Apply(5, n => n + 3)); // 8
```

Here `n => n * 2` is a lambda: a short way to write a function. `Apply` does not need to know whether the caller chose multiplication or addition.

### Follow-up: What is the difference between `Func`, `Action` and `Predicate`?

**Answer:**

- `Func` represents a method that **returns a value**. Its last type is the return type.
- `Action` represents a method that returns **no value** (`void`).
- `Predicate<T>` takes a `T` and returns a `bool`.

```csharp
Func<int, int> square = n => n * n;
Action<string> print = text => Console.WriteLine(text);
Predicate<int> isEven = n => n % 2 == 0;

Console.WriteLine(square(4));  // 16
print("Hello");                // Hello
Console.WriteLine(isEven(4));  // True
```

**Follow-up:** A lambda is the function expression; `Func`, `Action`, or `Predicate` is the delegate type it can be assigned to. `Func<int, bool>` can also represent a function like `isEven`.

### Follow-up: Can a delegate call more than one method?

**Answer:** Yes. A multicast delegate can hold an invocation list. `+=` adds a handler and `-=` removes one.

```csharp
Action notify = () => Console.WriteLine("Email");
notify += () => Console.WriteLine("SMS");

notify();
```

**Output:** `Email`, then `SMS`.

If a handler throws and the exception is not handled, later handlers in that invocation are not called. For delegates that return a value, a normal multicast invocation returns the last handler's result; it is usually clearer to use `Action` for notifications.

### Follow-up: What is the difference between a delegate and an event?

**Answer:** A delegate is a type that can hold methods. An `event` uses a delegate type but protects how subscribers interact with it: outside the declaring type, subscribers normally use `+=` and `-=`; they cannot invoke or replace the event directly. The publisher decides when to raise it.

```csharp
var order = new Order();
order.Created += () => Console.WriteLine("Send notification");
order.Create();

class Order
{
    public event Action? Created;

    public void Create()
    {
        Created?.Invoke();
    }
}
```

**Output:** `Send notification`. `?.Invoke()` raises the event only when at least one handler is present.

### Follow-up: Why should I sometimes unsubscribe from an event?

**Answer:** The publisher's event holds references to its subscribers. If the publisher lives longer than a subscriber, the subscriber may stay in memory longer than expected. Unsubscribe when the subscription is no longer needed, especially for long-lived publishers.

```csharp
Action handler = OnOrderCreated;
order.Created += handler;

// Later, when this subscriber no longer needs notifications:
order.Created -= handler;
```

Keep the handler reference if you need to remove it. Writing a new lambda during unsubscription generally does not remove the earlier lambda.

## 12. What are generics?

**Answer:** Generics provide type-safe reusable code, such as `List<Order>` or `Repository<T>`, without forcing everything through `object`. Constraints such as `where T : class` describe required capabilities. Generic collections often avoid boxing for value types.

**Example:**

```csharp
var ids = new List<int> { 1, 2 };
ids.Add(3);
// ids.Add("three"); // Compile-time error: not an int.
```

### Follow-up: What problem do generics solve?

**Answer:** Generics let us write a method or class once while keeping the types checked by the compiler. `T` is a placeholder for a type chosen by the caller.

```csharp
static T First<T>(T first, T second)
{
    return first;
}

int number = First(10, 20);
string name = First("Ana", "Sam");
```

For the first call, the compiler infers `T` as `int`; for the second it infers `string`. You can also write `First<int>(10, 20)` explicitly.

### Follow-up: Why use `List<int>` instead of `List<object>`?

**Answer:** `List<int>` accepts integers and returns integers without requiring casts. A `List<object>` can contain unrelated types; accessing a value as an `int` requires a cast and may involve boxing for value types.

```csharp
var numbers = new List<int> { 10, 20 };
int first = numbers[0];
// numbers.Add("hello"); // Compile-time error

var objects = new List<object> { 10, "hello" };
int value = (int)objects[0];
```

Generics improve type safety; they do not automatically make every algorithm fast.

### Follow-up: What is a generic constraint and why is it useful?

**Answer:** A constraint tells the compiler what a type argument must support. That lets the generic code safely use those capabilities.

```csharp
interface IHasId
{
    int Id { get; }
}

class Store<T> where T : IHasId
{
    public int GetId(T item) => item.Id;
}
```

`where T : IHasId` means `Store<T>` can use `item.Id`. Without the constraint, the compiler cannot assume every possible `T` has an `Id`.

Other examples include `where T : class`, `where T : struct`, and `where T : new()` for a public parameterless constructor.

### Follow-up: What is covariance? Why can't I add a `Cat` to a list of dogs?

**Answer:** Covariance lets some read-oriented generic interfaces use a more specific type where a less specific type is expected. For example, `IEnumerable<Dog>` can be used as `IEnumerable<Animal>`. But `List<Dog>` cannot be assigned to `List<Animal>`: otherwise someone could add a `Cat` to a list that should contain only dogs.

```csharp
List<Dog> dogs = new();
IEnumerable<Animal> animals = dogs; // Allowed
// List<Animal> list = dogs;       // Not allowed

class Animal { }
class Dog : Animal { }
class Cat : Animal { }
```

Variance applies to compatible reference types and to interfaces or delegates declared with appropriate `out`/`in` variance; it does not make all generic collections interchangeable.

## 13. `IEnumerable<T>` versus `IQueryable<T>`?

**Answer:** `IEnumerable<T>` represents iteration in .NET; LINQ operators generally execute in the application when enumerated. `IQueryable<T>` builds expression trees that a provider, such as EF Core, can translate into a remote query. Materializing early with `ToList` can move filtering into memory; provider-specific expressions may not translate. Both can use deferred execution.

**Example:**

```csharp
// EF Core translates the filter into a database query.
var databaseFilter = db.Orders
    .Where(o => o.Total > 100)
    .ToList();

// Rows are loaded first; the filter then runs in memory.
var memoryFilter = db.Orders
    .ToList()
    .Where(o => o.Total > 100);
```

### Follow-up: Why does a LINQ query sometimes return different results later?

**Answer:** Many LINQ operations are deferred: creating the query describes the work, while enumeration runs it. If the source changes before enumeration, the result can change.

```csharp
var numbers = new List<int> { 1, 2 };
var evens = numbers.Where(n => n % 2 == 0);

numbers.Add(4);
Console.WriteLine(string.Join(", ", evens)); // 2, 4
```

Call `ToList()` to evaluate now and create a list of the current results. For reference-type elements, the new list still contains references to the same objects; it is not a deep copy.

### Follow-up: Why does `IQueryable` matter with EF Core?

**Answer:** `IQueryable` lets a provider build and translate a query, often into SQL. Filtering **before** materialization can reduce the rows brought into the application.

```csharp
// Filter in the database, when the provider can translate it.
var orders = await db.Orders
    .Where(o => o.Total > 100)
    .ToListAsync();
```

If `ToListAsync()` runs first, later LINQ filtering happens in memory. Translation depends on the provider; check generated SQL and query performance.

## 14. How do `async` and `await` work?

**Answer:** `await` asynchronously waits for an incomplete awaitable and resumes the method when it completes. It does not automatically create a thread or make CPU work faster. For I/O, use asynchronous APIs and propagate `async` through the call chain. For CPU-bound work, consider background processing or deliberate thread-pool offloading where appropriate. Avoid blocking on `.Result` or `.Wait()`, which can cause thread starvation and, in some contexts, deadlocks.

**Example:**

```csharp
string body = await client.GetStringAsync("https://example.com");
Console.WriteLine(body);
```

While the I/O is pending, the method can yield instead of blocking a thread.

### Follow-up: Does `async` create a new thread?

**Answer:** No. `async` lets a method use `await`; it does not itself start a new thread. For asynchronous I/O, the method can yield while the operation is pending and continue when it finishes.

```csharp
static async Task<string> FetchAsync(HttpClient client, string url)
{
    string body = await client.GetStringAsync(url);
    return body;
}
```

This is useful for I/O such as HTTP and database requests. CPU-heavy work needs a different approach; marking it `async` alone does not speed it up.

## 15. When would you use `Task.WhenAll`?

**Answer:** Use it to wait for independent asynchronous operations concurrently, for example fetching two unrelated resources. Await the returned task and handle failures; one or more failed tasks make the combined task fail. Limit concurrency for large collections to avoid overwhelming downstream services. Do not use parallel operations on the same EF Core `DbContext`.

**Example:**

```csharp
Task<string> first = client.GetStringAsync(url1);
Task<string> second = client.GetStringAsync(url2);

string[] bodies = await Task.WhenAll(first, second);
```

Both requests start before the code waits for their results.

### Follow-up: Does `Task.WhenAll` automatically make things safe?

**Answer:** No. It waits for tasks together, but shared resources still need to support concurrent use. For example, do not run two operations concurrently on the same EF Core `DbContext`.

```csharp
Task<string> first = client.GetStringAsync(url1);
Task<string> second = client.GetStringAsync(url2);

string[] results = await Task.WhenAll(first, second);
```

Both operations start before waiting. If one fails, the combined task fails; use bounded concurrency when there are many requests.

## 16. What is `CancellationToken`?

**Answer:** It communicates a cooperative cancellation request. Accept a token, pass it to downstream asynchronous operations, and stop promptly when cancellation is requested. Cancellation is not guaranteed to undo an operation already committed by a remote service.

**Example:**

```csharp
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
using var response = await client.GetAsync(url, cts.Token);
```

The token requests cancellation after five seconds if the call has not completed.

### Follow-up: If cancellation happens, is the operation always undone?

**Answer:** No. A cancellation token is a request to stop, not an automatic rollback across systems. A remote server may already have completed a payment even if the caller stopped waiting.

```csharp
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
using var response = await client.GetAsync(url, cts.Token);
```

For a critical payment, use a stable idempotency key and check the payment's actual status after an uncertain timeout.

## 17. How does garbage collection work? Do you still need `Dispose`?

**Answer:** The garbage collector reclaims managed objects that are no longer reachable; it uses generations to optimize collection. It does not guarantee prompt release of resources such as file handles or connections. Use `using`/`await using` with `IDisposable`/`IAsyncDisposable` resources to release them deterministically. Do not force collection as a routine optimization.

**Example:**

```csharp
using var stream = File.OpenRead(path);
// Read the file here.
```

The stream is closed when its containing scope exits, without waiting for garbage collection.

### Follow-up: If .NET has garbage collection, why use `using`?

**Answer:** Garbage collection eventually reclaims unreachable managed objects. It does not promise to release a file, socket, or database connection promptly. `using` calls `Dispose` at the end of the scope, even if an exception occurs.

```csharp
using var reader = File.OpenText(path);
string? firstLine = await reader.ReadLineAsync();
```

The reader is disposed when the containing scope exits. Use `await using` for types implementing `IAsyncDisposable`.

## 18. Explain SOLID with examples.

**Answer:** **S**ingle responsibility: change order pricing separately from notification. **O**pen/closed: add a new pricing strategy without editing every caller. **L**iskov substitution: a subtype must honor the base contract. **I**nterface segregation: offer small focused interfaces instead of forcing unused methods. **D**ependency inversion: business logic depends on abstractions, not concrete infrastructure. Apply these pragmatically, not by adding unnecessary interfaces.

**Example:**

```csharp
interface IDiscount
{
    decimal Apply(decimal price);
}
```

An order service can depend on this small contract and accept different implementations. Each must honor the same pricing contract.

## 19. What is dependency injection?

**Answer:** Dependencies are supplied from outside a class, commonly through its constructor, rather than constructed inside it. This makes dependencies explicit and supports substituting implementations in tests. It does not mean every class needs an interface.

**Example:**

```csharp
class OrderService
{
    private readonly IOrderRepository repository;

    public OrderService(IOrderRepository repository)
    {
        this.repository = repository;
    }
}
```

The service receives its dependency instead of constructing a database repository itself.

### Follow-up: Why should a class receive a dependency instead of creating it?

**Answer:** Creating a dependency inside a class ties that class to a particular implementation. Passing it in makes the dependency visible and allows another implementation to be supplied when needed.

```csharp
class OrderService
{
    private readonly INotifier notifier;

    public OrderService(INotifier notifier)
    {
        this.notifier = notifier;
    }

    public void Confirm() => notifier.Send("Order confirmed");
}
```

`OrderService` uses `INotifier`. The application can supply an email notifier; a test can supply a fake. The interface and implementations must be defined separately.

## 20. How should exceptions be handled?

**Answer:** Catch exceptions where you can add context, recover, or translate them to a meaningful boundary response. Do not swallow them or expose stack traces to API consumers. Use `throw;` to rethrow while preserving the original stack. Use validation or explicit results for expected business outcomes when that makes the contract clearer.

**Example:**

```csharp
try
{
    await SaveAsync();
}
catch (IOException ex)
{
    logger.LogError(ex, "Save failed");
    throw;
}
```

The exception is logged and rethrown with its original stack.

### Follow-up: What is the difference between `throw;` and `throw ex;`?

**Answer:** Inside a catch block, `throw;` rethrows the current exception and preserves its original stack trace. `throw ex;` resets the stack trace to the rethrow point, making the original failure harder to investigate.

```csharp
try
{
    await SaveAsync();
}
catch (IOException ex)
{
    logger.LogError(ex, "Save failed");
    throw;
}
```

Catch only when you can handle the exception, add useful context, or translate it at an appropriate boundary.

## Quick self-check

1. If `Animal a = new Dog()` calls a virtual method, which implementation runs? **Dog's override.**
2. Is an event the same as a delegate? **No; an event restricts how subscribers access a delegate-backed notification.**
3. Can `List<Dog>` become `List<Animal>`? **No; `IEnumerable<Dog>` can be used as `IEnumerable<Animal>` for reference types.**
4. Does `await` always start a thread? **No.**
5. Does cancellation prove a remote payment did not happen? **No; check its status.**
