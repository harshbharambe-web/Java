# 06 — Bean Lifecycle 🔄 (learning now)

## 🤔 What is Bean Lifecycle

The exact **sequence of steps** Spring performs on a bean from the moment
it's created to the moment it's destroyed — instantiation → dependency
injection → custom init logic → ready to use → custom cleanup logic on
shutdown.

Knowing this order matters because it tells you **exactly when** it's safe
to use injected dependencies, and **where** to hook custom setup/teardown
logic (opening a DB connection pool, warming a cache, closing resources).

## 📊 The full lifecycle, in order

```
1. Bean instantiated (constructor called)
        │
2. Dependencies injected (setter/field injection happens here;
   NOTE: constructor injection already happened in step 1)
        │
3. Aware interfaces called, if implemented
   (BeanNameAware, BeanFactoryAware, ApplicationContextAware...)
        │
4. BeanPostProcessor#postProcessBeforeInitialization (for ALL beans)
        │
5. @PostConstruct method called
        │
6. InitializingBean#afterPropertiesSet() called
        │
7. custom init-method (if specified via @Bean(initMethod=...) or XML)
        │
8. BeanPostProcessor#postProcessAfterInitialization (for ALL beans)
        │
   ══════════ BEAN IS NOW FULLY READY / IN USE ══════════
        │
9. (application runs, bean is used)
        │
   ══════════ CONTAINER SHUTTING DOWN ══════════
        │
10. @PreDestroy method called
        │
11. DisposableBean#destroy() called
        │
12. custom destroy-method (if specified via @Bean(destroyMethod=...) or XML)
```

**Key ordering to remember for interviews (steps 5→7 and 10→12):**
```
INIT:     @PostConstruct  →  afterPropertiesSet()  →  custom init-method
DESTROY:  @PreDestroy     →  destroy()              →  custom destroy-method
```
Same relative order both times: **annotation → interface → XML/config-style**.

## 💻 Code — seeing every step fire, in order

### `LifecycleBean.java` — using ALL three mechanisms at once (for learning; never do this in real code — pick ONE)

```java
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import org.springframework.beans.factory.BeanNameAware;
import org.springframework.beans.factory.DisposableBean;
import org.springframework.beans.factory.InitializingBean;

public class LifecycleBean implements BeanNameAware, InitializingBean, DisposableBean {

    public LifecycleBean() {
        System.out.println("1. Constructor called - bean instantiated");
    }

    // --- Aware interface ---
    @Override
    public void setBeanName(String name) {
        System.out.println("2. BeanNameAware -> setBeanName: " + name);
    }

    // --- Init callbacks, in the order Spring actually calls them ---
    @PostConstruct
    public void postConstruct() {
        System.out.println("3. @PostConstruct called");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("4. InitializingBean -> afterPropertiesSet()");
    }

    public void customInit() {
        System.out.println("5. custom init-method called");
    }

    // --- Destroy callbacks, in the order Spring actually calls them ---
    @PreDestroy
    public void preDestroy() {
        System.out.println("6. @PreDestroy called");
    }

    @Override
    public void destroy() {
        System.out.println("7. DisposableBean -> destroy()");
    }

    public void customDestroy() {
        System.out.println("8. custom destroy-method called");
    }
}
```

### `AppConfig.java` — registering the bean with init/destroy methods

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig {

    @Bean(initMethod = "customInit", destroyMethod = "customDestroy")
    public LifecycleBean lifecycleBean() {
        return new LifecycleBean();
    }
}
```

### `MainApp.java`

```java
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class MainApp {
    public static void main(String[] args) {
        var context = new AnnotationConfigApplicationContext(AppConfig.class);

        System.out.println("--- Bean is ready, application running ---");

        context.close(); // triggers the full destroy sequence
    }
}
```

**Expected output (this exact order is the whole point of this topic):**
```
1. Constructor called - bean instantiated
2. BeanNameAware -> setBeanName: lifecycleBean
3. @PostConstruct called
4. InitializingBean -> afterPropertiesSet()
5. custom init-method called
--- Bean is ready, application running ---
6. @PreDestroy called
7. DisposableBean -> destroy()
8. custom destroy-method called
```

> ⚠️ `context.close()` only triggers destroy callbacks for **singleton**
> beans. Prototype beans are NOT tracked for destruction (see
> [04-bean-scopes](../04-bean-scopes) gotchas).

## 🧵 A real-world use case

```java
@Component
public class DbConnectionPool {

