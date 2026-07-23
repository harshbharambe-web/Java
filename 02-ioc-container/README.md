# 02 — Inversion of Control (IoC) & IoC Container

## 🤔 What is IoC (in plain English)

Normally: **your code** controls the flow — you create objects, you call
methods, you manage lifecycles.

With IoC: **control is inverted** — the framework creates objects, wires them
together, and calls your code when needed. You just plug into it.

> Analogy: In a normal restaurant kitchen, you (the chef) decide when to
> chop, fry, plate. In a **conveyor-belt sushi place**, the belt (framework)
> decides the flow — you just place dishes on it (write beans) and pick
> them off when needed.

## 🏭 What is the IoC Container

It's the actual **engine inside Spring** that implements the IoC principle.
In Spring, this is the `ApplicationContext` (or the more basic `BeanFactory`).

Its job:
1. Read configuration (annotations / XML / Java config)
2. Create bean instances
3. Inject dependencies into them (DI)
4. Manage their scope and lifecycle
5. Hand them to you via `context.getBean(...)` (or auto-inject them)

```
        ┌────────────────────────────────────────────┐
        │              IoC Container                    │
        │        (ApplicationContext)                   │
        │                                                │
        │  1. Scans/reads config (@Component, @Bean...) │
        │  2. Creates bean objects                       │
        │  3. Injects dependencies (DI)                  │
        │  4. Manages scope + lifecycle                  │
        │  5. Gives you the ready-to-use bean             │
        └───────────────┬────────────────────────────────┘
                        │
             ┌──────────┼──────────┐
             ▼          ▼          ▼
          Bean A     Bean B     Bean C
       (fully wired, ready to use)
```

## 🧱 BeanFactory vs ApplicationContext

```
┌───────────────────┬─────────────────────────────┬───────────────────────────────┐
│                    │ BeanFactory                  │ ApplicationContext              │
├───────────────────┼─────────────────────────────┼───────────────────────────────┤
│ Loading            │ Lazy (bean created on demand) │ Eager (singleton beans created  │
│                    │                               │ at startup by default)          │
│ Features            │ Basic DI only                 │ DI + AOP + events + i18n +      │
│                    │                               │ environment abstraction + more  │
│ Used in practice?  │ Rarely directly               │ Almost always (this is what     │
│                    │                               │ Spring Boot uses under the hood)│
└───────────────────┴─────────────────────────────┴───────────────────────────────┘
```

## 💻 Code — seeing the container in action

### `Greeter.java`
```java
import org.springframework.stereotype.Component;

@Component
public class Greeter {
    public Greeter() {
        System.out.println("Greeter bean created by the container");
    }
    public void greet() {
        System.out.println("Hello from the IoC container!");
    }
}
```

### `MainApp.java`
```java
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan(basePackages = "com.example")
public class MainApp {
    public static void main(String[] args) {
        System.out.println("Before creating context - no beans exist yet");

        ApplicationContext context = new AnnotationConfigApplicationContext(MainApp.class);
        // ^ At this line, ALL singleton beans are already created (eager loading)

        System.out.println("Context created. Now fetching bean...");

        Greeter greeter = context.getBean(Greeter.class); // already created, just fetched
        greeter.greet();

        // proof it's the IoC container managing a single shared instance:
        Greeter greeter2 = context.getBean(Greeter.class);
        System.out.println("Same instance? " + (greeter == greeter2)); // true for singleton
    }
}
```

**Expected output:**
```
Before creating context - no beans exist yet
Greeter bean created by the container
Context created. Now fetching bean...
Hello from the IoC container!
Same instance? true
```

Notice: `"Greeter bean created..."` prints **before** `"Context created..."` —
proof the container eagerly creates singleton beans right when the context
is built, not when you call `getBean()`.

## 🎯 Interview Questions & Answers

**Q1. What is Inversion of Control?**
> A design principle where the control of creating and managing object
> dependencies is transferred from the application code to a
> framework/container. It's the "what" — DI is the "how."

**Q2. What is the IoC Container in Spring?**
> The runtime component (`ApplicationContext`/`BeanFactory`) responsible for
> instantiating, configuring, wiring, and managing the complete lifecycle of
> beans.

**Q3. Difference between `BeanFactory` and `ApplicationContext`?**
> `BeanFactory` is the basic container (lazy init, DI only). `ApplicationContext`
> extends it with eager singleton init by default, AOP support, event
> publishing, internationalization, and easier integration with Spring Boot.
> In real projects you almost always use `ApplicationContext`.

**Q4. Is IoC unique to Spring?**
> No — IoC is a general design principle (also seen in other frameworks/
> languages). Spring is just a very popular *implementation* of it via its
> IoC container. Dependency Injection is one way to implement IoC; Spring
> also uses it for things like AOP proxies and event handling.

**Q5. When are singleton beans actually created — at startup or on first use?**
> By default, **at startup**, right when the `ApplicationContext` is
> refreshed/created (eager initialization). You can override this per-bean
> with `@Lazy` to defer creation until first use.

## 🔥 Gotchas / Trending points

- `@Lazy` on a bean flips it from eager to lazy creation — useful for
  expensive beans not needed immediately, but can hide config errors until
  runtime (they'd normally surface at startup with eager loading).
- The container doesn't just do DI — it's also the backbone for AOP proxies,
  application events (`ApplicationEventPublisher`), and profile-based
  bean selection (`@Profile`).
- In Spring Boot, `SpringApplication.run()` internally creates and refreshes
  an `ApplicationContext` for you — you rarely construct it manually like in
  this demo, but it's exactly the same mechanism underneath.
