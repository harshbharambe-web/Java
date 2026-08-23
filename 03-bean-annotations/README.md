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

`