    private Connection connection;

    @PostConstruct
    public void openConnection() {
        System.out.println("Opening DB connection pool...");
        // connection = dataSource.getConnection();
    }

    @PreDestroy
    public void closeConnection() {
        System.out.println("Closing DB connection pool gracefully...");
        // connection.close();
    }
}
```
This is the #1 practical reason lifecycle callbacks exist: **resource setup
and graceful cleanup**, without manually calling init/cleanup methods
yourself everywhere.

## 🎯 Interview Questions & Answers

**Q1. What are the different ways to hook into a bean's init/destroy phase?**
> Three ways, and the exact call order (init) is:
> `@PostConstruct` → `InitializingBean.afterPropertiesSet()` → custom
> `init-method`. Destroy order is the mirror: `@PreDestroy` →
> `DisposableBean.destroy()` → custom `destroy-method`.

**Q2. Which mechanism should you actually use in real projects?**
> `@PostConstruct` / `@PreDestroy` (from `jakarta.annotation`) — they're
> simple, framework-agnostic (part of Java/Jakarta EE, not Spring-specific),
> and don't force your class to implement a Spring interface. Use
> `init-method`/`destroy-method` mainly for **third-party classes** you
> can't annotate directly (declared via `@Bean(initMethod=..., destroyMethod=...)`).

**Q3. When exactly does dependency injection finish relative to `@PostConstruct`?**
> DI is fully complete **before** `@PostConstruct` runs — that's the whole
> guarantee of that annotation: "by the time this method runs, all my
> `@Autowired` fields/constructor args are populated and safe to use."

**Q4. Are lifecycle callbacks called for prototype-scoped beans?**
> Init callbacks (`@PostConstruct` etc.) — yes, they run normally when the
> bean is created. Destroy callbacks — **no**, Spring hands off a prototype
> bean and stops tracking it, so `@PreDestroy`/`destroy()` are never
> invoked automatically for prototype beans.

**Q5. What's the difference between `BeanPostProcessor` and `@PostConstruct`?**
> `@PostConstruct` is defined **inside** a specific bean, for that bean
> only. A `BeanPostProcessor` is a separate, container-wide component that
> can intercept and modify **every bean** before and after initialization —
> it's how Spring itself implements features like `@PostConstruct`
> processing, AOP proxy creation, and validation, under the hood.

**Q6. What happens if you don't call `context.close()` (e.g. in a Spring Boot web app)?**
> In a normal Spring Boot app, a shutdown hook is registered automatically,
> so destroy callbacks fire when the JVM shuts down gracefully (Ctrl+C,
> `SIGTERM`, etc). You only need to call `.close()` manually in plain
> `AnnotationConfigApplicationContext` console apps like this demo.

## 🔥 Gotchas / Trending points

- **Never use all 3 mechanisms together** in real code (only done above for
  learning) — pick `@PostConstruct`/`@PreDestroy` and stick with it for
  consistency.
- `@PostConstruct`/`@PreDestroy` come from `jakarta.annotation.*` (or
  `javax.annotation.*` on older Spring versions) — **not** a Spring package,
  which is exactly why it's the recommended, portable choice.
- Order to remember cold for interviews:
  **Constructor → DI → Aware interfaces → BeanPostProcessor (before) →
  @PostConstruct → afterPropertiesSet() → init-method →
  BeanPostProcessor (after) → [bean in use] → @PreDestroy → destroy() →
  destroy-method.**
- A common trick question: *"If a bean implements both `InitializingBean`
  and has a `@PostConstruct` method, which runs first?"* → `@PostConstruct`
  always runs first, since annotation-driven processing (via a
  `BeanPostProcessor`) happens before the `InitializingBean` interface
  callback in Spring's internal bean creation code.
