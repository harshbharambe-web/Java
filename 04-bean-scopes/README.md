# 04 — Bean Scope

## 🤔 What is Bean Scope

Scope decides **how many instances** of a bean the container creates and
**how long they live**.

```
┌───────────────┬────────────────────────────────────────────────────────────┐
│ Scope           │ Meaning                                                        │
├───────────────┼────────────────────────────────────────────────────────────┤
│ singleton (default) │ ONE instance per Spring container, shared everywhere      │
│ prototype       │ A NEW instance every time the bean is requested/injected      │
│ request         │ One instance PER HTTP request (web apps only)                 │
│ session         │ One instance PER HTTP session (web apps only)                  │
│ application     │ One instance per ServletContext (web apps only)               │
│ websocket       │ One instance per WebSocket session (web apps only)              │
└───────────────┴────────────────────────────────────────────────────────────┘
```

The last four only make sense in a web application context (need
`spring-web`); in a plain console app you'll mostly deal with
**singleton vs prototype**.

## 📊 Singleton vs Prototype — visualized

```
SINGLETON (default)
────────────────────
context.getBean(A.class) ──► instance #1 ──┐
context.getBean(A.class) ──► instance #1 ──┼──► SAME object every time
context.getBean(A.class) ──► instance #1 ──┘

PROTOTYPE
─────────
context.getBean(B.class) ──► instance #1
context.getBean(B.class) ──► instance #2   ──► NEW object every time
context.getBean(B.class) ──► instance #3
```

## 💻 Code

### `SingletonBean.java`
```java
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;

@Component
// @Scope("singleton") // this is the DEFAULT, so it's usually omitted
public class SingletonBean {
    public SingletonBean() {
        System.out.println("SingletonBean created: " + this);
    }
}
```

### `PrototypeBean.java`
```java
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;

@Component
@Scope("prototype")
public class PrototypeBean {
    public PrototypeBean() {
        System.out.println("PrototypeBean created: " + this);
    }
}
```

### `MainApp.java`
```java
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan(basePackages = "com.example")
public class MainApp {
    public static void main(String[] args) {
        var context = new AnnotationConfigApplicationContext(MainApp.class);

        System.out.println("--- Singleton test ---");
        SingletonBean s1 = context.getBean(SingletonBean.class);
        SingletonBean s2 = context.getBean(SingletonBean.class);
        System.out.println("s1 == s2 ? " + (s1 == s2)); // true

        System.out.println("--- Prototype test ---");
        PrototypeBean p1 = context.getBean(PrototypeBean.class);
        PrototypeBean p2 = context.getBean(PrototypeBean.class);
        System.out.println("p1 == p2 ? " + (p1 == p2)); // false
    }
}
```

**Expected output (note WHEN each bean is created):**
```
SingletonBean created: com.example.SingletonBean@1b6d3586   <- created eagerly at startup
--- Singleton test ---
s1 == s2 ? true
--- Prototype test ---
PrototypeBean created: com.example.PrototypeBean@4b1210ee    <- created on 1st getBean() call
PrototypeBean created: com.example.PrototypeBean@4eec7777    <- created on 2nd getBean() call
p1 == p2 ? false
```

Prototype beans are **always lazy** — the container never creates them
ahead of time, only when actually requested.

## ⚠️ The classic gotcha: injecting a Prototype bean into a Singleton

```java
@Component
public class SingletonService {

    @Autowired
    private PrototypeBean prototypeBean; // ⚠️ injected ONCE, at SingletonService's creation

    public void printBean() {
        System.out.println(prototypeBean); // ALWAYS the SAME instance!
    }
}
```

Even though `PrototypeBean` is `prototype`-scoped, because it's injected
directly into a **singleton** field, Spring wires it **only once** — when
the singleton is created. So every call to `printBean()` prints the same
instance, defeating the purpose of prototype scope.

### ✅ Fix: use `ObjectProvider` (or `ApplicationContext.getBean()`, or a lookup method) to get a fresh instance every time

```java
@Component
public class SingletonServiceFixed {

    @Autowired
    private ObjectProvider<PrototypeBean> prototypeBeanProvider;

    public void printBean() {
        PrototypeBean freshBean = prototypeBeanProvider.getObject(); // new instance each call
        System.out.println(freshBean);
    }
}
```

## 🎯 Interview Questions & Answers

**Q1. What is the default bean scope in Spring?**
> `singleton` — one shared instance per Spring container.

**Q2. Is Spring's singleton the same as the Gang-of-Four Singleton pattern?**
> No. GoF Singleton = one instance per **JVM/classloader**, usually enforced
> via a private constructor + static instance. Spring singleton = one
> instance **per Spring container (`ApplicationContext`)** — you can still
> have multiple containers in the same JVM, each with its own "singleton."

**Q3. Why is directly injecting a prototype bean into a singleton bean a problem?**
> Because the singleton is created once, DI happens once, so the prototype
> field is set once and never refreshed — you lose the "new instance every
> time" behavior of prototype scope.

**Q4. How do you correctly get a fresh prototype instance inside a singleton?**
> Use `ObjectProvider<T>` (modern, recommended), method injection via
> `@Lookup`, or inject the `ApplicationContext` itself and call
> `context.getBean(PrototypeBean.class)` on demand (less clean, more
> coupling to Spring API).

**Q5. Does Spring call `@PreDestroy` on prototype beans?**
> No — Spring manages the **full lifecycle** of singleton beans (including
> destruction), but for prototype beans it hands off the instance and
> **does not track it further** — so destroy callbacks are never invoked
> automatically. You're responsible for any cleanup.


  bean created, and does that match how I expect to use it?"
- `@Scope("prototype")` beans are NOT managed for destruction by the
  container — if they hold resources (files, connections), you must close
  them yourself.
