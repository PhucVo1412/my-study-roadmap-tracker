# OOP, SOLID & Common Design Patterns

A practical guide with examples in **Java** and **C#**.

---

## Table of Contents

1. [Object-Oriented Programming (OOP)](#1-object-oriented-programming-oop)
   - [Encapsulation](#11-encapsulation)
   - [Abstraction](#12-abstraction)
   - [Inheritance](#13-inheritance)
   - [Polymorphism](#14-polymorphism)
   - [Composition over Inheritance](#15-composition-over-inheritance)
2. [SOLID Principles](#2-solid-principles)
   - [S — Single Responsibility](#21-s--single-responsibility-principle-srp)
   - [O — Open/Closed](#22-o--openclosed-principle-ocp)
   - [L — Liskov Substitution](#23-l--liskov-substitution-principle-lsp)
   - [I — Interface Segregation](#24-i--interface-segregation-principle-isp)
   - [D — Dependency Inversion](#25-d--dependency-inversion-principle-dip)
3. [Design Patterns](#3-design-patterns)
   - [Strategy](#31-strategy-pattern)
   - [Factory](#32-factory-pattern)
   - [Builder](#33-builder-pattern)
4. [Collections](#4-collections)
   - [Choosing List / Set / Map](#41-choosing-list--set--map)
   - [How HashMap Works](#42-how-hashmap-works)
   - [equals() and hashCode()](#43-equals-and-hashcode)
5. [Streams, Optional & Exceptions](#5-streams-optional--exceptions)
   - [Stream API (Java) / LINQ (C#)](#51-stream-api-java--linq-c)
   - [Optional (Java) / Nullable (C#)](#52-optional-java--nullable-c)
   - [Handling Exceptions Properly](#53-handling-exceptions-properly)
6. [Modern Java (17 & 21)](#6-modern-java-17--21)
   - [Records](#61-records)
   - [Sealed Classes](#62-sealed-classes)
   - [Pattern Matching](#63-pattern-matching)
   - [Virtual Threads (Concept)](#64-virtual-threads-concept)
7. [Concurrency Basics](#7-concurrency-basics)
   - [Thread Pools: ExecutorService / Task](#71-thread-pools-executorservice--task)
   - [CompletableFuture / async-await](#72-completablefuture--async-await)
   - [Race Conditions](#73-race-conditions)
   - [Writing Thread-Safe Code](#74-writing-thread-safe-code)
8. [Cheat Sheet](#8-cheat-sheet)

---

## 1. Object-Oriented Programming (OOP)

OOP is a way of structuring software around **objects**: units that bundle **state** (fields) and **behavior** (methods). A **class** is the blueprint; an **object** is an instance of that blueprint.

OOP rests on four pillars.

### 1.1 Encapsulation

**Hide internal state and expose only what is needed through a controlled interface.** Outside code cannot put the object into an invalid state because it cannot touch the fields directly.

**Java**

```java
public class BankAccount {
    private double balance;            // hidden

    public double getBalance() {
        return balance;
    }

    public void deposit(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        balance += amount;
    }

    public void withdraw(double amount) {
        if (amount > balance) {
            throw new IllegalStateException("Insufficient funds");
        }
        balance -= amount;
    }
}
```

**C#**

```csharp
public class BankAccount
{
    public decimal Balance { get; private set; }   // read-only from outside

    public void Deposit(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException("Amount must be positive");
        Balance += amount;
    }

    public void Withdraw(decimal amount)
    {
        if (amount > Balance)
            throw new InvalidOperationException("Insufficient funds");
        Balance -= amount;
    }
}
```

> **Why it matters:** the rules ("no negative deposits", "no overdraft") live in one place and can't be bypassed.

### 1.2 Abstraction

**Expose *what* an object does, hide *how* it does it.** Callers depend on a simple contract, not on implementation details.

**Java**

```java
public interface Notifier {
    void send(String to, String message);
}

public class EmailNotifier implements Notifier {
    @Override
    public void send(String to, String message) {
        // SMTP connection, headers, retries... all hidden
        System.out.println("Email to " + to + ": " + message);
    }
}
```

**C#**

```csharp
public interface INotifier
{
    void Send(string to, string message);
}

public class EmailNotifier : INotifier
{
    public void Send(string to, string message)
    {
        // SMTP connection, headers, retries... all hidden
        Console.WriteLine($"Email to {to}: {message}");
    }
}
```

> **Encapsulation vs. Abstraction:** encapsulation is about *protecting data*; abstraction is about *simplifying the interface*. They work together.

### 1.3 Inheritance

**A class can derive from another class and reuse or extend its behavior.** It models an **"is-a"** relationship.

**Java**

```java
public abstract class Animal {
    protected final String name;

    protected Animal(String name) { this.name = name; }

    public void eat() { System.out.println(name + " is eating"); }

    public abstract String makeSound();
}

public class Dog extends Animal {
    public Dog(String name) { super(name); }

    @Override
    public String makeSound() { return "Woof"; }
}
```

**C#**

```csharp
public abstract class Animal
{
    protected readonly string Name;

    protected Animal(string name) => Name = name;

    public void Eat() => Console.WriteLine($"{Name} is eating");

    public abstract string MakeSound();
}

public class Dog : Animal
{
    public Dog(string name) : base(name) { }

    public override string MakeSound() => "Woof";
}
```

> **Note:** in C#, methods are **not** virtual by default — you must mark them `virtual` (or `abstract`) to allow overriding. In Java, all non-`final`, non-`static`, non-`private` methods are overridable.

### 1.4 Polymorphism

**One interface, many forms.** Code written against a base type works with any subtype, and the correct behavior is chosen at runtime.

**Java**

```java
List<Animal> animals = List.of(new Dog("Rex"), new Cat("Tom"));

for (Animal a : animals) {
    System.out.println(a.makeSound());   // "Woof", then "Meow"
}
```

**C#**

```csharp
var animals = new List<Animal> { new Dog("Rex"), new Cat("Tom") };

foreach (var a in animals)
{
    Console.WriteLine(a.MakeSound());    // "Woof", then "Meow"
}
```

There are two kinds:

| Kind | Also called | Mechanism | Example |
|---|---|---|---|
| Compile-time | Static polymorphism | Method **overloading** | `add(int, int)` vs. `add(double, double)` |
| Runtime | Dynamic polymorphism | Method **overriding** | `Dog.makeSound()` overrides `Animal.makeSound()` |

### 1.5 Composition over Inheritance

Inheritance creates tight coupling: a change in the parent can break every child. Prefer **composition** ("has-a") when you only need to *reuse behavior*, and keep inheritance for genuine "is-a" relationships.

**Java**

```java
// Instead of: class Car extends Engine  ❌ (a car is not an engine)
public class Car {
    private final Engine engine;          // a car HAS an engine ✅

    public Car(Engine engine) { this.engine = engine; }

    public void start() { engine.start(); }
}
```

**C#**

```csharp
public class Car
{
    private readonly IEngine _engine;

    public Car(IEngine engine) => _engine = engine;

    public void Start() => _engine.Start();
}
```

Now you can swap a `PetrolEngine` for an `ElectricEngine` without touching `Car`.

---

## 2. SOLID Principles

SOLID is a set of five design principles (popularized by Robert C. Martin) that make code **easier to change, test and understand**.

| Letter | Principle | One-liner |
|---|---|---|
| **S** | Single Responsibility | A class should have only one reason to change. |
| **O** | Open/Closed | Open for extension, closed for modification. |
| **L** | Liskov Substitution | Subtypes must be usable wherever their base type is expected. |
| **I** | Interface Segregation | Don't force clients to depend on methods they don't use. |
| **D** | Dependency Inversion | Depend on abstractions, not on concrete implementations. |

### 2.1 S — Single Responsibility Principle (SRP)

> *A class should have one, and only one, reason to change.*

**❌ Bad** — one class handles business logic, persistence *and* formatting:

**Java**

```java
public class Invoice {
    public double calculateTotal() { /* ... */ }
    public void saveToDatabase()   { /* ... */ }   // persistence concern
    public String toPdf()          { /* ... */ }   // presentation concern
}
```

**C#**

```csharp
public class Invoice
{
    public decimal CalculateTotal() { /* ... */ }
    public void SaveToDatabase()    { /* ... */ }   // persistence concern
    public byte[] ToPdf()           { /* ... */ }   // presentation concern
}
```

**✅ Good** — split by responsibility:

**Java**

```java
public class Invoice {
    private final List<LineItem> items;
    public Invoice(List<LineItem> items) { this.items = items; }

    public double calculateTotal() {
        return items.stream().mapToDouble(LineItem::getPrice).sum();
    }
}

public class InvoiceRepository {
    public void save(Invoice invoice) { /* DB logic */ }
}

public class InvoicePdfExporter {
    public byte[] export(Invoice invoice) { /* PDF logic */ return new byte[0]; }
}
```

**C#**

```csharp
public class Invoice
{
    private readonly List<LineItem> _items;
    public Invoice(List<LineItem> items) => _items = items;

    public decimal CalculateTotal() => _items.Sum(i => i.Price);
}

public class InvoiceRepository
{
    public void Save(Invoice invoice) { /* DB logic */ }
}

public class InvoicePdfExporter
{
    public byte[] Export(Invoice invoice) { /* PDF logic */ return Array.Empty<byte>(); }
}
```

Now a change to the database schema only touches `InvoiceRepository`; a new PDF layout only touches `InvoicePdfExporter`.

### 2.2 O — Open/Closed Principle (OCP)

> *Software entities should be open for extension but closed for modification.*

**❌ Bad** — every new shape forces you to edit this method:

**Java**

```java
public double area(Object shape) {
    if (shape instanceof Circle c)    return Math.PI * c.r * c.r;
    if (shape instanceof Rectangle r) return r.w * r.h;
    // add Triangle? → modify this method again
    throw new IllegalArgumentException();
}
```

**C#**

```csharp
public double Area(object shape) => shape switch
{
    Circle c    => Math.PI * c.R * c.R,
    Rectangle r => r.W * r.H,
    // add Triangle? → modify this method again
    _ => throw new ArgumentException("Unknown shape")
};
```

**✅ Good** — add new shapes by adding new classes:

**Java**

```java
public interface Shape {
    double area();
}

public class Circle implements Shape {
    private final double r;
    public Circle(double r) { this.r = r; }
    public double area() { return Math.PI * r * r; }
}

public class Rectangle implements Shape {
    private final double w, h;
    public Rectangle(double w, double h) { this.w = w; this.h = h; }
    public double area() { return w * h; }
}

// Never needs to change when a new Shape is added
public double totalArea(List<Shape> shapes) {
    return shapes.stream().mapToDouble(Shape::area).sum();
}
```

**C#**

```csharp
public interface IShape
{
    double Area();
}

public class Circle : IShape
{
    private readonly double _r;
    public Circle(double r) => _r = r;
    public double Area() => Math.PI * _r * _r;
}

public class Rectangle : IShape
{
    private readonly double _w, _h;
    public Rectangle(double w, double h) { _w = w; _h = h; }
    public double Area() => _w * _h;
}

public double TotalArea(IEnumerable<IShape> shapes) => shapes.Sum(s => s.Area());
```

### 2.3 L — Liskov Substitution Principle (LSP)

> *If `S` is a subtype of `T`, objects of type `T` may be replaced with objects of type `S` without breaking the program.*

**❌ Bad** — the classic Rectangle/Square problem:

**Java**

```java
public class Rectangle {
    protected int width, height;
    public void setWidth(int w)  { width = w; }
    public void setHeight(int h) { height = h; }
    public int area() { return width * height; }
}

public class Square extends Rectangle {
    @Override public void setWidth(int w)  { width = w; height = w; }
    @Override public void setHeight(int h) { width = h; height = h; }
}

// Client code that works for Rectangle but breaks for Square:
Rectangle r = new Square();
r.setWidth(5);
r.setHeight(4);
assert r.area() == 20;   // ❌ fails: area is 16
```

**C#**

```csharp
public class Rectangle
{
    public virtual int Width  { get; set; }
    public virtual int Height { get; set; }
    public int Area() => Width * Height;
}

public class Square : Rectangle
{
    public override int Width  { set { base.Width = value; base.Height = value; } }
    public override int Height { set { base.Width = value; base.Height = value; } }
}

// Client code that works for Rectangle but breaks for Square:
Rectangle r = new Square();
r.Width = 5;
r.Height = 4;
Debug.Assert(r.Area() == 20);   // ❌ fails: area is 16
```

A `Square` *is* a rectangle mathematically, but it **doesn't behave** like this `Rectangle` class promises.

**✅ Good** — model them as siblings under a shared abstraction:

**Java**

```java
public interface Shape { int area(); }

public record Rectangle(int width, int height) implements Shape {
    public int area() { return width * height; }
}

public record Square(int side) implements Shape {
    public int area() { return side * side; }
}
```

**C#**

```csharp
public interface IShape { int Area(); }

public record Rectangle(int Width, int Height) : IShape
{
    public int Area() => Width * Height;
}

public record Square(int Side) : IShape
{
    public int Area() => Side * Side;
}
```

**Common LSP smells:**
- An override throws `UnsupportedOperationException` / `NotImplementedException`.
- An override strengthens preconditions (accepts less) or weakens postconditions (guarantees less).
- Client code checks `instanceof` / `is` to special-case a subtype.

### 2.4 I — Interface Segregation Principle (ISP)

> *Clients should not be forced to depend on interfaces they do not use.*

**❌ Bad** — a "fat" interface:

**Java**

```java
public interface Worker {
    void work();
    void eat();
    void attendMeeting();
}

public class Robot implements Worker {
    public void work() { /* ... */ }
    public void eat() { throw new UnsupportedOperationException(); }  // ❌
    public void attendMeeting() { throw new UnsupportedOperationException(); } // ❌
}
```

**C#**

```csharp
public interface IWorker
{
    void Work();
    void Eat();
    void AttendMeeting();
}

public class Robot : IWorker
{
    public void Work() { /* ... */ }
    public void Eat() => throw new NotSupportedException();            // ❌
    public void AttendMeeting() => throw new NotSupportedException();  // ❌
}
```

**✅ Good** — small, focused interfaces:

**Java**

```java
public interface Workable  { void work(); }
public interface Eatable   { void eat(); }
public interface Meetable  { void attendMeeting(); }

public class Human implements Workable, Eatable, Meetable {
    public void work() { }
    public void eat() { }
    public void attendMeeting() { }
}

public class Robot implements Workable {
    public void work() { }
}
```

**C#**

```csharp
public interface IWorkable { void Work(); }
public interface IEatable  { void Eat(); }
public interface IMeetable { void AttendMeeting(); }

public class Human : IWorkable, IEatable, IMeetable
{
    public void Work() { }
    public void Eat() { }
    public void AttendMeeting() { }
}

public class Robot : IWorkable
{
    public void Work() { }
}
```

### 2.5 D — Dependency Inversion Principle (DIP)

> 1. *High-level modules should not depend on low-level modules. Both should depend on abstractions.*
> 2. *Abstractions should not depend on details. Details should depend on abstractions.*

**❌ Bad** — the service is welded to a specific database:

**Java**

```java
public class OrderService {
    private final MySqlOrderRepository repo = new MySqlOrderRepository(); // ❌ hard-coded

    public void placeOrder(Order order) { repo.save(order); }
}
```

**C#**

```csharp
public class OrderService
{
    private readonly SqlServerOrderRepository _repo = new();   // ❌ hard-coded

    public void PlaceOrder(Order order) => _repo.Save(order);
}
```

**✅ Good** — depend on an interface and inject the implementation:

**Java**

```java
public interface OrderRepository {
    void save(Order order);
}

public class MySqlOrderRepository implements OrderRepository {
    public void save(Order order) { /* MySQL logic */ }
}

public class OrderService {
    private final OrderRepository repo;

    public OrderService(OrderRepository repo) {   // constructor injection
        this.repo = repo;
    }

    public void placeOrder(Order order) { repo.save(order); }
}

// Wiring (or let Spring do it)
OrderService service = new OrderService(new MySqlOrderRepository());
```

**C#**

```csharp
public interface IOrderRepository
{
    void Save(Order order);
}

public class SqlServerOrderRepository : IOrderRepository
{
    public void Save(Order order) { /* SQL Server logic */ }
}

public class OrderService
{
    private readonly IOrderRepository _repo;

    public OrderService(IOrderRepository repo) => _repo = repo;

    public void PlaceOrder(Order order) => _repo.Save(order);
}

// Wiring with ASP.NET Core DI
builder.Services.AddScoped<IOrderRepository, SqlServerOrderRepository>();
builder.Services.AddScoped<OrderService>();
```

**Benefits:** swap the database, or pass an in-memory fake in unit tests, without touching `OrderService`.

> **DIP vs. DI:** *Dependency Inversion* is the **principle** (depend on abstractions). *Dependency Injection* is a **technique** (pass dependencies in from outside) that helps you follow it. A *DI container* (Spring, ASP.NET Core's built-in container) automates the wiring.

---

## 3. Design Patterns

Design patterns are **reusable solutions to recurring design problems**. The classic catalogue (the "Gang of Four" book, 1994) groups them into three families:

| Family | Purpose | Examples |
|---|---|---|
| **Creational** | How objects are created | Factory Method, Abstract Factory, Builder, Singleton, Prototype |
| **Structural** | How objects are composed | Adapter, Decorator, Facade, Proxy, Composite |
| **Behavioral** | How objects communicate | Strategy, Observer, Command, State, Template Method |

Below are three of the most commonly used ones.

### 3.1 Strategy Pattern

**Family:** Behavioral

**Intent:** define a family of algorithms, put each in its own class, and make them interchangeable at runtime.

**Use it when:**
- You have several ways of doing the same thing (payment methods, sorting, pricing rules, compression…).
- You find a growing `if/else` or `switch` that picks an algorithm.

```
          ┌───────────────┐        uses       ┌────────────────────┐
          │   Context     │──────────────────▶│ «interface»        │
          │ (Checkout)    │                   │ PaymentStrategy    │
          └───────────────┘                   │ + pay(amount)      │
                                              └─────────▲──────────┘
                                   ┌────────────────────┼────────────────────┐
                          ┌────────┴───────┐   ┌────────┴───────┐   ┌────────┴───────┐
                          │ CreditCardPay  │   │  PayPalPay     │   │  CryptoPay     │
                          └────────────────┘   └────────────────┘   └────────────────┘
```

**Java**

```java
// Strategy
public interface PaymentStrategy {
    void pay(double amount);
}

// Concrete strategies
public class CreditCardPayment implements PaymentStrategy {
    private final String cardNumber;
    public CreditCardPayment(String cardNumber) { this.cardNumber = cardNumber; }

    @Override
    public void pay(double amount) {
        System.out.printf("Paid %.2f with card %s%n", amount, cardNumber);
    }
}

public class PayPalPayment implements PaymentStrategy {
    private final String email;
    public PayPalPayment(String email) { this.email = email; }

    @Override
    public void pay(double amount) {
        System.out.printf("Paid %.2f via PayPal (%s)%n", amount, email);
    }
}

// Context
public class Checkout {
    private PaymentStrategy strategy;

    public Checkout(PaymentStrategy strategy) { this.strategy = strategy; }

    public void setStrategy(PaymentStrategy strategy) { this.strategy = strategy; }

    public void complete(double amount) { strategy.pay(amount); }
}

// Usage
Checkout checkout = new Checkout(new CreditCardPayment("4111-xxxx"));
checkout.complete(99.90);

checkout.setStrategy(new PayPalPayment("me@example.com"));  // swap at runtime
checkout.complete(15.00);

// Java 8+: a functional interface can be passed as a lambda
checkout.setStrategy(amount -> System.out.println("Paid " + amount + " in cash"));
```

**C#**

```csharp
// Strategy
public interface IPaymentStrategy
{
    void Pay(decimal amount);
}

// Concrete strategies
public class CreditCardPayment : IPaymentStrategy
{
    private readonly string _cardNumber;
    public CreditCardPayment(string cardNumber) => _cardNumber = cardNumber;

    public void Pay(decimal amount) =>
        Console.WriteLine($"Paid {amount:F2} with card {_cardNumber}");
}

public class PayPalPayment : IPaymentStrategy
{
    private readonly string _email;
    public PayPalPayment(string email) => _email = email;

    public void Pay(decimal amount) =>
        Console.WriteLine($"Paid {amount:F2} via PayPal ({_email})");
}

// Context
public class Checkout
{
    public IPaymentStrategy Strategy { get; set; }

    public Checkout(IPaymentStrategy strategy) => Strategy = strategy;

    public void Complete(decimal amount) => Strategy.Pay(amount);
}

// Usage
var checkout = new Checkout(new CreditCardPayment("4111-xxxx"));
checkout.Complete(99.90m);

checkout.Strategy = new PayPalPayment("me@example.com");    // swap at runtime
checkout.Complete(15.00m);
```

> **C# tip:** for simple cases, a delegate like `Func<decimal, bool>` or `Action<decimal>` can act as a lightweight strategy.

**SOLID link:** Strategy is a direct application of **OCP** (add a new algorithm without editing `Checkout`) and **DIP** (`Checkout` depends on the interface).

**Pros:** removes conditionals, easy to add algorithms, each algorithm is testable in isolation.
**Cons:** more classes; clients must know which strategy to choose (often solved by pairing with a Factory).

### 3.2 Factory Pattern

**Family:** Creational

**Intent:** move object-creation logic out of client code so the client doesn't need to know the concrete class it gets back.

"Factory" usually refers to one of three related ideas:

| Variant | What it is |
|---|---|
| **Simple Factory** | A single method/class with a `switch` that returns the right subtype. (Not an official GoF pattern, but the most common in practice.) |
| **Factory Method** (GoF) | A base class declares an abstract creation method; subclasses decide which concrete class to instantiate. |
| **Abstract Factory** (GoF) | An interface for creating *families* of related objects (e.g. `WindowsUIFactory` creates `WindowsButton` + `WindowsCheckbox`). |

#### Simple Factory

**Java**

```java
public interface Notification {
    void send(String message);
}

public class EmailNotification implements Notification {
    public void send(String message) { System.out.println("Email: " + message); }
}

public class SmsNotification implements Notification {
    public void send(String message) { System.out.println("SMS: " + message); }
}

public class PushNotification implements Notification {
    public void send(String message) { System.out.println("Push: " + message); }
}

public enum Channel { EMAIL, SMS, PUSH }

public final class NotificationFactory {
    private NotificationFactory() {}

    public static Notification create(Channel channel) {
        return switch (channel) {
            case EMAIL -> new EmailNotification();
            case SMS   -> new SmsNotification();
            case PUSH  -> new PushNotification();
        };
    }
}

// Usage — the client never calls `new` on a concrete class
Notification n = NotificationFactory.create(Channel.SMS);
n.send("Your code is 123456");
```

**C#**

```csharp
public interface INotification
{
    void Send(string message);
}

public class EmailNotification : INotification
{
    public void Send(string message) => Console.WriteLine($"Email: {message}");
}

public class SmsNotification : INotification
{
    public void Send(string message) => Console.WriteLine($"SMS: {message}");
}

public class PushNotification : INotification
{
    public void Send(string message) => Console.WriteLine($"Push: {message}");
}

public enum Channel { Email, Sms, Push }

public static class NotificationFactory
{
    public static INotification Create(Channel channel) => channel switch
    {
        Channel.Email => new EmailNotification(),
        Channel.Sms   => new SmsNotification(),
        Channel.Push  => new PushNotification(),
        _ => throw new ArgumentOutOfRangeException(nameof(channel))
    };
}

// Usage
INotification n = NotificationFactory.Create(Channel.Sms);
n.Send("Your code is 123456");
```

#### Factory Method (GoF)

The creation step is a method that subclasses override. The base class contains the shared workflow.

**Java**

```java
public abstract class Logistics {
    // Factory method
    protected abstract Transport createTransport();

    // Shared workflow that uses the product
    public void planDelivery(String cargo) {
        Transport t = createTransport();
        t.deliver(cargo);
    }
}

public class RoadLogistics extends Logistics {
    @Override protected Transport createTransport() { return new Truck(); }
}

public class SeaLogistics extends Logistics {
    @Override protected Transport createTransport() { return new Ship(); }
}

// Usage
Logistics logistics = new SeaLogistics();
logistics.planDelivery("Containers");   // uses a Ship
```

**C#**

```csharp
public abstract class Logistics
{
    // Factory method
    protected abstract ITransport CreateTransport();

    public void PlanDelivery(string cargo)
    {
        ITransport t = CreateTransport();
        t.Deliver(cargo);
    }
}

public class RoadLogistics : Logistics
{
    protected override ITransport CreateTransport() => new Truck();
}

public class SeaLogistics : Logistics
{
    protected override ITransport CreateTransport() => new Ship();
}

// Usage
Logistics logistics = new SeaLogistics();
logistics.PlanDelivery("Containers");
```

**Use it when:**
- The exact type to create is decided at runtime (config, user input, environment).
- Creation is complex or needs to be centralized (caching, pooling, validation).
- You want client code to depend only on an interface (**DIP**).

**Pros:** decouples clients from concrete classes; one place to change creation logic.
**Cons:** a Simple Factory's `switch` must be edited for every new type (a mild OCP violation — acceptable for small, stable sets, or replace with a registry/DI container).

> **Real-world examples:** `Calendar.getInstance()`, `List.of(...)`, `DriverManager.getConnection(...)` in Java; `HttpClientFactory`, `DbProviderFactory`, `ILoggerFactory.CreateLogger()` in .NET.

### 3.3 Builder Pattern

**Family:** Creational

**Intent:** construct a complex object step by step, separating *how it's built* from *what it is*.

**Use it when:**
- A constructor has many parameters, especially optional ones (the "telescoping constructor" problem).
- You want the final object to be **immutable** but still easy to configure.
- You need validation before the object is created.

**❌ The problem:**

**Java**

```java
// Which argument is which? What do null and false mean?
new HttpRequest("https://api.example.com", "POST", null, body, 30, true, false, 3);
```

**C#**

```csharp
// Same problem (named arguments help, but the constructor is still a mess)
new HttpRequest("https://api.example.com", "POST", null, body, 30, true, false, 3);
```

**Java**

```java
public final class HttpRequest {
    private final String url;              // required
    private final String method;           // optional, default GET
    private final Map<String, String> headers;
    private final String body;
    private final int timeoutSeconds;
    private final int retries;

    private HttpRequest(Builder b) {
        this.url = b.url;
        this.method = b.method;
        this.headers = Map.copyOf(b.headers);
        this.body = b.body;
        this.timeoutSeconds = b.timeoutSeconds;
        this.retries = b.retries;
    }

    public static Builder builder(String url) { return new Builder(url); }

    // getters omitted for brevity

    public static final class Builder {
        private final String url;
        private String method = "GET";
        private final Map<String, String> headers = new HashMap<>();
        private String body;
        private int timeoutSeconds = 30;
        private int retries = 0;

        private Builder(String url) { this.url = Objects.requireNonNull(url); }

        public Builder method(String method)           { this.method = method; return this; }
        public Builder header(String key, String value){ headers.put(key, value); return this; }
        public Builder body(String body)               { this.body = body; return this; }
        public Builder timeout(int seconds)            { this.timeoutSeconds = seconds; return this; }
        public Builder retries(int retries)            { this.retries = retries; return this; }

        public HttpRequest build() {
            if ("GET".equals(method) && body != null) {
                throw new IllegalStateException("GET requests cannot have a body");
            }
            if (timeoutSeconds <= 0) {
                throw new IllegalStateException("Timeout must be positive");
            }
            return new HttpRequest(this);
        }
    }
}

// Usage — readable, self-documenting
HttpRequest request = HttpRequest.builder("https://api.example.com/orders")
        .method("POST")
        .header("Content-Type", "application/json")
        .header("Authorization", "Bearer abc123")
        .body("{\"item\":\"book\"}")
        .timeout(10)
        .retries(3)
        .build();
```

> **Java tip:** Lombok's `@Builder` annotation generates this boilerplate for you.

**C#**

```csharp
public sealed class HttpRequest
{
    public string Url { get; }
    public string Method { get; }
    public IReadOnlyDictionary<string, string> Headers { get; }
    public string? Body { get; }
    public int TimeoutSeconds { get; }
    public int Retries { get; }

    private HttpRequest(Builder b)
    {
        Url = b.Url;
        Method = b.Method;
        Headers = new Dictionary<string, string>(b.Headers);
        Body = b.Body;
        TimeoutSeconds = b.TimeoutSeconds;
        Retries = b.Retries;
    }

    public static Builder CreateBuilder(string url) => new(url);

    public sealed class Builder
    {
        internal string Url { get; }
        internal string Method { get; private set; } = "GET";
        internal Dictionary<string, string> Headers { get; } = new();
        internal string? Body { get; private set; }
        internal int TimeoutSeconds { get; private set; } = 30;
        internal int Retries { get; private set; }

        internal Builder(string url) =>
            Url = url ?? throw new ArgumentNullException(nameof(url));

        public Builder WithMethod(string method)              { Method = method; return this; }
        public Builder WithHeader(string key, string value)   { Headers[key] = value; return this; }
        public Builder WithBody(string body)                  { Body = body; return this; }
        public Builder WithTimeout(int seconds)               { TimeoutSeconds = seconds; return this; }
        public Builder WithRetries(int retries)               { Retries = retries; return this; }

        public HttpRequest Build()
        {
            if (Method == "GET" && Body is not null)
                throw new InvalidOperationException("GET requests cannot have a body");
            if (TimeoutSeconds <= 0)
                throw new InvalidOperationException("Timeout must be positive");
            return new HttpRequest(this);
        }
    }
}

// Usage
var request = HttpRequest.CreateBuilder("https://api.example.com/orders")
    .WithMethod("POST")
    .WithHeader("Content-Type", "application/json")
    .WithHeader("Authorization", "Bearer abc123")
    .WithBody("{\"item\":\"book\"}")
    .WithTimeout(10)
    .WithRetries(3)
    .Build();
```

#### Lighter alternative: records

For simple data objects with few fields and little validation, a `record` is often enough.

**Java**

```java
public record HttpRequestOptions(String url, String method, int timeoutSeconds) {
    public HttpRequestOptions {                              // compact constructor: validation
        Objects.requireNonNull(url);
        if (timeoutSeconds <= 0) throw new IllegalArgumentException("timeout");
    }

    public HttpRequestOptions(String url) { this(url, "GET", 30); }   // defaults

    public HttpRequestOptions withTimeout(int seconds) {    // "wither" returns a copy
        return new HttpRequestOptions(url, method, seconds);
    }
}

var opts  = new HttpRequestOptions("https://api.example.com");
var retry = opts.withTimeout(60);
```

**C#**

```csharp
public record HttpRequestOptions
{
    public required string Url { get; init; }
    public string Method { get; init; } = "GET";
    public int TimeoutSeconds { get; init; } = 30;
}

var opts  = new HttpRequestOptions { Url = "https://api.example.com", TimeoutSeconds = 10 };
var retry = opts with { TimeoutSeconds = 60 };              // built-in copy-with
```

C# object initializers and `with` expressions cover more of the Builder's job than Java records do. Reach for a real Builder when you need cross-field validation, step ordering, or a fluent API.

**Pros:** readable construction, immutable results, validation in one place, sensible defaults.
**Cons:** extra code; overkill for objects with only two or three fields.

> **Real-world examples:** `StringBuilder`, `HttpRequest.newBuilder()`, `Stream.builder()` in Java; `StringBuilder`, `WebApplication.CreateBuilder()`, `UriBuilder`, `DbContextOptionsBuilder` in .NET.

---

## 4. Collections

Picking the right collection is one of the easiest performance wins, and getting `equals`/`hashCode` wrong is one of the most common sources of subtle bugs. This section covers both.

### 4.1 Choosing List / Set / Map

There are three basic shapes:

| Shape | Answers the question… | Duplicates? | Typical use |
|---|---|---|---|
| **List** | "What is at position *i*?" / "Keep these in order." | ✅ Allowed | Ordered items, results, history |
| **Set** | "Have I seen this before?" | ❌ Unique | Deduplication, fast membership tests |
| **Map** | "What value belongs to this key?" | Keys unique, values not | Lookups, caches, grouping, counting |

#### Decision guide

```
Do you look things up by a key?
├── Yes → Map / Dictionary
│         ├── Need keys sorted?           → TreeMap            / SortedDictionary
│         ├── Need insertion order?       → LinkedHashMap      / OrderedDictionary (.NET 9+)
│         ├── Shared across threads?      → ConcurrentHashMap  / ConcurrentDictionary
│         └── Otherwise                   → HashMap            / Dictionary
│
└── No → Do you need uniqueness or fast "contains"?
          ├── Yes → Set
          │         ├── Need sorted?          → TreeSet        / SortedSet
          │         ├── Need insertion order? → LinkedHashSet  / (no built-in; combine List + HashSet)
          │         └── Otherwise             → HashSet        / HashSet
          │
          └── No → Sequence
                    ├── FIFO queue / LIFO stack → ArrayDeque    / Queue<T>, Stack<T>
                    ├── Always take smallest?   → PriorityQueue / PriorityQueue<TElement, TPriority>
                    └── Otherwise               → ArrayList     / List<T>
```

#### Java ↔ C# equivalents and complexity

| Java | C# | Order | Lookup / contains | Add | Remove |
|---|---|---|---|---|---|
| `ArrayList` | `List<T>` | Insertion | `get(i)` O(1), `contains` O(n) | O(1) amortized at end | O(n) (shifts elements) |
| `LinkedList` | `LinkedList<T>` | Insertion | `get(i)` O(n) | O(1) at ends | O(1) at a known node |
| `ArrayDeque` | `Queue<T>` / `Stack<T>` | FIFO / LIFO | — | O(1) at ends | O(1) at ends |
| `PriorityQueue` | `PriorityQueue<TElement,TPriority>` | By priority | peek O(1) | O(log n) | poll O(log n) |
| `HashSet` | `HashSet<T>` | None | O(1) avg | O(1) avg | O(1) avg |
| `LinkedHashSet` | — | Insertion | O(1) avg | O(1) avg | O(1) avg |
| `TreeSet` | `SortedSet<T>` | Sorted | O(log n) | O(log n) | O(log n) |
| `HashMap` | `Dictionary<TKey,TValue>` | None | O(1) avg | O(1) avg | O(1) avg |
| `LinkedHashMap` | `OrderedDictionary<TKey,TValue>` (.NET 9+) | Insertion | O(1) avg | O(1) avg | Java O(1); .NET O(n) |
| `TreeMap` | `SortedDictionary<TKey,TValue>` | Sorted by key | O(log n) | O(log n) | O(log n) |
| `ConcurrentHashMap` | `ConcurrentDictionary<TKey,TValue>` | None | O(1) avg | O(1) avg | O(1) avg |

**Practical notes**

- **`ArrayList` / `List<T>` is the right default** for sequences. `LinkedList` almost never wins in practice, because its nodes are scattered in memory (poor CPU cache use) and finding a position is O(n) anyway.
- **"Unordered" means unordered.** Don't rely on `HashMap` or `Dictionary` iteration order. A C# `Dictionary` often *looks* insertion-ordered until you remove something.
- **Declare variables by interface** (`List<String> names = new ArrayList<>()`, `IReadOnlyList<string>` in C# APIs). This is DIP applied to collections: you can change the implementation later.

#### Example: the "contains in a loop" trap

A `List.contains` call scans the whole list. Inside a loop, that turns into O(n·m).

**Java**

```java
List<String> blocked = loadBlockedUsers();          // 100,000 items
for (String user : incomingUsers) {                 // 100,000 items
    if (blocked.contains(user)) { reject(user); }   // ❌ O(n) each → ~10 billion comparisons
}

Set<String> blockedSet = new HashSet<>(blocked);    // ✅ build once: O(n)
for (String user : incomingUsers) {
    if (blockedSet.contains(user)) { reject(user); } // O(1) each
}
```

**C#**

```csharp
List<string> blocked = LoadBlockedUsers();
var blockedSet = new HashSet<string>(blocked);      // ✅ build once

foreach (var user in incomingUsers)
{
    if (blockedSet.Contains(user)) Reject(user);    // O(1) each
}
```

#### Example: counting and grouping with a Map

**Java**

```java
// Count words
Map<String, Integer> counts = new HashMap<>();
for (String word : words) {
    counts.merge(word, 1, Integer::sum);
}

// Group orders by customer
Map<String, List<Order>> byCustomer = new HashMap<>();
for (Order o : orders) {
    byCustomer.computeIfAbsent(o.customerId(), k -> new ArrayList<>()).add(o);
}

// Or with streams
Map<String, List<Order>> grouped = orders.stream()
        .collect(Collectors.groupingBy(Order::customerId));
```

**C#**

```csharp
// Count words
var counts = new Dictionary<string, int>();
foreach (var word in words)
{
    counts.TryGetValue(word, out var c);   // c = 0 if missing
    counts[word] = c + 1;
}

// Group orders by customer with LINQ
Dictionary<string, List<Order>> byCustomer = orders
    .GroupBy(o => o.CustomerId)
    .ToDictionary(g => g.Key, g => g.ToList());

// Case-insensitive keys: pass a comparer
var headers = new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase);
```

#### Don't modify a collection while iterating over it

Both runtimes detect this and throw (`ConcurrentModificationException` in Java, `InvalidOperationException` in C#).

**Java**

```java
for (Session s : sessions) {
    if (s.isExpired()) sessions.remove(s);   // ❌ ConcurrentModificationException
}

sessions.removeIf(Session::isExpired);       // ✅
```

**C#**

```csharp
foreach (var s in sessions)
{
    if (s.IsExpired) sessions.Remove(s);     // ❌ InvalidOperationException
}

sessions.RemoveAll(s => s.IsExpired);        // ✅ (List<T>)
```

### 4.2 How HashMap Works

A hash map gives O(1) average lookup by turning a key into an **array index**. Here is the Java `HashMap` (Java 8+) in detail, followed by how C#'s `Dictionary` differs.

#### Structure

```
table (array of buckets), length is always a power of 2 (default 16)

 index   bucket
 ┌───┐
 │ 0 │ → null
 │ 1 │ → [hash|"cat"|3] → [hash|"act"|7]      ← collision: two keys, same index
 │ 2 │ → null
 │ 3 │ → [hash|"dog"|5]
 │...│
 │15 │ → [hash|"emu"|1]
 └───┘
 Each node stores: hash, key, value, next
```

#### `put(key, value)` step by step

1. **Compute the hash.**
   `h = key.hashCode()`, then `hash = h ^ (h >>> 16)`.
   The XOR mixes the high bits into the low bits, because only the low bits are used for the index.
2. **Find the bucket.**
   `index = hash & (table.length - 1)`.
   Because the length is a power of 2, this bitmask is a fast equivalent of `hash % length`.
3. **Search the bucket.** For each node in the bucket:
   - If `node.hash == hash` **and** (`node.key == key` **or** `key.equals(node.key)`), the key already exists, so **replace the value**.
   - Comparing the cached `hash` first is a cheap filter; `equals` only runs on a likely match.
4. **If no match**, append a new node to the bucket.
5. **Check the load.** If `size > capacity × loadFactor` (default `0.75`, so 12 for capacity 16), **resize**: double the table and redistribute the nodes. Because the size doubles, each node either stays at index `i` or moves to `i + oldCapacity`.

#### `get(key)`

Same steps 1 to 3: compute the hash, go to the bucket, walk it comparing `hash` and then `equals`. Return the value, or `null` if not found.

#### Collisions and treeification

- Different keys can land in the same bucket. They're chained in a **linked list**.
- Since **Java 8**, if a single bucket grows beyond **8 nodes** *and* the table has at least **64 buckets**, that bucket is converted to a **red-black tree**. Worst-case lookup in that bucket drops from O(n) to O(log n). If the table is smaller than 64, it resizes instead.
- When a tree bucket shrinks to **6** nodes or fewer during a resize, it turns back into a list.
- Tree buckets order nodes by hash, then by `compareTo` if the keys are `Comparable`. That's one more reason to make key types `Comparable` when it's natural.

#### Complexity summary

| Case | `get` / `put` |
|---|---|
| Average (good `hashCode`) | O(1) |
| Many collisions, Java 8+ | O(log n) per bucket |
| Many collisions, before Java 8 | O(n) |
| Resize | O(n), but rare, so amortized O(1) |

#### Other facts worth knowing

- `HashMap` allows **one `null` key** (stored in bucket 0) and `null` values. `Hashtable` and `ConcurrentHashMap` allow neither.
- `HashMap` is **not thread-safe**. Use `ConcurrentHashMap` for shared mutable maps.
- **Pre-size** when you know the count, to avoid repeated resizes:

**Java**

```java
Map<String, User> users = HashMap.newHashMap(10_000);   // Java 19+
Map<String, User> users = new HashMap<>(13_334);        // older: expected / 0.75 + 1
```

**C#**

```csharp
var users = new Dictionary<string, User>(capacity: 10_000);  // capacity = number of entries
```

#### How C# `Dictionary<TKey, TValue>` differs

Same idea (hash → bucket → compare with `Equals`), different layout:

| Aspect | Java `HashMap` | C# `Dictionary` |
|---|---|---|
| Storage | Array of buckets pointing to separate node objects | Two arrays: `int[] buckets` + `Entry[] entries` (struct with `hashCode`, `next`, `key`, `value`) |
| Collision chain | Linked nodes (or a tree) | `next` indices *inside* the `entries` array, so fewer allocations and better cache locality |
| Table size | Powers of 2, index via bitmask | Prime numbers, index via modulo |
| Resize trigger | `size > capacity × 0.75` | When the `entries` array is full; grows to the next prime ≥ 2× size |
| Removed slots | Node unlinked and garbage-collected | Slot added to a free list and reused |
| `null` key | Allowed (one) | Throws `ArgumentNullException` |
| Equality source | `key.hashCode()` / `key.equals()` | `IEqualityComparer<TKey>` (default: `GetHashCode()` / `Equals()`) |

> **Note:** in .NET Core and later, `string.GetHashCode()` is **randomized per process**. Never store hash codes in a database or file, or send them over the network. Java's `String.hashCode()` is deterministic, but you still shouldn't rely on it for persistence.

### 4.3 `equals()` and `hashCode()`

Hash-based collections (`HashMap`, `HashSet`, `Dictionary`, `HashSet<T>`) use **two** methods together:

- `hashCode()` / `GetHashCode()` picks the **bucket**.
- `equals()` / `Equals()` confirms the **exact match** inside that bucket.

If the two disagree, lookups silently fail.

#### The contract

1. **If `a.equals(b)` is true, `a.hashCode() == b.hashCode()` must be true.** This is the rule that matters most.
2. Two *unequal* objects **may** share a hash code. That's a collision; it's legal but slows things down.
3. `hashCode()` must return the same value every time **as long as the fields used by `equals` don't change**.
4. `equals` must be:
   - **Reflexive:** `a.equals(a)` is true.
   - **Symmetric:** `a.equals(b)` ⇔ `b.equals(a)`.
   - **Transitive:** if `a.equals(b)` and `b.equals(c)`, then `a.equals(c)`.
   - **Consistent:** repeated calls give the same answer if nothing changed.
   - **Null-safe:** `a.equals(null)` is false and never throws.

> **Default behavior:** if you don't override them, both Java and C# classes use **reference identity**: two objects are equal only if they are the *same instance*. That's often wrong for value-like types (points, money, IDs).

#### Bug #1: overriding `equals` but not `hashCode`

**Java**

```java
class Point {
    final int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }

    @Override
    public boolean equals(Object o) {
        return o instanceof Point p && x == p.x && y == p.y;
    }
    // ❌ hashCode() not overridden: still identity-based
}

Set<Point> set = new HashSet<>();
set.add(new Point(1, 2));

new Point(1, 2).equals(new Point(1, 2));   // true
set.contains(new Point(1, 2));             // ❌ false (almost always)
```

**C#**

```csharp
class Point
{
    public int X { get; }
    public int Y { get; }
    public Point(int x, int y) { X = x; Y = y; }

    public override bool Equals(object? obj) =>
        obj is Point p && X == p.X && Y == p.Y;
    // ❌ GetHashCode() not overridden: still identity-based
    //    (the compiler warns: CS0659)
}

var set = new HashSet<Point> { new Point(1, 2) };

new Point(1, 2).Equals(new Point(1, 2));   // true
set.Contains(new Point(1, 2));             // ❌ false (almost always)
```

The two equal points get different identity hash codes, so `contains` looks in the wrong bucket and never even calls `equals`.

#### Bug #2: mutating a key after inserting it

**Java**

```java
List<Integer> key = new ArrayList<>(List.of(1, 2));
Map<List<Integer>, String> map = new HashMap<>();
map.put(key, "value");

key.add(3);                         // hashCode of key changes

map.get(key);                       // ❌ null: looks in a different bucket
map.get(List.of(1, 2));             // ❌ null: right bucket, but equals fails
map.size();                         // 1: the entry is still there, unreachable
```

**C#**

```csharp
// A record has value equality, but this one has a settable property: a bad key
public record UserKey
{
    public int Id { get; set; }
}

var key  = new UserKey { Id = 1 };
var dict = new Dictionary<UserKey, string> { [key] = "value" };

key.Id = 2;                                   // hash code of key changes

dict.ContainsKey(key);                        // ❌ false: hash no longer matches
dict.ContainsKey(new UserKey { Id = 1 });     // ❌ false: hash matches, Equals fails
dict.Count;                                   // 1: the entry is still there, unreachable
```

> In C#, `List<T>` and arrays use **reference** equality, so mutating them doesn't break a dictionary the way it does in Java. The bug appears with any key type that has **value equality and mutable fields**.

**Rule:** use **immutable** objects as map keys and set elements (`String`, `Integer`, enums, records, or your own immutable classes).

#### Correct implementation

**Java (manual)**

```java
public final class Point {
    private final int x;
    private final int y;

    public Point(int x, int y) { this.x = x; this.y = y; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;                    // fast path
        if (!(o instanceof Point other)) return false; // handles null too
        return x == other.x && y == other.y;
    }

    @Override
    public int hashCode() {
        return Objects.hash(x, y);                     // use the SAME fields as equals
    }
}
```

**Java (record: generated for you)**

```java
public record Point(int x, int y) { }   // equals, hashCode, toString all generated
```

**C# (manual)**

```csharp
public sealed class Point : IEquatable<Point>
{
    public int X { get; }
    public int Y { get; }

    public Point(int x, int y) { X = x; Y = y; }

    // Strongly typed: no boxing or casting, used by Dictionary/HashSet
    public bool Equals(Point? other) =>
        other is not null && X == other.X && Y == other.Y;

    public override bool Equals(object? obj) => Equals(obj as Point);

    public override int GetHashCode() => HashCode.Combine(X, Y);

    // Optional, but keeps == consistent with Equals
    public static bool operator ==(Point? a, Point? b) => Equals(a, b);
    public static bool operator !=(Point? a, Point? b) => !Equals(a, b);
}
```

**C# (record: generated for you)**

```csharp
public record Point(int X, int Y);          // value equality, GetHashCode, ==, != generated
public readonly record struct Money(decimal Amount, string Currency);
```

#### Common pitfalls

| Pitfall | Why it breaks | Fix |
|---|---|---|
| Overriding only `equals` | Equal objects land in different buckets | Always override both, using the **same fields** |
| Mutable keys | Hash changes after insertion; the entry becomes unreachable | Use immutable keys (records, `final`/`readonly` fields) |
| Arrays as keys | Arrays use reference equality in both Java and C# | Java: use `List` or wrap with `Arrays.equals`/`Arrays.hashCode`. C#: use a custom `IEqualityComparer<T[]>` |
| `==` on objects in Java | `==` compares references, not values | Use `.equals()` (or `Objects.equals(a, b)` for null safety) |
| `instanceof` equality across a class hierarchy | `Point.equals(ColorPoint)` ≠ `ColorPoint.equals(Point)` breaks symmetry | Make value classes `final`/`sealed`, or compare `getClass()` / `GetType()` |
| Constant `hashCode()` (e.g. `return 1;`) | Technically legal, but every key collides, so O(1) becomes O(n) or O(log n) | Combine all significant fields |
| Sorted collections | `TreeMap`/`SortedDictionary` use `compareTo`/`IComparer`, **not** `hashCode` | Keep `compareTo` consistent with `equals` (return 0 only when equal) |

---

## 5. Streams, Optional & Exceptions

### 5.1 Stream API (Java) / LINQ (C#)

Both let you process collections **declaratively**: you describe *what* you want (filter, transform, group) instead of writing the loop by hand.

A pipeline has three parts:

```
source  →  intermediate operations (lazy)  →  terminal operation (runs everything)
list       filter / map / sorted / ...        toList / count / sum / forEach / ...
```

#### Operation mapping

| Purpose | Java Stream | C# LINQ |
|---|---|---|
| Keep matching items | `filter` | `Where` |
| Transform each item | `map` | `Select` |
| Flatten nested collections | `flatMap` | `SelectMany` |
| Sort | `sorted(Comparator...)` | `OrderBy` / `OrderByDescending` / `ThenBy` |
| Remove duplicates | `distinct` | `Distinct` |
| Take / skip first *n* | `limit` / `skip` | `Take` / `Skip` |
| Any / all match | `anyMatch` / `allMatch` | `Any` / `All` |
| First item | `findFirst()` (returns `Optional`) | `First` / `FirstOrDefault` |
| Count / sum / average | `count`, `mapToInt(...).sum()`, `average()` | `Count`, `Sum`, `Average` |
| Fold into one value | `reduce` | `Aggregate` |
| To list | `toList()` (Java 16+) | `ToList()` |
| To map | `Collectors.toMap` | `ToDictionary` |
| Group | `Collectors.groupingBy` | `GroupBy` / `ToLookup` |
| Join strings | `Collectors.joining(", ")` | `string.Join(", ", ...)` |

#### Example: top 3 customers by total paid amount

**Java**

```java
record Order(String customer, double amount, Status status) {}

List<String> top3 = orders.stream()
        .filter(o -> o.status() == Status.PAID)
        .collect(Collectors.groupingBy(
                Order::customer,
                Collectors.summingDouble(Order::amount)))     // Map<String, Double>
        .entrySet().stream()
        .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
        .limit(3)
        .map(Map.Entry::getKey)
        .toList();
```

**C#**

```csharp
record Order(string Customer, decimal Amount, Status Status);

List<string> top3 = orders
    .Where(o => o.Status == Status.Paid)
    .GroupBy(o => o.Customer)
    .Select(g => new { Customer = g.Key, Total = g.Sum(o => o.Amount) })
    .OrderByDescending(x => x.Total)
    .Take(3)
    .Select(x => x.Customer)
    .ToList();
```

#### Example: flatten, distinct, join

**Java**

```java
// All distinct tags across all posts, sorted, as "a, b, c"
String tags = posts.stream()
        .flatMap(p -> p.tags().stream())
        .distinct()
        .sorted()
        .collect(Collectors.joining(", "));
```

**C#**

```csharp
string tags = string.Join(", ", posts
    .SelectMany(p => p.Tags)
    .Distinct()
    .Order());                 // .NET 7+; older: .OrderBy(t => t)
```

#### Laziness and reuse

Intermediate operations do nothing until a terminal operation runs. The two platforms then differ in what happens when you use the pipeline again.

**Java**: a stream can be consumed **only once**.

```java
Stream<String> active = users.stream().filter(User::isActive).map(User::name);

long n = active.count();     // runs the pipeline
active.toList();             // ❌ IllegalStateException: stream has already been operated upon
```

**C#**: a LINQ query **re-runs every time** you enumerate it (deferred execution).

```csharp
IEnumerable<string> active = users.Where(u => u.IsActive).Select(u => u.Name);

users.Add(new User("Zoe", isActive: true));
int n = active.Count();      // runs the query now, so it includes Zoe
var list = active.ToList();  // runs it AGAIN; for expensive queries, materialize once
```

> **EF Core / `IQueryable`:** LINQ on a `DbSet` is translated to SQL. Calling `.ToList()` too early loads the whole table into memory: `db.Orders.ToList().Where(...)` ❌ vs. `db.Orders.Where(...).ToList()` ✅.

#### Common pitfalls

**1. Side effects inside the pipeline**

**Java**

```java
List<String> names = new ArrayList<>();
users.stream().filter(User::isActive).forEach(u -> names.add(u.name()));   // ❌ mutating outside state

List<String> names = users.stream().filter(User::isActive).map(User::name).toList();  // ✅
```

**C#**

```csharp
var names = new List<string>();
users.Where(u => u.IsActive).ToList().ForEach(u => names.Add(u.Name));     // ❌

var names = users.Where(u => u.IsActive).Select(u => u.Name).ToList();     // ✅
```

**2. Duplicate keys when building a map**

**Java**

```java
// ❌ IllegalStateException if two users share an email
Map<String, User> byEmail = users.stream()
        .collect(Collectors.toMap(User::email, u -> u));

// ✅ decide what happens on a duplicate: keep the first one
Map<String, User> byEmail = users.stream()
        .collect(Collectors.toMap(User::email, u -> u, (first, second) -> first));
```

**C#**

```csharp
// ❌ ArgumentException if two users share an email
var byEmail = users.ToDictionary(u => u.Email);

// ✅ keep the first one per email
var byEmail = users
    .GroupBy(u => u.Email)
    .ToDictionary(g => g.Key, g => g.First());

// or, .NET 6+
var byEmail = users.DistinctBy(u => u.Email).ToDictionary(u => u.Email);
```

**3. Default values that hide "not found"**

**Java**

```java
Optional<Integer> big = numbers.stream().filter(n -> n > 100).findFirst();  // ✅ empty if none
int value = big.orElse(-1);
```

**C#**

```csharp
int big = numbers.FirstOrDefault(n => n > 100);       // ❌ returns 0 if none: is 0 a real value?

int big = numbers.FirstOrDefault(n => n > 100, -1);   // ✅ explicit default (.NET 6+)
int? big = numbers.Where(n => n > 100)
                  .Cast<int?>().FirstOrDefault();     // ✅ null if none
```

**4. Parallel isn't free**

**Java**

```java
long count = hugeList.parallelStream().filter(this::isExpensiveCheck).count();
```

**C#**

```csharp
int count = hugeList.AsParallel().Count(IsExpensiveCheck);
```

Use these only for large, CPU-heavy work with no shared mutable state. For small lists, the overhead makes them slower, and they don't help with I/O-bound work.

**5. Readability**

If a pipeline needs a comment to explain it, or has nested lambdas several levels deep, a plain `for`/`foreach` loop is often clearer. Streams and LINQ are a tool, not a rule.

### 5.2 Optional (Java) / Nullable (C#)

The problem is the same on both platforms: **"this value might be missing"** should be visible in the type, so callers can't forget to handle it.

- **Java** uses `Optional<T>` as a return type.
- **C#** uses **nullable reference types** (`string?`, C# 8+ with `<Nullable>enable</Nullable>`), `Nullable<T>` for value types (`int?`), and null operators (`?.`, `??`, `??=`). The compiler warns when you might dereference a null.

#### Concept mapping

| Intent | Java `Optional` | C# nullable |
|---|---|---|
| "May be absent" return type | `Optional<User>` | `User?` |
| Nothing | `Optional.empty()` | `null` |
| Wrap a possibly-null value | `Optional.ofNullable(x)` | just `x` (type `T?`) |
| Transform if present | `opt.map(User::name)` | `user?.Name` |
| Chain optional lookups | `opt.flatMap(User::address)` | `user?.Address?.City` |
| Default value | `opt.orElse("n/a")` | `x ?? "n/a"` |
| Lazily computed default | `opt.orElseGet(this::compute)` | `x ?? Compute()` (already lazy) |
| Throw if missing | `opt.orElseThrow(() -> new NotFound(...))` | `x ?? throw new NotFoundException(...)` |
| Do something if present | `opt.ifPresent(this::send)` | `if (x is not null) Send(x);` |
| Primitive / value type | `OptionalInt`, `OptionalDouble` | `int?`, `double?` |

#### Example: find a user's city, or "Unknown"

**Java**

```java
public interface UserRepository {
    Optional<User> findById(long id);          // the signature says "may be missing"
}

public record User(String name, Optional<Address> address) {}   // see note on fields below
public record Address(String city) {}

String city = userRepo.findById(42)
        .flatMap(User::address)
        .map(Address::city)
        .orElse("Unknown");
```

**C#**

```csharp
public interface IUserRepository
{
    User? FindById(long id);                   // the signature says "may be missing"
}

public record User(string Name, Address? Address);
public record Address(string City);

string city = userRepo.FindById(42)?.Address?.City ?? "Unknown";
```

#### Example: "not found" should be an error

**Java**

```java
User user = userRepo.findById(id)
        .orElseThrow(() -> new UserNotFoundException(id));
```

**C#**

```csharp
User user = userRepo.FindById(id)
    ?? throw new UserNotFoundException(id);
```

#### Do's and don'ts

**1. Don't treat it as a fancy null check**

**Java**

```java
Optional<User> opt = repo.findById(id);
if (opt.isPresent()) {                    // ❌ just a null check with extra steps
    send(opt.get());
}
User u = opt.get();                       // ❌ throws NoSuchElementException if empty

opt.ifPresent(this::send);                // ✅
User u = opt.orElseThrow();               // ✅ same failure, but the intent is explicit
```

**C#**

```csharp
User? user = repo.FindById(id);
Send(user!);                              // ❌ "!" silences the warning; NullReferenceException at runtime

if (user is not null) Send(user);         // ✅
if (repo.FindById(id) is { } u) Send(u);  // ✅ pattern: non-null and assigned in one step
```

**2. `orElse` evaluates its argument eagerly**

**Java**

```java
String name = findName(id).orElse(loadDefaultFromDb());       // ❌ DB call ALWAYS runs
String name = findName(id).orElseGet(this::loadDefaultFromDb); // ✅ only runs when empty
```

**C#**

```csharp
string name = FindName(id) ?? LoadDefaultFromDb();   // ✅ ?? only evaluates the right side when needed
```

**3. Use it for return values, not everywhere**

**Java**

| ✅ Good use | ❌ Avoid |
|---|---|
| Method return type | Fields (not `Serializable`; use a nullable field plus an `Optional` getter) |
| End of a stream (`findFirst`, `max`) | Method parameters (use overloads or `@Nullable`) |
| | `Optional<List<T>>`: return an **empty list** instead |
| | Returning `null` from a method declared as `Optional` |

(The `User` record above uses an `Optional` field to keep the example short. In production code, store a nullable `Address` and expose `Optional<Address> address()` as a method.)

**C#**

| ✅ Good use | ❌ Avoid |
|---|---|
| Turn on `<Nullable>enable</Nullable>` project-wide | Scattering `!` to silence warnings |
| `T?` on returns, fields and parameters that really can be null | `List<T>?`: return an **empty list** instead |
| `TryXxx(out T value)` pattern (`int.TryParse`, `TryGetValue`) | Treating warnings as noise: they're the whole point |

**4. Validate non-null parameters at the boundary**

**Java**

```java
public Order(String customerId, List<Item> items) {
    this.customerId = Objects.requireNonNull(customerId, "customerId");
    this.items = List.copyOf(items);      // also throws on null
}
```

**C#**

```csharp
public Order(string customerId, List<Item> items)
{
    ArgumentNullException.ThrowIfNull(customerId);   // .NET 6+
    ArgumentNullException.ThrowIfNull(items);
    CustomerId = customerId;
    Items = items.ToList();
}
```

### 5.3 Handling Exceptions Properly

#### Exception types at a glance

| | Java | C# |
|---|---|---|
| Root | `Throwable` | `Exception` |
| Unrecoverable (don't catch) | `Error` (`OutOfMemoryError`, `StackOverflowError`) | `OutOfMemoryException`, `StackOverflowException` |
| Checked (must declare or catch) | `Exception` subclasses, except `RuntimeException` (`IOException`, `SQLException`) | **None.** C# has no checked exceptions |
| Unchecked | `RuntimeException` and subclasses | Everything |

#### Which exception to throw

| Situation | Java | C# |
|---|---|---|
| Invalid argument | `IllegalArgumentException` | `ArgumentException`, `ArgumentOutOfRangeException` |
| Null argument | `NullPointerException` (via `Objects.requireNonNull`) | `ArgumentNullException` |
| Object in the wrong state for this call | `IllegalStateException` | `InvalidOperationException` |
| Operation not supported | `UnsupportedOperationException` | `NotSupportedException` |
| Timeout | `TimeoutException` | `TimeoutException` |
| Domain error (order not found, insufficient stock) | Your own, e.g. `OrderNotFoundException` | Your own, e.g. `OrderNotFoundException` |

Never throw the bare `Exception`, `RuntimeException` or `Throwable` types. Callers can't catch them selectively.

#### Rule 1: Throw early, catch late

Validate inputs **as early as possible**, so the error points at the real cause. Catch only **where you can actually do something**: retry, fall back, or turn it into a user-facing response.

**Java**

```java
public void transfer(Account from, Account to, BigDecimal amount) {
    Objects.requireNonNull(from, "from");
    Objects.requireNonNull(to, "to");
    if (amount.signum() <= 0) {
        throw new IllegalArgumentException("amount must be > 0, was " + amount);
    }
    if (from.balance().compareTo(amount) < 0) {
        throw new InsufficientFundsException(from.id(), amount);
    }
    // ...
}
```

**C#**

```csharp
public void Transfer(Account from, Account to, decimal amount)
{
    ArgumentNullException.ThrowIfNull(from);
    ArgumentNullException.ThrowIfNull(to);
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(amount);   // .NET 8+

    if (from.Balance < amount)
        throw new InsufficientFundsException(from.Id, amount);
    // ...
}
```

#### Rule 2: Never swallow exceptions

**Java**

```java
try {
    sendEmail(user);
} catch (Exception e) {
    // ❌ silence: the bug is now invisible
}
```

**C#**

```csharp
try
{
    SendEmail(user);
}
catch (Exception)
{
    // ❌ silence: the bug is now invisible
}
```

If ignoring the error really is correct, catch the **specific** exception, and log it or leave a comment explaining why.

#### Rule 3: Catch specific exceptions, most specific first

**Java**

```java
public Config loadConfig(Path path) {
    try (BufferedReader reader = Files.newBufferedReader(path)) {   // closed automatically
        return parse(reader);
    } catch (NoSuchFileException e) {          // subclass of IOException: must come first
        return Config.defaults();              // handled: we know what to do
    } catch (IOException e) {
        throw new ConfigLoadException("Cannot read config from " + path, e);
    }
}

// Several unrelated types, same handling: multi-catch
catch (SQLException | TimeoutException e) { ... }
```

**C#**

```csharp
public Config LoadConfig(string path)
{
    try
    {
        using var reader = new StreamReader(path);       // disposed automatically
        return Parse(reader);
    }
    catch (FileNotFoundException)                        // subclass of IOException: must come first
    {
        return Config.Defaults();
    }
    catch (IOException ex)
    {
        throw new ConfigLoadException($"Cannot read config from {path}", ex);
    }
}

// Exception filters: catch only when a condition holds
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound) { ... }
```

Catch the broad `Exception` type only at the **top-level boundary** (see Rule 7).

#### Rule 4: Preserve the cause and stack trace

**Java**

```java
catch (IOException e) {
    throw new ConfigLoadException("Cannot read config");      // ❌ original cause lost
}

catch (IOException e) {
    throw new ConfigLoadException("Cannot read config from " + path, e);   // ✅ cause chained
}

public class ConfigLoadException extends RuntimeException {
    public ConfigLoadException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

**C#**

```csharp
catch (IOException ex)
{
    throw new ConfigLoadException("Cannot read config");         // ❌ inner exception lost
}

catch (IOException ex)
{
    throw new ConfigLoadException($"Cannot read config from {path}", ex);  // ✅ InnerException set
}

catch (Exception ex)
{
    _logger.LogError(ex, "Failed");
    throw ex;     // ❌ resets the stack trace to this line
    throw;        // ✅ rethrows with the original stack trace
}

public class ConfigLoadException : Exception
{
    public ConfigLoadException(string message, Exception inner) : base(message, inner) { }
}
```

#### Rule 5: Always release resources

**Java**: `try`-with-resources works with anything that implements `AutoCloseable`.

```java
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql)) {
    // ...
}   // ps.close() then conn.close(), even if an exception is thrown
```

**C#**: `using` works with anything that implements `IDisposable` (`await using` for `IAsyncDisposable`).

```csharp
using var conn = new SqlConnection(connectionString);
using var cmd  = new SqlCommand(sql, conn);
// ...
// cmd.Dispose() then conn.Dispose() at the end of the scope, even on exceptions
```

Never `return` from a `finally` block. In Java, it silently discards any exception in flight. C# doesn't allow it (compile error CS0157).

**Java**

```java
try {
    throw new RuntimeException("boom");
} finally {
    return 42;     // ❌ "boom" disappears
}
```

**C#**

```csharp
try
{
    throw new Exception("boom");
}
finally
{
    return 42;     // ❌ won't compile: CS0157
}
```

#### Rule 6: Don't use exceptions for normal control flow

Exceptions are slow (they capture a stack trace) and hide intent. If "failure" is an expected outcome, model it in the return type.

**Java**

```java
// ❌ an exception as an if-statement
int port;
try { port = Integer.parseInt(input); }
catch (NumberFormatException e) { port = 8080; }

// ✅ Java has no TryParse, so wrap it once in a helper
static OptionalInt tryParseInt(String s) {
    try { return OptionalInt.of(Integer.parseInt(s)); }
    catch (NumberFormatException e) { return OptionalInt.empty(); }
}
int port = tryParseInt(input).orElse(8080);
```

**C#**

```csharp
// ❌
int port;
try { port = int.Parse(input); }
catch (FormatException) { port = 8080; }

// ✅ built-in Try pattern
int port = int.TryParse(input, out var p) ? p : 8080;
```

#### Rule 7: Handle (and log) once, at the boundary

Avoid catch → log → rethrow in every layer; you get the same error logged five times. Let exceptions bubble up to **one global handler** that logs them and maps them to a response.

**Java (Spring Boot)**

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<ErrorDto> notFound(OrderNotFoundException e) {
        return ResponseEntity.status(404).body(new ErrorDto(e.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorDto> unexpected(Exception e) {
        log.error("Unhandled error", e);                 // logged once, with stack trace
        return ResponseEntity.status(500).body(new ErrorDto("Internal error"));
    }
}
```

**C# (ASP.NET Core 8+)**

```csharp
public class GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext ctx, Exception ex, CancellationToken ct)
    {
        var (status, message) = ex switch
        {
            OrderNotFoundException => (404, ex.Message),
            _                      => (500, "Internal error")
        };

        if (status == 500)
            logger.LogError(ex, "Unhandled error");      // logged once, with stack trace

        ctx.Response.StatusCode = status;
        await ctx.Response.WriteAsJsonAsync(new { error = message }, ct);
        return true;
    }
}

// Program.cs
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();
app.UseExceptionHandler();
```

Never expose stack traces or internal messages to end users. Log the details, and return a generic message.

#### Rule 8: Platform-specific traps

**Java: checked exceptions inside lambdas.** Stream lambdas can't throw checked exceptions, so wrap them:

```java
List<String> contents = paths.stream()
        .map(p -> {
            try { return Files.readString(p); }
            catch (IOException e) { throw new UncheckedIOException(e); }
        })
        .toList();
```

C# has no checked exceptions, so the same lambda just works:

```csharp
List<string> contents = paths.Select(File.ReadAllText).ToList();
```

**Java: don't swallow `InterruptedException`.** Restore the interrupt flag so the thread's owner knows it was asked to stop:

```java
try {
    Thread.sleep(1_000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();    // ✅ restore the flag
    return;                                // or throw a domain exception
}
```

The C# equivalent is cooperative cancellation with `CancellationToken`. Let `OperationCanceledException` propagate rather than swallowing it:

```csharp
try
{
    await Task.Delay(1_000, cancellationToken);
}
catch (OperationCanceledException)
{
    // ✅ clean up if needed, then rethrow or return; don't treat it as an error
    throw;
}
```

**C#: avoid `async void`.** Its exceptions can't be caught by the caller and can crash the process. Use `async Task` (except for UI event handlers).

```csharp
public async void SaveAsync() { ... }   // ❌ caller can't await or catch
public async Task SaveAsync() { ... }   // ✅

try { await SaveAsync(); }              // exception surfaces here
catch (IOException ex) { ... }
```

Java's equivalent trap is a `CompletableFuture` whose failure nobody observes. Always attach `exceptionally` / `handle`, or call `join()`:

```java
CompletableFuture.runAsync(this::save);                     // ❌ failure silently lost

CompletableFuture.runAsync(this::save)
        .exceptionally(ex -> { log.error("Save failed", ex); return null; });   // ✅
```

---

## 6. Modern Java (17 & 21)

Java 17 (2021) and Java 21 (2023) are **LTS** (long-term support) releases. Together they changed how everyday Java code looks: less boilerplate, data modelled as plain values, and simple blocking code that still scales. Each feature below comes with its closest C# equivalent.

#### When each feature became final

| Feature | Preview in | Final in | First LTS with it |
|---|---|---|---|
| Switch expressions (`->`, `yield`) | 12, 13 | **14** | 17 |
| Records | 14, 15 | **16** | 17 |
| Pattern matching for `instanceof` | 14, 15 | **16** | 17 |
| Sealed classes | 15, 16 | **17** | 17 |
| Pattern matching for `switch` | 17 – 20 | **21** | 21 |
| Record patterns | 19, 20 | **21** | 21 |
| Virtual threads | 19, 20 | **21** | 21 |
| Unnamed variables & patterns (`_`) | 21 | **22** | 25 |

### 6.1 Records

A **record** is a class whose job is to carry data. You declare the components once, and the compiler generates the rest:

- a `private final` field for each component,
- a constructor taking all of them (the *canonical constructor*),
- accessors named after the components (`amount()`, **not** `getAmount()`),
- `equals`, `hashCode` and `toString` based on all components.

#### Example: a `Money` value object

**Java**

```java
public record Money(BigDecimal amount, String currency) {

    public Money {                                    // compact constructor: validate / normalize
        Objects.requireNonNull(amount, "amount");
        Objects.requireNonNull(currency, "currency");
        if (amount.signum() < 0) throw new IllegalArgumentException("amount < 0");
    }

    public static Money zero(String currency) {       // static factories are allowed
        return new Money(BigDecimal.ZERO, currency);
    }

    public Money plus(Money other) {                  // so are instance methods
        if (!currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch");
        }
        return new Money(amount.add(other.amount), currency);
    }
}

var a = new Money(new BigDecimal("10.00"), "USD");
a.amount();                                              // 10.00
a.equals(new Money(new BigDecimal("10.00"), "USD"));     // true
System.out.println(a);                                   // Money[amount=10.00, currency=USD]

// ⚠️ BigDecimal.equals also compares scale: 10.00 and 10.0 are NOT equal.
//    Normalize in the compact constructor (e.g. amount.setScale(2)) if that matters.
```

**C#**

```csharp
public record Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        ArgumentOutOfRangeException.ThrowIfNegative(amount);   // .NET 8+
        ArgumentException.ThrowIfNullOrEmpty(currency);        // .NET 7+
        Amount = amount;
        Currency = currency;
    }

    public static Money Zero(string currency) => new(0m, currency);

    public Money Plus(Money other) =>
        Currency == other.Currency
            ? new Money(Amount + other.Amount, Currency)
            : throw new ArgumentException("Currency mismatch");
}

var a = new Money(10.00m, "USD");
a.Amount;                                // 10.00
a == new Money(10.00m, "USD");           // true: records generate == and !=
Console.WriteLine(a);                    // Money { Amount = 10.00, Currency = USD }
```

> **Why the C# version isn't positional:** in a positional record (`record Money(decimal Amount, string Currency)`), a `with` expression copies the object and sets properties directly, so **it skips constructor validation**: `a with { Amount = -5 }` would succeed. With get-only properties and an explicit constructor, that line doesn't compile.

#### Example: records that hold collections

Records are only **shallowly** immutable. A `List` component can still be changed from outside.

**Java**

```java
public record Order(String id, List<Item> items) {
    public Order {
        items = List.copyOf(items);          // defensive, unmodifiable copy
    }
}
// equals compares items with List.equals → element by element ✅
```

**C#**

```csharp
public record Order
{
    public string Id { get; }
    public IReadOnlyList<Item> Items { get; }

    public Order(string id, IEnumerable<Item> items)
    {
        Id = id;
        Items = items.ToList().AsReadOnly();  // defensive, read-only copy
    }
}
// ⚠️ the generated Equals compares Items by REFERENCE, not element by element.
//    Override Equals/GetHashCode if two orders with the same items must be equal.
```

#### Java records vs. C# records

| | Java `record` | C# `record` |
|---|---|---|
| Kind | Always a class, implicitly `final` | `record` / `record class` (reference type) or `record struct` (value type) |
| Fields | Always `private final` | Positional properties are `init`-only; you can add mutable ones (but shouldn't) |
| Inheritance | Can't extend another class; can implement interfaces | Can inherit from another record |
| Accessors | `money.amount()` | `money.Amount` |
| Copy with changes | Not built in: write `withX(...)` methods | `money with { Amount = 5 }` |
| Deconstruction | Record patterns (Java 21), see 6.3 | `var (amount, currency) = money;` (positional records) |
| Collection fields in `equals` | Value comparison (`List.equals`) | Reference comparison |

**Use records for:** DTOs, API requests and responses, value objects (`Money`, `Email`, `DateRange`), map keys, results returned from methods.
**Don't use them for:** JPA/Hibernate entities (they need a no-arg constructor and mutable state), or objects with identity and a lifecycle.

### 6.2 Sealed Classes

A **sealed** class or interface lists exactly which classes may extend it. The set of subtypes is **closed** and known to the compiler.

**Java**

```java
public sealed interface PaymentResult
        permits Approved, Declined, Pending { }

public record Approved(String transactionId) implements PaymentResult { }
public record Declined(String reason)        implements PaymentResult { }
public record Pending(Duration retryAfter)   implements PaymentResult { }
```

Every permitted subclass must say how it continues the hierarchy:

| Modifier | Meaning |
|---|---|
| `final` | No further subclasses (records are implicitly `final`) |
| `sealed ... permits ...` | Another closed level |
| `non-sealed` | Open again: anyone may extend it |

```java
public sealed abstract class Vehicle permits Car, Truck, Bike { }
public final class Car extends Vehicle { }
public sealed class Truck extends Vehicle permits DumpTruck { }
public final class DumpTruck extends Truck { }
public non-sealed class Bike extends Vehicle { }       // anyone can extend Bike
```

The permitted subclasses must be in the same module, or in the same package if you don't use modules. If they're all in the same file, you can leave out `permits`.

**C#**

In C#, `sealed` means something else: **no** subclasses at all (like Java's `final`). There's no direct "only these subclasses" keyword. The usual approximation is an abstract record with a **private constructor** and nested subtypes:

```csharp
public abstract record PaymentResult
{
    private PaymentResult() { }          // only nested types can call this

    public sealed record Approved(string TransactionId) : PaymentResult;
    public sealed record Declined(string Reason)        : PaymentResult;
    public sealed record Pending(TimeSpan RetryAfter)   : PaymentResult;
}
```

#### Why sealed matters: exhaustive `switch`

Because the compiler knows every subtype, a `switch` over a sealed type needs **no `default`**. And if you add a new subtype, every `switch` that doesn't handle it **stops compiling**.

**Java**

```java
String message(PaymentResult result) {
    return switch (result) {
        case Approved a -> "Paid. Transaction " + a.transactionId();
        case Declined d -> "Declined: " + d.reason();
        case Pending p  -> "Retry in " + p.retryAfter().toSeconds() + "s";
        // no default: add `Refunded` to `permits` and this switch fails to compile ✅
    };
}
```

**C#**

```csharp
string Message(PaymentResult result) => result switch
{
    PaymentResult.Approved a => $"Paid. Transaction {a.TransactionId}",
    PaymentResult.Declined d => $"Declined: {d.Reason}",
    PaymentResult.Pending p  => $"Retry in {p.RetryAfter.TotalSeconds}s",
    _ => throw new UnreachableException()   // .NET 7+. C# can't prove the list is complete,
                                            // so without this you get warning CS8509
};
```

> **Sealed types vs. the Open/Closed Principle.** These are opposite trade-offs, and both are useful:
>
> - **Interface + polymorphism** (section 2.2): adding a new *type* is easy (one new class). Adding a new *operation* means editing every class.
> - **Sealed type + `switch`**: adding a new *operation* is easy (one new `switch`). Adding a new *type* means updating every `switch`, and the compiler finds them all for you.
>
> Use sealed hierarchies for **closed sets of cases**: results, events, commands, AST nodes, states.

> **Link to exceptions (5.3):** a sealed result type like `PaymentResult` is a good way to model *expected* outcomes ("declined" is a normal business result, not an exception).

### 6.3 Pattern Matching

Pattern matching lets you **test a value's shape and extract parts of it in one step**.

#### `instanceof` with a binding (Java 16)

**Java**

```java
// Before
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.length());
}

// Java 16+
if (obj instanceof String s && !s.isBlank()) {
    System.out.println(s.length());
}
```

**C#**

```csharp
if (obj is string s && s.Length > 0)
{
    Console.WriteLine(s.Length);
}
```

#### `switch` with type patterns, guards and `null` (Java 21)

**Java**

```java
static String describe(Object o) {
    return switch (o) {
        case null                       -> "null";            // no more NPE from switch
        case Integer i when i > 100     -> "big int " + i;    // guard
        case Integer i                  -> "int " + i;
        case String s when s.isEmpty()  -> "empty string";
        case String s                   -> "string of length " + s.length();
        case int[] arr                  -> "int array of " + arr.length;
        default                         -> "something else";
    };
}
```

**C#**

```csharp
static string Describe(object? o) => o switch
{
    null                 => "null",
    int i and > 100      => $"big int {i}",          // or: int i when i > 100
    int i                => $"int {i}",
    string { Length: 0 } => "empty string",          // property pattern
    string s             => $"string of length {s.Length}",
    int[] arr            => $"int array of {arr.Length}",
    _                    => "something else"
};
```

**Order matters in both languages.** Put the more specific case (with a guard) before the general one. Java and C# both give a compile error if an earlier case already covers a later one.

#### Record patterns: deconstruct in the pattern (Java 21)

**Java**

```java
record Point(int x, int y) { }
record Line(Point start, Point end) { }

static String classify(Object o) {
    return switch (o) {
        case Point(int x, int y) when x == 0 && y == 0 -> "origin";
        case Point(int x, int y) -> "point at " + x + "," + y;
        case Line(Point(var x1, var y1), Point(var x2, var y2)) when x1 == x2 -> "vertical line";
        case Line l -> "line";
        default -> "unknown";
    };
}

// Also works with instanceof
if (o instanceof Point(int x, int y)) {
    System.out.println(x + y);
}
```

Java record patterns can't contain constants (`Point(0, 0)` isn't allowed), so use a `when` guard. Java 22+ adds `_` for parts you don't need: `Line(Point(var x1, _), Point(var x2, _))`.

**C#** (positional patterns use the record's generated `Deconstruct` method):

```csharp
record Point(int X, int Y);
record Line(Point Start, Point End);

static string Classify(object o) => o switch
{
    Point(0, 0)                                     => "origin",   // constants allowed
    Point(var x, var y)                             => $"point at {x},{y}",
    Line(Point(var x1, _), Point(var x2, _)) when x1 == x2 => "vertical line",
    Line                                            => "line",
    _                                               => "unknown"
};

if (o is Point(var x, var y))
{
    Console.WriteLine(x + y);
}
```

#### Matching on properties and ranges

C# has more pattern kinds (property, relational, `and`/`or`/`not`, list patterns). In Java, you express the same thing with `when` guards.

**Java**

```java
String category = switch (order) {
    case Order o when o.total() > 1000 && o.customer().isVip() -> "VIP big order";
    case Order o when o.total() > 1000                          -> "big order";
    case Order o when o.items().isEmpty()                       -> "empty";
    case Order o                                                -> "normal";
};

// int score: primitive patterns are still preview, so box to Integer and use guards
String grade = switch (Integer.valueOf(score)) {
    case Integer s when s >= 90 -> "A";
    case Integer s when s >= 75 -> "B";
    default                     -> "C";
};
```

**C#**

```csharp
string category = order switch
{
    { Total: > 1000, Customer.IsVip: true } => "VIP big order",   // C# 10 extended property pattern
    { Total: > 1000 }                        => "big order",
    { Items.Count: 0 }                       => "empty",
    _                                        => "normal"
};

string grade = score switch
{
    >= 90 => "A",                               // relational patterns (C# 9)
    >= 75 => "B",
    _     => "C"
};

bool ok = numbers is [1, .., > 0];              // list pattern (C# 11): starts with 1, last > 0
```

### 6.4 Virtual Threads (Concept)

#### The problem they solve

A classic Java server uses **one thread per request**. Each Java thread used to be a **platform thread**, a thin wrapper around an operating-system thread:

- expensive to create, with a large stack (often around 1 MB reserved),
- limited to a few thousand per machine.

Most requests spend their time **waiting**: on the database, on another HTTP service, on a file. A waiting thread does nothing but still occupies one of those few thousand slots. Under load, the thread pool runs out while the CPU sits idle.

The old fix was asynchronous / reactive code (`CompletableFuture`, WebFlux). It scales, but it's harder to write, read and debug.

#### The idea

A **virtual thread** is a lightweight thread managed by the **JVM**, not by the operating system.

```
 Virtual threads (millions possible, cheap, stack lives on the heap)
   VT1   VT2   VT3   VT4   VT5   VT6   ...   VT1,000,000
    │     │     │
    ▼     ▼     ▼          mounted while running
 ┌──────┬──────┬──────┐
 │ CT 1 │ CT 2 │ CT 3 │   Carrier threads = a few platform threads (≈ number of CPU cores)
 └──────┴──────┴──────┘

 When VT1 blocks on I/O (socket read, sleep, lock):
   1. the JVM unmounts VT1 and saves its stack to the heap
   2. carrier CT1 immediately runs another virtual thread
   3. when the I/O completes, VT1 is mounted again (on any carrier) and continues
```

**Result:** you write ordinary blocking code (`jdbc.query(...)`, `httpClient.send(...)`), and it scales like async code.

#### Using them

**Java**

```java
// One new virtual thread per task. Don't pool virtual threads
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<String>> pages = urls.stream()
            .map(url -> executor.submit(() -> fetch(url)))   // plain blocking call
            .toList();

    for (Future<String> page : pages) {
        System.out.println(page.get());
    }
}   // close() waits for all tasks to finish

// Starting one directly
Thread.startVirtualThread(() -> handle(request));
Thread t = Thread.ofVirtual().name("worker-", 0).start(() -> doWork());
```

```java
// 100,000 tasks that each wait 1 second: finishes in about 1 second.
// With 100,000 platform threads, this would be slow or run out of memory.
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 100_000).forEach(i ->
            executor.submit(() -> {
                Thread.sleep(Duration.ofSeconds(1));
                return i;
            }));
}
```

```properties
# Spring Boot 3.2+: handle web requests on virtual threads
spring.threads.virtual.enabled=true
```

**C#**

.NET solves the same problem differently, with **`async`/`await`**. The compiler rewrites an `async` method into a state machine. At each `await` on I/O, the thread goes back to the thread pool, and the method resumes later on any free thread.

```csharp
using var http = new HttpClient();

string[] pages = await Task.WhenAll(urls.Select(url => http.GetStringAsync(url)));
```

```csharp
// 100,000 tasks that each wait 1 second: about 1 second, using only a handful of threads
var tasks = Enumerable.Range(0, 100_000).Select(async i =>
{
    await Task.Delay(TimeSpan.FromSeconds(1));
    return i;
});
await Task.WhenAll(tasks);
```

#### Rules of thumb

**1. They help I/O-bound work, not CPU-bound work.** There are still only as many CPU cores as before. Heavy computation needs parallelism (`parallelStream`, `ForkJoinPool` / `Parallel.For`, `Task.Run`), not more threads.

**2. Limit access to scarce resources with a semaphore, not a pool size.** With a million virtual threads, a database with 20 connections still accepts only 20 at a time.

**Java**

```java
private final Semaphore dbPermits = new Semaphore(20);

String query() throws InterruptedException {
    dbPermits.acquire();
    try {
        return db.call();
    } finally {
        dbPermits.release();
    }
}
```

**C#**

```csharp
private readonly SemaphoreSlim _dbPermits = new(20);

async Task<string> QueryAsync()
{
    await _dbPermits.WaitAsync();
    try
    {
        return await _db.CallAsync();
    }
    finally
    {
        _dbPermits.Release();
    }
}
```

**3. Watch out for blocking that can't be unmounted.**

**Java**: in Java 21–23, blocking inside a `synchronized` block **pins** the virtual thread to its carrier, so the carrier can't run anything else. Prefer `ReentrantLock` around blocking I/O. (Java 24 removes this limitation for `synchronized`.)

```java
synchronized (lock) {                 // ⚠️ Java 21–23: pins the carrier while waiting
    callRemoteService();
}

private final ReentrantLock lock = new ReentrantLock();
lock.lock();                          // ✅ the virtual thread can unmount while waiting
try { callRemoteService(); } finally { lock.unlock(); }
```

**C#**: the equivalent mistake is **sync-over-async**: blocking on a task with `.Result` or `.Wait()`. It ties up a thread-pool thread and can deadlock in some contexts.

```csharp
string page = http.GetStringAsync(url).Result;   // ⚠️ blocks a thread
string page = await http.GetStringAsync(url);    // ✅
```

**4. Be careful with `ThreadLocal` / `[ThreadStatic]` caches.** One expensive cached object per thread is fine with 200 threads, but not with a million virtual threads.

#### Java virtual threads vs. C# async/await

| | Java virtual threads | C# `async` / `await` |
|---|---|---|
| Who does the work | The **runtime** unmounts blocked threads transparently | The **compiler** rewrites methods into state machines |
| Code style | Ordinary blocking code, no new keywords | `async`, `await` and `Task<T>` spread up the call chain ("async all the way") |
| Existing libraries | Blocking JDK APIs (JDBC, `HttpClient`, sockets) work as they are | Need `...Async` APIs; blocking calls don't benefit |
| CPU-bound work | No gain | No gain |
| Limit concurrency | `Semaphore` | `SemaphoreSlim` |
| Main trap | Pinning (`synchronized` in Java 21–23, native calls) | Sync-over-async (`.Result`, `.Wait()`), `async void` |

> **Structured concurrency** (`StructuredTaskScope`, still preview) builds on virtual threads. It treats a group of subtasks as one unit: if one fails, the others are cancelled. It's roughly like `Task.WhenAll` plus a shared `CancellationToken` in C#.

---

## 7. Concurrency Basics

Concurrency means several tasks are in progress at the same time. It's what makes servers handle many requests and keeps UIs responsive. It's also where the hardest bugs live, because they depend on **timing** and often don't show up in tests.

| Concept | Java | C# |
|---|---|---|
| Run work on a pool of threads | `ExecutorService` | `ThreadPool` via `Task.Run` |
| Handle to a result that arrives later | `Future<T>` / `CompletableFuture<T>` | `Task<T>` |
| Compose async steps | `thenApply`, `thenCompose`, `allOf` | `await`, `Task.WhenAll` |
| Mutual exclusion | `synchronized`, `ReentrantLock` | `lock`, `Monitor`, `SemaphoreSlim` |
| Atomic counter | `AtomicInteger`, `LongAdder` | `Interlocked` |
| Thread-safe map | `ConcurrentHashMap` | `ConcurrentDictionary` |
| Producer–consumer queue | `BlockingQueue` | `Channel<T>`, `BlockingCollection<T>` |
| Visibility across threads | `volatile` | `volatile`, `Volatile.Read/Write` |
| Cancellation | `Thread.interrupt()`, `Future.cancel` | `CancellationToken` |

### 7.1 Thread Pools: ExecutorService / Task

Creating a new thread for every task is expensive, and the number of threads is unbounded. A **thread pool** keeps a fixed set of worker threads and feeds them tasks from a queue.

**Java**

```java
ExecutorService pool = Executors.newFixedThreadPool(4);
Future<Integer> future = pool.submit(() -> computeHeavy(42));   // Callable → Future

try {
    Integer result = future.get(5, TimeUnit.SECONDS);   // blocks until done or timeout
    System.out.println(result);
} catch (ExecutionException e) {
    Throwable realError = e.getCause();                 // the task's exception is wrapped
    log.error("Task failed", realError);
} catch (TimeoutException e) {
    future.cancel(true);                                // interrupt the task
} finally {
    pool.shutdown();                                    // stop accepting new tasks
    if (!pool.awaitTermination(30, TimeUnit.SECONDS)) {
        pool.shutdownNow();                             // interrupt what's still running
    }
}
```

Since Java 19, `ExecutorService` is `AutoCloseable`, so try-with-resources shuts it down and waits for you:

```java
try (ExecutorService pool = Executors.newFixedThreadPool(4)) {
    List<Callable<Report>> jobs = regions.stream()
            .map(r -> (Callable<Report>) () -> buildReport(r))
            .toList();

    for (Future<Report> f : pool.invokeAll(jobs)) {     // run all, wait for all
        save(f.get());
    }
}
```

**C#**

.NET already has a shared, self-tuning thread pool. You rarely create one yourself: `Task.Run` queues work onto it.

```csharp
Task<int> task = Task.Run(() => ComputeHeavy(42));

try
{
    int result = await task.WaitAsync(TimeSpan.FromSeconds(5));   // .NET 6+
    Console.WriteLine(result);
}
catch (TimeoutException)
{
    // ⚠️ WaitAsync stops *waiting*; the work keeps running unless it observes a CancellationToken
}
catch (Exception ex)
{
    // await rethrows the task's original exception (not wrapped)
    logger.LogError(ex, "Task failed");
}
```

```csharp
// Run many jobs with a concurrency limit, the equivalent of a fixed pool of 4
await Parallel.ForEachAsync(regions,
    new ParallelOptions { MaxDegreeOfParallelism = 4 },
    async (region, ct) =>
    {
        Report report = await BuildReportAsync(region, ct);
        await SaveAsync(report, ct);
    });
```

#### Choosing a pool (Java)

| Factory | What you get | Watch out |
|---|---|---|
| `newFixedThreadPool(n)` | `n` threads | The queue is **unbounded**: under overload, tasks pile up until you run out of memory |
| `newCachedThreadPool()` | New thread whenever none is free | Thread count is **unbounded** |
| `newSingleThreadExecutor()` | One thread, tasks run in order | Throughput limited to one task at a time |
| `newScheduledThreadPool(n)` | Delayed / periodic tasks | See the scheduling pitfall below |
| `newVirtualThreadPerTaskExecutor()` | One virtual thread per task (Java 21) | For I/O-bound work; see 6.4 |

For production services, build a **bounded** pool explicitly so overload is handled on purpose:

```java
ExecutorService pool = new ThreadPoolExecutor(
        4, 8,                                       // core and max threads
        60, TimeUnit.SECONDS,                       // idle time before extra threads are removed
        new ArrayBlockingQueue<>(1_000),            // bounded queue
        new ThreadPoolExecutor.CallerRunsPolicy()); // when full: the caller runs the task (natural back-pressure)
```

**Sizing rule of thumb:** for CPU-bound work, use about the number of cores (`Runtime.getRuntime().availableProcessors()` / `Environment.ProcessorCount`). For I/O-bound work, use more threads, or virtual threads (Java) / `async` I/O (C#).

#### Periodic tasks

**Java**

```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

scheduler.scheduleAtFixedRate(() -> {
    try {
        refreshCache();
    } catch (Exception e) {
        log.error("Refresh failed", e);   // ⚠️ an uncaught exception silently cancels ALL future runs
    }
}, 0, 5, TimeUnit.MINUTES);
```

**C#**

```csharp
using var timer = new PeriodicTimer(TimeSpan.FromMinutes(5));   // .NET 6+

while (await timer.WaitForNextTickAsync(stoppingToken))
{
    try
    {
        await RefreshCacheAsync(stoppingToken);
    }
    catch (Exception ex) when (ex is not OperationCanceledException)
    {
        logger.LogError(ex, "Refresh failed");                   // keep the loop alive
    }
}
```

#### `execute` vs. `submit`, and lost exceptions

**Java**

```java
pool.execute(() -> { throw new RuntimeException("boom"); });  // goes to the thread's uncaught handler (printed)
pool.submit(() -> { throw new RuntimeException("boom"); });   // ⚠️ stored in the Future; invisible until get()
```

**C#**

```csharp
_ = Task.Run(() => throw new Exception("boom"));   // ⚠️ fire-and-forget: nobody observes the failure
await Task.Run(() => throw new Exception("boom")); // ✅ the exception surfaces here
```

### 7.2 CompletableFuture / async-await

A plain `Future` can only be waited on with `get()`, which blocks. `CompletableFuture` (Java 8+) lets you **chain** and **combine** asynchronous steps. In C#, `Task<T>` plus `async`/`await` covers the same ground with ordinary-looking code.

#### Operation mapping

| Purpose | Java `CompletableFuture` | C# `Task` |
|---|---|---|
| Start async work | `supplyAsync(fn, executor)` | `Task.Run(fn)` or call an `...Async` method |
| Transform the result | `thenApply(fn)` | `var x = await t; Transform(x)` |
| Chain another async step | `thenCompose(fn)` | `await NextAsync(await t)` |
| Combine two results | `a.thenCombine(b, fn)` | `await Task.WhenAll(a, b)` then use both |
| Wait for all | `allOf(...)` | `Task.WhenAll(...)` |
| First to finish | `anyOf(...)` | `Task.WhenAny(...)` |
| Recover from failure | `exceptionally`, `handle` | `try` / `catch` around `await` |
| Timeout | `orTimeout`, `completeOnTimeout` (Java 9+) | `WaitAsync(timeout)`, `CancellationTokenSource.CancelAfter` |
| Get the result | `join()` / `get()` (blocking) | `await` (avoid `.Result`) |

#### Example: build a dashboard from three services

Load the user first. Then load orders and recommendations **in parallel**. Recommendations are optional, so fall back to an empty list if they're slow or fail. The whole thing has a 2-second timeout.

**Java**

```java
ExecutorService io = Executors.newVirtualThreadPerTaskExecutor();   // or a bounded pool for I/O

CompletableFuture<Dashboard> dashboard =
        CompletableFuture.supplyAsync(() -> userService.find(userId), io)
                .thenCompose(user -> {
                    var orders = CompletableFuture.supplyAsync(
                            () -> orderService.recent(user), io);

                    var recs = CompletableFuture.supplyAsync(
                                    () -> recoService.forUser(user), io)
                            .completeOnTimeout(List.of(), 500, TimeUnit.MILLISECONDS)
                            .exceptionally(ex -> List.of());              // optional: fall back

                    return orders.thenCombine(recs,
                            (o, r) -> new Dashboard(user, o, r));
                })
                .orTimeout(2, TimeUnit.SECONDS);                          // overall limit

Dashboard d = dashboard.join();   // throws CompletionException wrapping the real cause
```

**C#**

```csharp
async Task<Dashboard> LoadDashboardAsync(long userId, CancellationToken ct)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
    cts.CancelAfter(TimeSpan.FromSeconds(2));                             // overall limit

    User user = await userService.FindAsync(userId, cts.Token);

    Task<List<Order>>   orders = orderService.RecentAsync(user, cts.Token);   // both start now,
    Task<List<Product>> recs   = GetRecsOrEmptyAsync(user, cts.Token);       // in parallel

    await Task.WhenAll(orders, recs);
    return new Dashboard(user, await orders, await recs);   // already finished: no extra wait
}

async Task<List<Product>> GetRecsOrEmptyAsync(User user, CancellationToken ct)
{
    try
    {
        return await recoService.ForUserAsync(user, ct)
                                .WaitAsync(TimeSpan.FromMilliseconds(500), ct);
    }
    catch (Exception ex) when (ex is TimeoutException or HttpRequestException)
    {
        return new List<Product>();                          // optional: fall back
    }
}
```

#### Common pitfalls

**1. Running blocking I/O on the default pool**

**Java**: without an executor argument, `supplyAsync` uses `ForkJoinPool.commonPool()`. Parallel streams use the same small pool (about one thread per core). Blocking I/O there starves everything else.

```java
CompletableFuture.supplyAsync(() -> jdbc.query(sql));        // ❌ blocks the common pool
CompletableFuture.supplyAsync(() -> jdbc.query(sql), io);    // ✅ dedicated I/O executor
```

**C#**: wrapping I/O in `Task.Run` wastes a pool thread that just waits. Call the real async API instead.

```csharp
var data = await Task.Run(() => File.ReadAllText(path));    // ❌ a thread blocks on I/O
var data = await File.ReadAllTextAsync(path);               // ✅ no thread waits
```

**2. Blocking on async results**

**Java**

```java
// Inside a CompletableFuture stage running on a small pool:
.thenApply(user -> otherFuture.join())          // ❌ blocks a pool thread; can starve or deadlock
.thenCompose(user -> otherFuture)               // ✅ compose instead of waiting
```

**C#**

```csharp
var user = GetUserAsync(id).Result;             // ❌ blocks; can deadlock in UI / legacy ASP.NET contexts
var user = await GetUserAsync(id);              // ✅
```

**3. Cancellation doesn't work the way you'd expect**

In both languages, cancellation is **cooperative**: the running code has to check for it.

**Java**: `CompletableFuture.cancel(true)` **doesn't interrupt** the running task. It only completes the future with a `CancellationException`. Cancel the underlying `Future` from an executor (which interrupts the thread), and have long-running work check the interrupt flag:

```java
for (Item item : items) {
    if (Thread.currentThread().isInterrupted()) {
        throw new CancellationException();
    }
    process(item);
}
```

**C#**: pass the `CancellationToken` all the way down, and check it in loops:

```csharp
foreach (var item in items)
{
    ct.ThrowIfCancellationRequested();
    await ProcessAsync(item, ct);
}
```

### 7.3 Race Conditions

A **race condition** happens when the result depends on the timing of threads. It needs three ingredients:

1. **Shared** state,
2. that is **mutable**,
3. accessed by several threads **without synchronization**, with at least one of them writing.

Remove any one ingredient and the race disappears. That's the basis for all the fixes in 7.4.

#### Race #1: read-modify-write (`count++`)

**Java**

```java
class Counter {
    private int count = 0;
    void increment() { count++; }        // ❌ not atomic
    int get() { return count; }
}

Counter c = new Counter();
try (var pool = Executors.newFixedThreadPool(8)) {
    for (int i = 0; i < 100_000; i++) {
        pool.submit(c::increment);
    }
}
System.out.println(c.get());             // expected 100000, often less (e.g. 99412)
```

**C#**

```csharp
class Counter
{
    private int _count;
    public void Increment() => _count++;  // ❌ not atomic
    public int Count => _count;
}

var c = new Counter();
Parallel.For(0, 100_000, _ => c.Increment());
Console.WriteLine(c.Count);               // expected 100000, often less
```

`count++` is really three steps: **read**, **add**, **write**. Two threads can interleave them:

```
Thread A: read count   → 5
Thread B: read count   → 5
Thread A: write 5 + 1  → 6
Thread B: write 5 + 1  → 6     ← one increment is lost
```

#### Race #2: check-then-act

The check and the action are each fine, but another thread can act **between** them.

**Java**

```java
// ❌ two threads can both see "missing" and both load and put
if (!cache.containsKey(key)) {
    cache.put(key, loadExpensive(key));
}

// ❌ lazy initialization: two threads can both see null and create two instances
if (instance == null) {
    instance = new ExpensiveService();
}
```

**C#**

```csharp
// ❌ same problem
if (!cache.ContainsKey(key))
{
    cache[key] = LoadExpensive(key);
}

// ❌ same problem
if (_instance == null)
{
    _instance = new ExpensiveService();
}
```

Using a thread-safe map doesn't fix this on its own: each call is safe, but the *sequence* of two calls is not. Section 7.4 shows atomic alternatives.

#### Race #3: visibility

For performance, the CPU and compiler may keep a value in a register or cache. Without synchronization, **one thread may never see another thread's write**.

**Java**

```java
private boolean running = true;           // ❌ the worker may never see `false`

void stop()   { running = false; }
void worker() { while (running) { doWork(); } }

private volatile boolean running = true;  // ✅ every read sees the latest write
```

**C#**

```csharp
private bool _running = true;             // ❌ the JIT may hoist the read out of the loop

public void Stop()   => _running = false;
public void Worker() { while (_running) DoWork(); }

private volatile bool _running = true;    // ✅
// Even better in C#: pass a CancellationToken instead of a flag
```

`volatile` fixes **visibility** only, not atomicity. `volatile int count; count++` still loses updates.

#### Race #4: non-thread-safe collections

**Java**

```java
Map<String, Integer> map = new HashMap<>();
// Several threads calling map.put(...) at once:
// lost entries, corrupted internal structure, wrong size()
```

**C#**

```csharp
var dict = new Dictionary<string, int>();
// Several threads writing at once:
// lost entries, exceptions, or even an infinite loop inside the dictionary
```

`ArrayList`/`List<T>`, `HashMap`/`Dictionary` and `HashSet` are **not** safe for concurrent writes. Concurrent reads with no writer are fine.

#### Bonus: deadlock

Two threads each hold one lock and wait forever for the other's.

**Java**

```java
// Thread 1: transfer(a, b)   locks a, then waits for b
// Thread 2: transfer(b, a)   locks b, then waits for a   → both stuck forever
void transfer(Account from, Account to, BigDecimal amount) {
    synchronized (from.lock()) {
        synchronized (to.lock()) {
            from.withdraw(amount);
            to.deposit(amount);
        }
    }
}

// ✅ always lock in the same global order (e.g. by id)
void transfer(Account from, Account to, BigDecimal amount) {
    Account first  = from.id() < to.id() ? from : to;
    Account second = from.id() < to.id() ? to : from;
    synchronized (first.lock()) {
        synchronized (second.lock()) {
            from.withdraw(amount);
            to.deposit(amount);
        }
    }
}
```

**C#**

```csharp
// ❌ same deadlock with lock (from.SyncRoot) { lock (to.SyncRoot) { ... } }

// ✅ consistent ordering
void Transfer(Account from, Account to, decimal amount)
{
    var (first, second) = from.Id < to.Id ? (from, to) : (to, from);
    lock (first.SyncRoot)
    {
        lock (second.SyncRoot)
        {
            from.Withdraw(amount);
            to.Deposit(amount);
        }
    }
}
```

Other fixes: use a timeout (`tryLock(timeout)` in Java, `Monitor.TryEnter(obj, timeout)` in C#), or redesign so a thread never needs two locks at once.

### 7.4 Writing Thread-Safe Code

Work down this list. The first strategies are the simplest and safest; reach for locks only when the earlier ones don't fit.

#### Strategy 1: Don't share (confinement)

If each thread works on **its own data** and returns a result, there's nothing to race on.

**Java**

```java
// ❌ threads add to a shared list
List<Result> results = new ArrayList<>();
inputs.parallelStream().forEach(in -> results.add(process(in)));    // race

// ✅ each task returns its result; the framework combines them safely
List<Result> results = inputs.parallelStream().map(this::process).toList();
```

**C#**

```csharp
// ❌
var results = new List<Result>();
Parallel.ForEach(inputs, input => results.Add(Process(input)));       // race

// ✅
List<Result> results = inputs.AsParallel().Select(Process).ToList();

// ✅ async version
Result[] results = await Task.WhenAll(inputs.Select(ProcessAsync));
```

Local variables are always confined to their thread. Keep state local wherever you can.

#### Strategy 2: Make it immutable

An object that can't change can be shared freely. Use records (section 6.1), `final`/`readonly` fields, and immutable collections.

To **update** shared data, build a new immutable snapshot and **swap the reference** in one step. Readers never need a lock.

**Java**

```java
public class PriceCatalog {
    private volatile Map<String, Price> prices = Map.of();   // immutable snapshot

    public void refresh() {
        prices = Map.copyOf(loadPricesFromDb());   // build new, then publish in one write
    }

    public Price get(String sku) {
        return prices.get(sku);                     // lock-free read of a consistent snapshot
    }
}
```

**C#**

```csharp
public class PriceCatalog
{
    private volatile IReadOnlyDictionary<string, Price> _prices =
        ImmutableDictionary<string, Price>.Empty;

    public void Refresh() =>
        _prices = LoadPricesFromDb().ToFrozenDictionary();   // .NET 8+: fast, read-only

    public Price? Get(string sku) =>
        _prices.TryGetValue(sku, out var p) ? p : null;
}
```

#### Strategy 3: Use thread-safe building blocks

**Atomic counters**

**Java**

```java
private final AtomicInteger count = new AtomicInteger();
count.incrementAndGet();                  // atomic read-modify-write

private final LongAdder hits = new LongAdder();   // faster when many threads update often
hits.increment();
long total = hits.sum();
```

**C#**

```csharp
private int _count;
Interlocked.Increment(ref _count);        // atomic read-modify-write
int current = Volatile.Read(ref _count);
```

**Concurrent maps: use the atomic compound operations**

**Java**

```java
Map<String, Integer> hits = new ConcurrentHashMap<>();
hits.merge(page, 1, Integer::sum);               // ✅ atomic "increment or insert 1"

Map<String, Data> cache = new ConcurrentHashMap<>();
Data d = cache.computeIfAbsent(key, this::load); // ✅ atomic check-then-act; load runs at most once
// ⚠️ the function runs while part of the map is locked: keep it short,
//    and never modify the same map inside it
```

**C#**

```csharp
var hits = new ConcurrentDictionary<string, int>();
hits.AddOrUpdate(page, 1, (_, old) => old + 1);  // ✅ atomic "increment or insert 1"

var cache = new ConcurrentDictionary<string, Data>();
Data d = cache.GetOrAdd(key, Load);
// ⚠️ under contention, Load may run MORE THAN ONCE (only one result is kept).
//    If Load is expensive or has side effects, wrap it in Lazy<T>:
var lazyCache = new ConcurrentDictionary<string, Lazy<Data>>();
Data d2 = lazyCache.GetOrAdd(key, k => new Lazy<Data>(() => Load(k))).Value;
```

**Lazy initialization**

**Java**

```java
// Holder idiom: the JVM guarantees the class is initialized once, thread-safely, on first use
public class Services {
    private static class Holder {
        static final ExpensiveService INSTANCE = new ExpensiveService();
    }
    public static ExpensiveService get() { return Holder.INSTANCE; }
}
```

**C#**

```csharp
public static class Services
{
    private static readonly Lazy<ExpensiveService> _instance =
        new(() => new ExpensiveService());       // thread-safe by default

    public static ExpensiveService Get() => _instance.Value;
}
```

**Producer–consumer queues** (message passing instead of shared state)

**Java**

```java
BlockingQueue<Job> queue = new ArrayBlockingQueue<>(100);   // bounded: producers wait when full

// Producer thread
queue.put(job);

// Consumer thread
while (!Thread.currentThread().isInterrupted()) {
    Job job = queue.take();                                 // waits while empty
    process(job);
}
```

**C#**

```csharp
var channel = Channel.CreateBounded<Job>(100);              // bounded: producers wait when full

// Producer
await channel.Writer.WriteAsync(job, ct);

// Consumer
await foreach (Job job in channel.Reader.ReadAllAsync(ct))
{
    await ProcessAsync(job, ct);
}
```

#### Strategy 4: Use locks correctly

When several fields must change **together**, or an invariant spans multiple values, protect them with a lock.

**Java**

```java
public class BankAccount {
    private final Object lock = new Object();      // private lock object
    private BigDecimal balance = BigDecimal.ZERO;
    private final List<Transaction> history = new ArrayList<>();

    public void deposit(BigDecimal amount) {
        synchronized (lock) {                       // both fields change together
            balance = balance.add(amount);
            history.add(new Transaction(amount));
        }
    }

    public BigDecimal balance() {
        synchronized (lock) {                       // reads need the lock too (visibility)
            return balance;
        }
    }
}
```

`ReentrantLock` gives you more control: `tryLock` with a timeout, interruptible waits, fairness. It also works better with virtual threads in Java 21–23 (see 6.4):

```java
private final ReentrantLock lock = new ReentrantLock();

public void deposit(BigDecimal amount) {
    lock.lock();
    try {
        balance = balance.add(amount);
    } finally {
        lock.unlock();                              // always in finally
    }
}
```

**C#**

```csharp
public class BankAccount
{
    private readonly object _lock = new();         // .NET 9+: private readonly Lock _lock = new();
    private decimal _balance;
    private readonly List<Transaction> _history = new();

    public void Deposit(decimal amount)
    {
        lock (_lock)                               // both fields change together
        {
            _balance += amount;
            _history.Add(new Transaction(amount));
        }
    }

    public decimal Balance
    {
        get { lock (_lock) { return _balance; } }  // reads need the lock too
    }
}
```

You **can't `await` inside a `lock`** in C# (compile error CS1996). For async code, use `SemaphoreSlim(1, 1)`:

```csharp
private readonly SemaphoreSlim _mutex = new(1, 1);

public async Task SaveAsync(Data data)
{
    await _mutex.WaitAsync();
    try
    {
        await _store.WriteAsync(data);
    }
    finally
    {
        _mutex.Release();
    }
}
```

**Lock rules (both languages)**

| ✅ Do | ❌ Don't |
|---|---|
| Lock on a **private, final/readonly** object | Lock on `this`, a `String`/`string`, or a `Class`/`Type`: outside code can lock on them too |
| Keep the locked section **short** | Do I/O, network calls or call unknown code while holding a lock |
| Use the **same lock** for every read and write of the guarded data | Guard writes but read without the lock |
| Acquire multiple locks in a **fixed order** | Nest locks in different orders (deadlock) |
| Release in `finally` (`ReentrantLock`, `SemaphoreSlim`) | Forget to unlock on the exception path |
| Use `ReadWriteLock` / `ReaderWriterLockSlim` for read-heavy data | Reach for read/write locks before measuring: they're slower for short sections |

#### Thread-safety checklist

Before you ship code that runs on several threads, check:

1. **What state is shared?** List every field that more than one thread can touch.
2. **Can it be unshared or immutable?** That's the best fix (strategies 1 and 2).
3. **Is every compound action atomic?** Look for `x++`, check-then-act and read-modify-write sequences.
4. **Is every read of shared mutable state synchronized, or `volatile`/atomic?** Visibility bugs come from unguarded reads.
5. **Are your collections thread-safe** for the way they're used?
6. **Could two locks ever be taken in different orders?**
7. **Is any blocking work** running on a shared pool (common pool, UI thread, `.Result`)?
8. **Is it documented?** Write "thread-safe" or "not thread-safe" in the class comment, so the next person knows.

> Race conditions are often invisible in unit tests and appear only under production load. Reason about the code, and prefer designs where races can't happen, over testing to show they don't.

---

## 8. Cheat Sheet

### OOP pillars

| Pillar | Key idea | Keyword(s) |
|---|---|---|
| Encapsulation | Hide state, guard invariants | `private`, getters/properties |
| Abstraction | Expose *what*, hide *how* | `interface`, `abstract` |
| Inheritance | Reuse via "is-a" | `extends` (Java), `:` (C#) |
| Polymorphism | One interface, many behaviors | `@Override` / `override`, `virtual` |

### SOLID quick check

| Ask yourself… | If yes → |
|---|---|
| Does this class change for more than one reason? | Split it (**SRP**) |
| Do I edit existing code every time I add a variant? | Introduce an abstraction (**OCP**) |
| Does a subclass throw "not supported" or need `instanceof` checks? | Rethink the hierarchy (**LSP**) |
| Do implementers leave methods empty? | Split the interface (**ISP**) |
| Does a class call `new` on its dependencies? | Inject an interface (**DIP**) |

### Patterns at a glance

| Pattern | Family | Problem it solves | Signal you need it |
|---|---|---|---|
| **Strategy** | Behavioral | Swap algorithms at runtime | Long `if/else` or `switch` choosing *behavior* |
| **Factory** | Creational | Hide which concrete class gets created | Long `if/else` or `switch` choosing *which class to `new`* |
| **Builder** | Creational | Build complex objects step by step | Constructors with many (optional) parameters |

> **Patterns combine well.** A common pairing: a **Factory** picks the right **Strategy** from config, and a **Builder** constructs the configuration object. All three keep code aligned with **OCP** and **DIP**.

### Collections quick pick

| Need | Java | C# |
|---|---|---|
| Ordered list, index access (default) | `ArrayList` | `List<T>` |
| Unique items, fast `contains` | `HashSet` | `HashSet<T>` |
| Key → value lookup | `HashMap` | `Dictionary<TKey,TValue>` |
| Sorted keys / items | `TreeMap` / `TreeSet` | `SortedDictionary` / `SortedSet` |
| Insertion-ordered map | `LinkedHashMap` | `OrderedDictionary<TKey,TValue>` (.NET 9+) |
| Queue / stack | `ArrayDeque` | `Queue<T>` / `Stack<T>` |
| Thread-safe map | `ConcurrentHashMap` | `ConcurrentDictionary` |

### equals / hashCode rules

1. Override **both** or **neither**.
2. Use the **same fields** in both.
3. Equal objects **must** have equal hash codes.
4. Keep keys **immutable**.
5. Prefer **records** (Java 16+, C# 9+). They get this right for you.

### Streams / LINQ

| Java | C# | Watch out for |
|---|---|---|
| `filter` / `map` / `flatMap` | `Where` / `Select` / `SelectMany` | Side effects inside lambdas |
| `toList()` | `ToList()` | Java streams are single-use; LINQ queries re-run on each enumeration |
| `Collectors.toMap` | `ToDictionary` | Both throw on duplicate keys |
| `findFirst()` → `Optional` | `FirstOrDefault()` | C# returns `0` / `null` when nothing is found |

### Optional / Nullable

| Java | C# |
|---|---|
| `Optional<T>` return type | `T?` with `<Nullable>enable</Nullable>` |
| `.map(...).orElse(x)` | `?.` … `?? x` |
| `.orElseThrow(...)` | `?? throw ...` |
| `orElseGet` for expensive defaults | `??` is already lazy |
| Not for fields, parameters or collections | Don't silence warnings with `!` |

### Exception rules

1. **Throw early** (validate inputs) and **catch late** (only where you can handle it).
2. **Never swallow.** No empty `catch` blocks.
3. **Catch specific types**, most specific first; catch `Exception` only at the top-level boundary.
4. **Keep the cause:** pass it to the new exception. In C#, use `throw;`, not `throw ex;`.
5. **Release resources** with `try`-with-resources / `using`. Never `return` from `finally`.
6. **No exceptions for control flow:** use `Optional` / `TryParse`.
7. **Log once**, in a global handler, and never leak stack traces to users.

### Modern Java ↔ C#

| Java 17 / 21 | C# equivalent | Use it for |
|---|---|---|
| `record Money(BigDecimal amount, String currency)` | `record Money(decimal Amount, string Currency)` | DTOs, value objects, map keys |
| Compact constructor `public Money { ... }` | Explicit constructor (positional `with` skips validation) | Validation, defensive copies |
| `sealed interface X permits A, B` | Abstract record + private constructor + nested subtypes | Closed sets of cases: results, events, states |
| Exhaustive `switch` with no `default` | `switch` with `_ => throw new UnreachableException()` | Compile-time safety when cases are added |
| `obj instanceof String s` | `obj is string s` | Type test + cast in one step |
| `case Integer i when i > 100 ->` | `int i and > 100 =>` / `when` | Type patterns with conditions |
| `case Point(int x, int y) ->` | `Point(var x, var y) =>` | Deconstructing records |
| Virtual threads | `async` / `await` | Scaling I/O-bound concurrency |
| `Semaphore` | `SemaphoreSlim` | Limiting access to scarce resources |

### Concurrency

| Need | Java | C# |
|---|---|---|
| Run tasks on a pool | `ExecutorService` (bounded `ThreadPoolExecutor` in production) | `Task.Run`, `Parallel.ForEachAsync` |
| Chain / combine async steps | `CompletableFuture` + `thenCompose` / `allOf` | `async` / `await` + `Task.WhenAll` |
| Atomic counter | `AtomicInteger`, `LongAdder` | `Interlocked.Increment` |
| Atomic check-then-act on a map | `computeIfAbsent`, `merge` | `GetOrAdd` (+ `Lazy<T>`), `AddOrUpdate` |
| Mutual exclusion | `synchronized`, `ReentrantLock` | `lock`, `SemaphoreSlim(1,1)` for async code |
| Hand work between threads | `BlockingQueue` | `Channel<T>` |
| Stop flag visible to all threads | `volatile` (or interrupts) | `CancellationToken` |

**Thread-safety, in order of preference:** don't share → make it immutable → use atomics and concurrent collections → use locks.

---

### Final advice

- Patterns are tools, not goals. Don't add one until the problem it solves actually shows up.
- Prefer simple code first; refactor toward a pattern when duplication or rigid conditionals appear.
- SOLID principles are guidelines — apply them with judgment, not dogmatically.