# 01 — Dependency Injection (DI)

## 🤔 What is it (in plain English)

Instead of a class **creating** the objects it depends on, those objects are
**given to it from outside** (by Spring). That's it. That's DI.

> Analogy: A car doesn't manufacture its own engine. The engine is built
> separately and **installed into** the car. The car just uses it.

Without DI:
```java
class Car {
    Engine engine = new Engine(); // Car is tightly coupled to Engine
}
```

With DI:
```java
class Car {
    Engine engine;
    Car(Engine engine) {      // Engine is "injected" from outside
        this.engine = engine;
    }
}
```

Now `Car` doesn't care *how* `Engine` is built, or which implementation of
`Engine` it gets. That's loose coupling — the whole point.

## 🔗 How DI relates to IoC

DI is the **mechanism**; IoC is the **principle**. See [02-ioc-container](../02-ioc-container) for the full picture.

## 🧩 Types of DI in Spring

```
┌─────────────────────┬───────────────────────────────────────────┬─────────────────┐
│ Type                 │ How                                        │ Preferred?       │
├─────────────────────┼───────────────────────────────────────────┼─────────────────┤
│ Constructor Injection│ dependency passed via constructor           │ ✅ YES (best)    │
│ Setter Injection     │ dependency passed via a setter method       │ ⚠️ sometimes     │
│ Field Injection      │ @Autowired directly on the field            │ ❌ avoid in prod │
└─────────────────────┴───────────────────────────────────────────┴─────────────────┘
```

## 💻 Code

### `Engine.java`
```java
public interface Engine {
    void start();
}
```

### `PetrolEngine.java`
```java
import org.springframework.stereotype.Component;

@Component
public class PetrolEngine implements Engine {
    @Override
    public void start() {
        System.out.println("Petrol engine starting...");
    }
}
```

### `Car.java` — Constructor Injection (recommended)
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class Car {

    private final Engine engine; // final -> guarantees immutability

    @Autowired // optional on single constructor since Spring 4.3, but explicit is clearer
    public Car(Engine engine) {
        this.engine = engine;
    }

    public void drive() {
        engine.start();
        System.out.println("Car is driving");
    }
}
```

### `CarSetter.java` — Setter Injection (for optional dependencies)
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class CarSetter {

    private Engine engine;

    @Autowired
    public void setEngine(Engine engine) {
        this.engine = engine;
    }

    public void drive() {
        engine.start();
    }
}
```

### `CarFieldInjection.java` — Field Injection (easy but discouraged)
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class CarFieldInjection {

    @Autowired
    private Engine engine; // can't be final, harder to unit test without Spring/reflection

    public void drive() {
        engine.start();
    }
}
```

### `MainApp.java`
```java
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan(basePackages = "com.example") // scans for @Component classes
public class MainApp {
    public static void main(String[] args) {
        var context = new AnnotationConfigApplicationContext(MainApp.class);
        Car car = context.getBean(Car.class);
        car.drive();
    }
}
```

**Expected output:**
```
Petrol engine starting...
Car is driving
```

## 🎯 Interview Questions & Answers

**Q1. Why is constructor injection preferred over field injection?**
> - Makes fields `final` → immutable, thread-safe by design
> - Dependencies are explicit and mandatory — object can't exist in a half-wired state
> - Easy to unit test — just call `new Car(mockEngine)`, no Spring/reflection needed
> - Fails fast at startup if a dependency is missing, instead of a `NullPointerException` at runtime
> - Helps you *notice* circular dependencies immediately (Spring throws an error instead of silently working around it)

**Q2. What's the difference between DI and a Factory pattern?**
> Both decouple object creation from usage, but Factory is a design pattern you write yourself and call explicitly (`Factory.create()`), while DI is done *for* you by a framework/container based on configuration/annotations — you never call `new` or a factory method yourself.

**Q3. Can Spring inject a dependency that doesn't have `@Component`?**
> Not automatically — the dependency must itself be a Spring bean (registered via `@Component`, `@Bean`, XML config, etc.) unless you define it manually with `@Bean` in a `@Configuration` class.

**Q4. What happens if two beans of the same interface type exist and you use constructor injection without qualifying?**
> `NoUniqueBeanDefinitionException` — you must disambiguate using `@Qualifier` or `@Primary` (covered in [03-bean-annotations](../03-bean-annotations)).

**Q5. Is field injection actually "wrong"?**
> It works, but it's discouraged in production code for the testing/immutability reasons above. It's fine for quick prototypes/demos.

## 🔥 Gotchas / Trending points

- `@Autowired` is *not required* on a constructor if the class has **only one constructor** (Spring 4.3+) — but writing it explicitly is still good practice for readability.
- Mixing all 3 injection types in one project is a code smell — pick constructor injection as default, use setter injection only for **optional** dependencies.
- DI doesn't remove coupling entirely — it moves the coupling from "class knows the concrete implementation" to "class knows the interface." That's the tradeoff, and it's a good one.
