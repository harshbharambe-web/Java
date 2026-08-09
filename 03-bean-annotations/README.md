# 03 — Bean Annotations

## 🤔 What is a "Bean"

A **bean** is just an object that's created and managed by the Spring IoC
container (instead of you doing `new SomeClass()` yourself).

Annotations are how you **tell Spring** "this should become a bean" and
"here's how to inject it where needed."

## 🧩 The main annotations

```
┌───────────────┬──────────────────────────────────────────────────────────┐
│ Annotation      │ Purpose                                                    │
├───────────────┼──────────────────────────────────────────────────────────┤
│ @Component      │ Generic stereotype - marks a class as a Spring-managed bean │
│ @Service        │ Same as @Component, semantically = "business logic layer"  │
│ @Repository     │ Same as @Component + auto exception translation for DB     │
│                │ (DataAccessException wrapping)                              │
│ @Controller     │ Same as @Component, marks web layer (returns views)         │
│ @RestController │ @Controller + @ResponseBody combined (returns JSON/data)    │
│ @Bean           │ Used INSIDE a @Configuration class on a method - manually   │
│                │ tells Spring "the return value of this method is a bean"    │
│ @Autowired      │ Tells Spring to inject a matching bean here                 │
│ @Qualifier      │ Disambiguates between multiple beans of the same type       │
│ @Primary        │ Marks a bean as the default choice when multiple exist      │
└───────────────┴──────────────────────────────────────────────────────────┘
```

### Stereotype hierarchy
```
                @Component
                    │
      ┌─────────────┼─────────────┐
      ▼             ▼             ▼
  @Service     @Repository    @Controller ──► @RestController
 (business      (DAO/DB         (web layer)   (@Controller + @ResponseBody)
   logic)        layer)
```
All 4 are functionally `@Component` under the hood — Spring's classpath
scanning picks them all up the same way. The difference is **semantic
clarity** for readers + a couple of extra features (like `@Repository`'s
exception translation).

## 💻 Code

### `@Component` vs `@Bean`

```java
// Option 1: @Component - Spring creates the object itself via classpath scanning
@Component
public class NotificationService {
    public void notify(String msg) {
        System.out.println("Notifying: " + msg);
    }
}
```

```java
// Option 2: @Bean - YOU control object creation, useful for 3rd-party classes
// you don't own the source of (can't put @Component on them)
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig {

    @Bean
    public ObjectMapper objectMapper() {
        // e.g. Jackson's ObjectMapper - you don't own this class,
        // so @Component isn't an option. @Bean lets you register it manually.
        ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());
        return mapper;
    }
}
```

### `@Qualifier` and `@Primary` — resolving ambiguity

```java
public interface PaymentGateway {
    void pay(double amount);
}
```

```java
@Component
@Primary // default choice when no @Qualifier is specified
public class RazorpayGateway implements PaymentGateway {
    public void pay(double amount) {
        System.out.println("Paid via Razorpay: " + amount);
    }
}
```

```java
@Component
public class StripeGateway implements PaymentGateway {
    public void pay(double amount) {
        System.out.println("Paid via Stripe: " + amount);
    }
}
```

t.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan(basePackages = "com.example")
public class MainApp {
    public static void main(String[] args) {
        var context = new AnnotationConfigApplicationContext(MainApp.class);
        CheckoutService checkout = context.getBean(CheckoutService.class);
        checkout.checkout(999.0);
    }
}
```

**Expected output:**
```
Paid via Stripe: 999.0
```

## 🎯 Interview Questions & Answers

**Q1. Difference between `@Component` and `@Bean`?**
> `@Component` is a class-level annotation picked up automatically via
> classpath/component scanning — used for classes **you own**. `@Bean` is a
> method-level annotation inside a `@Configuration` class, used when you
> need manual control over object creation — especially for **third-party
> classes** you can't annotate directly (like `ObjectMapper`, `RestTemplate`).

**Q2. Difference between `@Component`, `@Service`, `@Repository`, `@Controller`?**
> All four register a class as a Spring bean identically under the hood.
> The differences are:
> - Semantic clarity (self-documenting code / layered architecture)
> - `@Repository` additionally enables automatic translation of
>   persistence-specific exceptions (like `SQLException`) into Spring's
>   unified `DataAccessException` hierarchy
> - `@Controller` is recognized specially by Spring MVC for request mapping

**Q3. What happens if you `@Autowired` an interface with 2+ implementations and don't use `@Qualifier`/`@Primary`?**
> Spring throws `NoUniqueBeanDefinitionException` at context startup — it
> can't decide which implementation to inject.

**Q4. `@Qualifier` vs `@Primary` — when to use which?**
> `@Primary` sets a **default** for when no other hint is given — good when
> one implementation is clearly the "usual" choice. `@Qualifier` is for
> **explicit, per-injection-point** selection — good when the choice varies
> by context. If both are present, `@Qualifier` wins (it's more specific).

**Q5. Can a `@Bean` method depend on another bean?**
> Yes — just add it as a method parameter; Spring auto-injects it:
> ```java
> @Bean
> public CheckoutService checkoutService(PaymentGateway gateway) {
>     return new CheckoutService(gateway);
> }
> ```

