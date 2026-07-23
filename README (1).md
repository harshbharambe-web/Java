# Java Spring Learning 🌱

My day-by-day notes and code while learning Spring / Spring Boot core concepts.
Same style as my DSA repo — one folder per topic, each with a `README.md`
that has: simple explanation → diagram → runnable code → interview Q&A → gotchas/trending tips.

Goal: not just "learn Spring" but be able to **explain it in an interview** and
**debug it in real projects**.

---

## 📁 How this repo is organized

```
java-spring-learning/
│
├── 01-dependency-injection/     ✅ done
├── 02-ioc-container/            ✅ done
├── 03-bean-annotations/         ✅ done
├── 04-bean-scopes/              ✅ done
├── 05-circular-dependency/      ✅ done
├── 06-bean-lifecycle/           🔄 learning now
├── 07-...                       ⏳ upcoming
└── README.md                    (this file)
```

Each topic folder has:
- `README.md` → theory, diagram, interview questions
- `src/` (or plain `.java` files) → runnable demo code with comments

Numbering = the order I learned things, not necessarily the "correct" teaching
order (IoC actually enables DI, but I've cross-linked them below so it reads
fine either way).

---

## ✅ Progress Tracker

| # | Topic | Status | Link |
|---|-------|--------|------|
| 1 | Dependency Injection (DI) | ✅ Done | [01-dependency-injection](./01-dependency-injection) |
| 2 | IoC & IoC Container | ✅ Done | [02-ioc-container](./02-ioc-container) |
| 3 | Bean Annotations (`@Component`, `@Bean`, `@Autowired`, `@Qualifier`, `@Primary`...) | ✅ Done | [03-bean-annotations](./03-bean-annotations) |
| 4 | Bean Scopes (singleton, prototype, request, session...) | ✅ Done | [04-bean-scopes](./04-bean-scopes) |
| 5 | Circular Dependency | ✅ Done | [05-circular-dependency](./05-circular-dependency) |
| 6 | Bean Lifecycle | 🔄 In progress | [06-bean-lifecycle](./06-bean-lifecycle) |
| 7 | AOP (Aspect Oriented Programming) | ⏳ Upcoming | — |
| 8 | Spring Boot Auto-configuration | ⏳ Upcoming | — |
| 9 | Spring MVC / REST | ⏳ Upcoming | — |
| 10 | Spring Data JPA | ⏳ Upcoming | — |
| 11 | Spring Security | ⏳ Upcoming | — |
| 12 | Transactions (`@Transactional`) | ⏳ Upcoming | — |

---

## 🧠 The Big Picture (how these topics connect)

```
                ┌─────────────────────────────┐
                │   IoC (a principle/idea)     │
                │  "Don't create objects       │
                │   yourself, let the           │
                │   framework do it"           │
                └───────────────┬──────────────┘
                                │ implemented by
                                ▼
                ┌─────────────────────────────┐
                │  IoC Container (ApplicationContext) │
                │  reads config → creates → wires beans│
                └───────────────┬──────────────┘
                                │ uses
                                ▼
                ┌─────────────────────────────┐
                │   Dependency Injection (DI)   │
                │  "how" objects get their      │
                │   dependencies injected       │
                └───────────────┬──────────────┘
                     ┌──────────┼──────────┐
                     ▼          ▼          ▼
              @Component   Bean Scope   Bean Lifecycle
              @Bean etc.   (singleton,  (instantiate →
              (WHAT is a   prototype..) populate →
              bean)        (HOW MANY    init → use →
                           instances)   destroy)
                     │
                     ▼
              Circular Dependency
              (a PROBLEM that happens
               during DI/wiring)
```

**One-line summary of each (interview soundbite):**
- **IoC** = a design principle: control of object creation is inverted, from your code to the framework.
- **IoC Container** = the actual engine (`ApplicationContext`) that implements IoC.
- **DI** = the technique the container uses to give a bean its dependencies (constructor/setter/field injection).
- **Bean Annotations** = how you tell Spring "this class/method should produce a bean."
- **Bean Scope** = how many instances of a bean exist and how long they live.
- **Bean Lifecycle** = the exact sequence of steps Spring goes through to create, configure, use, and destroy a bean.
- **Circular Dependency** = a wiring problem where two beans need each other, and how Spring resolves (or fails to resolve) it.

---

## 🔥 Trending / interview-favorite topics in this repo

These come up **very often** in Java backend interviews (product companies + service companies):

- Difference between `@Component`, `@Service`, `@Repository`, `@Controller` (all are `@Component` under the hood, but semantically different + some add extra behavior like exception translation for `@Repository`)
- Constructor injection vs field injection — **why constructor injection is preferred** (immutability, easier testing, avoids circular dependency issues, fails fast)
- `@Autowired` + `@Qualifier` + `@Primary` — how Spring resolves multiple beans of the same type
- Singleton vs Prototype scope — and the **classic gotcha**: injecting a prototype bean into a singleton bean
- Circular dependency — how Spring solves it for singleton + setter injection using the **3-level cache**, and why it **cannot** solve it for pure constructor injection
- Bean lifecycle — `@PostConstruct`/`@PreDestroy` vs `InitializingBean`/`DisposableBean` vs `init-method`/`destroy-method`, and **the exact order** they run in if you use more than one

---

## 🛠 How to run the code in each folder

Each demo is a small plain Spring (non-Boot, unless mentioned) console app so it's quick to run and see the logs (which is where the real learning is — **watch the console output**).

```bash
# example for any topic folder
cd 01-dependency-injection
javac -cp ".:libs/*" *.java -d out
java -cp ".:libs/*:out" MainApp
```

Or drop the files into a Spring Boot project (`spring-initializr` → Web dependency is enough)
and just run the `main` method / `@SpringBootApplication` class.

---

## 📌 Notes to self

- Always read the **console output order** for lifecycle/circular-dependency demos — the explanation only clicks once you see the actual print order.
- Diagrams in each folder's README are intentionally simple (boxes + arrows) — not fancy, just enough to visualize flow.
- Update the progress table above every time a new folder is added.
