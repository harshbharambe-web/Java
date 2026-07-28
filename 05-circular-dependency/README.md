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

## 🛠 How to actually fix a circular dependency (not just work around it)

1. **Best fix: redesign** — extract the shared logic both beans need into a
   third bean/class, so neither depends on the other directly.
2. **`@Lazy` on one of the constructor params** — delays resolution of that
   dependency until it's first *used*, not at construction. Works but is a
   band-aid, not a real fix:
   ```java
   public ServiceX(@Lazy ServiceY serviceY) { this.serviceY = serviceY; }
   ```
3. **Switch to setter/field injection** — lets Spring's cache mechanism
   handle it. Also a band-aid — you're just letting Spring hide the design
   smell instead of fixing it.

## 🎯 Interview Questions & Answers

**Q1. Can Spring resolve circular dependencies?**
> Yes, but only for **singleton-scoped beans using setter or field
> injection**, via early exposure of a partially-constructed bean using its
> internal 3-level cache. It **cannot** resolve circular dependencies for
> **constructor injection** or **prototype-scoped** beans.

**Q2. Why can't constructor injection resolve circular dependency?**
> Because constructor injection requires the dependency to be **fully
> available at the moment of instantiation** — there's no "raw, half-built"
> object to expose early like there is with setter/field injection.

**Q3. Is having a circular dependency a sign of bad design?**
> Generally yes — it usually means two classes are too tightly coupled or a
> responsibility is split awkwardly between them. `@Lazy` is a quick
> workaround, not a design fix; the real fix is usually to introduce a
> third class/service that both depend on.

