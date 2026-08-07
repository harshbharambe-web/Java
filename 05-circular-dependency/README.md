# 05 — Circular Dependency

## 🤔 What is it

Bean A needs Bean B, and Bean B needs Bean A — a **cycle** in the
dependency graph.

```
        ┌───────────┐          ┌───────────┐
        │  Bean A     │ ───────► │  Bean B     │
        │             │ ◄─────── │             │
        └───────────┘          └───────────┘
           needs B                needs A
```

This is usually a **design smell** (tight coupling / unclear responsibility
boundaries), but Spring can resolve it in some cases and not others.

## ⚙️ Does Spring resolve it? Depends on injection type.

```
┌─────────────────────┬───────────────────────────────────────────────────┐
│ Injection type         │ Can Spring resolve the circular dependency?          │
├─────────────────────┼───────────────────────────────────────────────────┤
│ Field / Setter          │ ✅ YES (for singleton scope) — via 3-level cache      │
│ Constructor             │ ❌ NO — throws BeanCurrentlyInCreationException      │
└─────────────────────┴───────────────────────────────────────────────────┘
```

### Why setter/field injection CAN work: the 3-level cache

Spring creates singleton beans in two logical steps:
1. **Instantiate** (call constructor) → raw, empty object
2. **Populate** (inject dependencies via setter/field) + initialize

Because instantiation and dependency-injection are **separate steps**,
Spring can hand out a **half-built, raw instance** during step 1 (before its
own dependencies are set), let the other bean use it temporarily, and fill
in the dependencies later in step 2.

```
Creating Bean A:
  1. Instantiate raw A (no dependencies yet) → put in cache
  2. Needs to inject B into A → go create B
       Creating Bean B:
         1. Instantiate raw B → put in cache
         2. Needs to inject A into B → check cache → FOUND raw A! → use it
         3. B is now fully built
  3. Back to A: inject the now-fully-built B into A
  4. A is now fully built
```

This "early exposure" of a raw bean via a cache is only possible because
setter/field injection happens **after** the object already exists.
Constructor injection needs **all dependencies at construction time itself**
— there's no "raw object" step to expose early, so the cycle can't be
broken. Hence: ❌ for constructor injection.

## 💻 Code

### ✅ Works: Setter Injection

```java
@Component
public class ServiceA {

    private ServiceB serviceB;

    @Autowired
    public void setServiceB(ServiceB serviceB) {
        this.serviceB = serviceB;
    }

    public void doSomething() {
        System.out.println("ServiceA using " + serviceB);
    }
}
```

```java
@Component
public class ServiceB {

    private ServiceA serviceA;

    @Autowired
    public void setServiceA(ServiceA serviceA) {
        this.serviceA = serviceA;
    }

    public void doSomethingElse() {
        System.out.println("ServiceB using " + serviceA);
    }
}
```

```java
@Configuration
@ComponentScan(basePackages = "com.example")
public class MainApp {
    public static void main(String[] args) {
        var context = new AnnotationConfigApplicationContext(MainApp.class);
        ServiceA a = context.getBean(ServiceA.class);
        a.doSomething(); // works fine
    }
}
```

**Output:**
```
ServiceA using com.example.ServiceB@...
```
No error — Spring quietly resolved the cycle using the early-exposure cache.

### ❌ Fails: Constructor Injection

```java
@Component
public class ServiceX {
    private final ServiceY serviceY;

    @Autowired
    public ServiceX(ServiceY serviceY) { // needs Y to even exist
        this.serviceY = serviceY;
    }
}
```

```java
@Component
public class ServiceY {
    private final ServiceX serviceX;

    @Autowired
    public ServiceY(ServiceX serviceX) { // needs X to even exist
        this.serviceX = serviceX;
    }
}
```

Running this throws:
```
BeanCurrentlyInCreationException:
Error creating bean with name 'serviceX': Requested bean is currently in
creation: Is there an unresolvable circular reference?
```

