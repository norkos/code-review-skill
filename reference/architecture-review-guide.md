# Architecture Review Guide

A guide to reviewing architecture and design, helping assess whether the code's architecture is sound and its design appropriate.

## SOLID Principles Checklist

### S - Single Responsibility Principle (SRP)

**What to check:**
- Does this class/module have only one reason to change?
- Do all of the class's methods serve the same purpose?
- Could you describe this class to a non-technical person in one sentence?

**Signals to spot during code review:**
```
⚠️ Class name contains generic words like "And", "Manager", "Handler", "Processor"
⚠️ A class exceeds 200-300 lines of code
⚠️ A class has more than 5-7 public methods
⚠️ Different methods operate on completely different data
```

**Review questions:**
- "What is this class responsible for? Can it be split?"
- "If requirement X changes, which methods need to change? What if requirement Y changes?"

### O - Open/Closed Principle (OCP)

**What to check:**
- Does adding a new feature require modifying existing code?
- Can new behavior be added through extension (inheritance, composition)?
- Are there many if/else or switch statements handling different types?

**Signals to spot during code review:**
```
⚠️ switch/if-else chains handling different types
⚠️ Adding a new feature requires modifying core classes
⚠️ Type checks (instanceof, typeof) scattered throughout the code
```

**Review questions:**
- "To add a new type of X, which files need to change?"
- "Will this switch statement keep growing as new types are added?"

### L - Liskov Substitution Principle (LSP)

**What to check:**
- Can a subclass fully replace its parent class?
- Does a subclass change the expected behavior of a parent method?
- Does a subclass throw exceptions the parent class doesn't declare?

**Signals to spot during code review:**
```
⚠️ Explicit type casting
⚠️ Subclass methods throw NotImplementedException
⚠️ Subclass methods are empty or only contain return
⚠️ Code using the base class has to check the concrete type
```

**Review questions:**
- "If the parent class is replaced with the subclass, does the calling code need to change?"
- "Does this method's behavior in the subclass honor the parent class's contract?"

### I - Interface Segregation Principle (ISP)

**What to check:**
- Is the interface small and focused enough?
- Are implementing classes forced to implement methods they don't need?
- Do clients depend on methods they don't use?

**Signals to spot during code review:**
```
⚠️ Interface has more than 5-7 methods
⚠️ Implementing classes have empty methods or throw NotImplementedException
⚠️ Interface names are too broad (IManager, IService)
⚠️ Different clients use only part of the interface's methods
```

**Review questions:**
- "Is every method of this interface used by every implementing class?"
- "Can this large interface be split into smaller, specialized interfaces?"

### D - Dependency Inversion Principle (DIP)

**What to check:**
- Do high-level modules depend on abstractions rather than concrete implementations?
- Is dependency injection used instead of directly instantiating objects with new?
- Are abstractions defined by high-level modules rather than low-level ones?

**Signals to spot during code review:**
```
⚠️ High-level modules directly new up concrete classes from low-level modules
⚠️ Importing concrete implementation classes instead of interfaces/abstract classes
⚠️ Configuration and connection strings hardcoded in business logic
⚠️ A class is hard to unit test
```

**Review questions:**
- "Can this class's dependencies be replaced with mocks during testing?"
- "If the database/API implementation is swapped out, how many places need to change?"

---

## Identifying Architectural Anti-Patterns

### Critical Anti-Patterns

| Anti-Pattern | Signals | Impact |
|--------|----------|------|
| **Big Ball of Mud** | No clear module boundaries; any code may call any other code | Hard to understand, modify, and test |
| **God Object** | A single class takes on too many responsibilities, knows too much, does too much | High coupling, hard to reuse and test |
| **Spaghetti Code** | Tangled control flow, goto or deep nesting, hard to trace execution paths | Hard to understand and maintain |
| **Lava Flow** | Old code nobody dares touch, lacking documentation and tests | Accumulating technical debt |

### Design Anti-Patterns

| Anti-Pattern | Signals | Recommendation |
|--------|----------|------|
| **Golden Hammer** | Using the same technology/pattern for every problem | Choose a solution suited to the problem |
| **Over-Engineering (Gas Factory)** | Complex solutions to simple problems, design pattern abuse | YAGNI: start simple, add complexity later |
| **Boat Anchor** | Unused code written because it "might be needed someday" | Delete unused code; write it when it's needed |
| **Copy-Paste Programming** | The same logic appears in multiple places | Extract a shared method or module |

### Review Questions

```markdown
🔴 [blocking] "This class has 2000 lines of code; consider splitting it into several focused classes"
🟡 [important] "This logic is duplicated in 3 places; consider extracting a shared method?"
💡 [suggestion] "This switch statement could be replaced with the strategy pattern for easier extension"
```

---

## Coupling and Cohesion Assessment

### Coupling Types (Best to Worst)

| Type | Description | Example |
|------|------|------|
| **Message Coupling** ✅ | Data passed via parameters | `calculate(price, quantity)` |
| **Data Coupling** ✅ | Sharing simple data structures | `processOrder(orderDTO)` |
| **Stamp Coupling** ⚠️ | Sharing a complex data structure but using only part of it | Passing a whole User object but only using name |
| **Control Coupling** ⚠️ | Passing control flags that affect behavior | `process(data, isAdmin=true)` |
| **Common Coupling** ❌ | Sharing global variables | Multiple modules reading and writing the same global state |
| **Content Coupling** ❌ | Directly accessing another module's internals | Directly manipulating another class's private properties |

### Cohesion Types (Best to Worst)

| Type | Description | Quality |
|------|------|------|
| **Functional Cohesion** | All elements accomplish a single task | ✅ Best |
| **Sequential Cohesion** | Output of one step is the input of the next | ✅ Good |
| **Communicational Cohesion** | Operate on the same data | ⚠️ Acceptable |
| **Temporal Cohesion** | Tasks executed at the same time | ⚠️ Poor |
| **Logical Cohesion** | Logically related but functionally different | ❌ Bad |
| **Coincidental Cohesion** | No obvious relationship | ❌ Worst |

### Metrics Reference

```yaml
Coupling metrics:
  CBO (Coupling Between Objects):
    Good: < 5
    Warning: 5-10
    Danger: > 10

  Ce (Efferent Coupling):
    Description: how many external classes it depends on
    Good: < 7

  Ca (Afferent Coupling):
    Description: how many classes depend on it
    High value means: changes have wide impact; must be stable

Cohesion metrics:
  LCOM4 (Lack of Cohesion of Methods):
    1: Single responsibility ✅
    2-3: May need splitting ⚠️
    >3: Should be split ❌
```

### Review Questions

- "How many other modules does this module depend on? Can that be reduced?"
- "How many other places would be affected by changing this class?"
- "Do all of this class's methods operate on the same data?"

---

## Layered Architecture Review

### Clean Architecture Layer Check

```
┌─────────────────────────────────────┐
│         Frameworks & Drivers        │ ← Outermost layer: Web, DB, UI
├─────────────────────────────────────┤
│         Interface Adapters          │ ← Controllers, Gateways, Presenters
├─────────────────────────────────────┤
│          Application Layer          │ ← Use Cases, Application Services
├─────────────────────────────────────┤
│            Domain Layer             │ ← Entities, Domain Services
└─────────────────────────────────────┘
          ↑ Dependencies point inward only ↑
```

### Dependency Rule Check

**Core rule: source code dependencies may only point inward**

```typescript
// ❌ Violates the dependency rule: Domain layer depends on Infrastructure
// domain/User.ts
import { MySQLConnection } from '../infrastructure/database';

// ✅ Correct: Domain layer defines the interface, Infrastructure implements it
// domain/UserRepository.ts (interface)
interface UserRepository {
  findById(id: string): Promise<User>;
}

// infrastructure/MySQLUserRepository.ts (implementation)
class MySQLUserRepository implements UserRepository {
  findById(id: string): Promise<User> { /* ... */ }
}
```

### Review Checklist

**Layer boundary checks:**
- [ ] Does the Domain layer have external dependencies (database, HTTP, file system)?
- [ ] Does the Application layer access the database or call external APIs directly?
- [ ] Do Controllers contain business logic?
- [ ] Are there cross-layer calls (UI calling a Repository directly)?

**Separation of concerns checks:**
- [ ] Is business logic separated from presentation logic?
- [ ] Is data access encapsulated in a dedicated layer?
- [ ] Is configuration and environment-specific code managed centrally?

### Review Questions

```markdown
🔴 [blocking] "The Domain entity directly imports a database connection, violating the dependency rule"
🟡 [important] "The Controller contains business calculation logic; consider moving it to the Service layer"
💡 [suggestion] "Consider using dependency injection to decouple these components"
```

---

## Design Pattern Usage Assessment

### When to Use Design Patterns

| Pattern | Good Fit | Poor Fit |
|------|----------|------------|
| **Factory** | Need to create objects of different types, with the type determined at runtime | Only one type, or the type never changes |
| **Strategy** | Algorithms need to be switched at runtime; multiple interchangeable behaviors | Only one algorithm, or the algorithm never changes |
| **Observer** | One-to-many dependency; state changes must notify multiple objects | A simple direct call meets the need |
| **Singleton** | A truly global unique instance is needed, e.g., configuration management | Objects that can be passed via dependency injection |
| **Decorator** | Need to add responsibilities dynamically and avoid an inheritance explosion | Responsibilities are fixed; no dynamic composition needed |

### Over-Engineering Warning Signs

```
⚠️ Signs of Patternitis:

1. A simple if/else replaced with strategy pattern + factory + registry
2. Interfaces with only one implementation
3. Abstraction layers added because they "might be needed someday"
4. Line count grows significantly due to applying patterns
5. Newcomers take a long time to understand the code structure
```

### Review Principles

```markdown
✅ Proper use of patterns:
- Solves a real extensibility problem
- Code becomes easier to understand and test
- Adding new features becomes simpler

❌ Overuse of patterns:
- Using a pattern for its own sake
- Adds unnecessary complexity
- Violates the YAGNI principle
```

### Review Questions

- "What concrete problem does using this pattern solve?"
- "What problems would the code have without this pattern?"
- "Does the value of this abstraction layer outweigh its complexity?"

---

## Extensibility Assessment

### Extensibility Checklist

**Feature extensibility:**
- [ ] Does adding a new feature require modifying core code?
- [ ] Are extension points provided (hooks, plugins, events)?
- [ ] Is configuration externalized (config files, environment variables)?

**Data extensibility:**
- [ ] Does the data model support adding new fields?
- [ ] Has data volume growth been considered?
- [ ] Do queries have appropriate indexes?

**Load scalability:**
- [ ] Can it scale horizontally (add more instances)?
- [ ] Are there stateful dependencies (session, local cache)?
- [ ] Do database connections use a connection pool?

### Extension Point Design Check

```typescript
// ✅ Good extensible design: use events/hooks
class OrderService {
  private hooks: OrderHooks;

  async createOrder(order: Order) {
    await this.hooks.beforeCreate?.(order);
    const result = await this.save(order);
    await this.hooks.afterCreate?.(result);
    return result;
  }
}

// ❌ Poor extensible design: all behavior hardcoded
class OrderService {
  async createOrder(order: Order) {
    await this.sendEmail(order);        // hardcoded
    await this.updateInventory(order);  // hardcoded
    await this.notifyWarehouse(order);  // hardcoded
    return await this.save(order);
  }
}
```

### Review Questions

```markdown
💡 [suggestion] "If a new payment method needs to be supported later, is this design easy to extend?"
🟡 [important] "This logic is hardcoded; consider using configuration or the strategy pattern?"
📚 [learning] "An event-driven architecture would make this feature easier to extend"
```

---

## Code Structure Best Practices

### Directory Organization

**Organize by feature/domain (recommended):**
```
src/
├── user/
│   ├── User.ts           (entity)
│   ├── UserService.ts    (service)
│   ├── UserRepository.ts (data access)
│   └── UserController.ts (API)
├── order/
│   ├── Order.ts
│   ├── OrderService.ts
│   └── ...
└── shared/
    ├── utils/
    └── types/
```

**Organize by technical layer (not recommended):**
```
src/
├── controllers/     ← Different domains mixed together
│   ├── UserController.ts
│   └── OrderController.ts
├── services/
├── repositories/
└── models/
```

### Naming Convention Check

| Type | Convention | Example |
|------|------|------|
| Class names | PascalCase, nouns | `UserService`, `OrderRepository` |
| Method names | camelCase, verbs | `createUser`, `findOrderById` |
| Interface names | I prefix or no prefix | `IUserService` or `UserService` |
| Constants | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| Private properties | Underscore prefix or none | `_cache` or `#cache` |

### File Size Guidelines

```yaml
Recommended limits:
  Single file: < 300 lines
  Single function: < 50 lines
  Single class: < 200 lines
  Function parameters: < 4
  Nesting depth: < 4 levels

When limits are exceeded:
  - Consider splitting into smaller units
  - Prefer composition over inheritance
  - Extract helper functions or classes
```

### Review Questions

```markdown
🟢 [nit] "This 500-line file could be split by responsibility"
🟡 [important] "Consider organizing the directory structure by feature domain rather than technical layer"
💡 [suggestion] "The function name `process` is not specific enough; consider renaming it to `calculateOrderTotal`?"
```

---

## Quick Reference Checklist

### 5-Minute Architecture Review

```markdown
□ Is the dependency direction correct? (outer layers depend on inner layers)
□ Are there circular dependencies?
□ Is core business logic decoupled from framework/UI/database?
□ Are SOLID principles followed?
□ Are there obvious anti-patterns?
```

### Red Flags (Must Fix)

```markdown
🔴 God Object - a single class exceeds 1000 lines
🔴 Circular dependency - A → B → C → A
🔴 Domain layer contains framework dependencies
🔴 Hardcoded configuration and secrets
🔴 External service calls without an interface
```

### Yellow Flags (Should Fix)

```markdown
🟡 Coupling Between Objects (CBO) > 10
🟡 More than 5 method parameters
🟡 Nesting depth exceeds 4 levels
🟡 Duplicated code blocks > 10 lines
🟡 Interfaces with only one implementation
```

---

## Recommended Tools

| Tool | Purpose | Language Support |
|------|------|----------|
| **SonarQube** | Code quality, coupling analysis | Multi-language |
| **NDepend** | Dependency analysis, architecture rules | .NET |
| **JDepend** | Package dependency analysis | Java |
| **Madge** | Module dependency graphs | JavaScript/TypeScript |
| **ESLint** | Code style, complexity checks | JavaScript/TypeScript |
| **CodeScene** | Technical debt, hotspot analysis | Multi-language |

---

## References

- [Clean Architecture - Uncle Bob](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [SOLID Principles in Code Review - JetBrains](https://blog.jetbrains.com/upsource/2015/08/31/what-to-look-for-in-a-code-review-solid-principles-2/)
- [Software Architecture Anti-Patterns](https://medium.com/@christophnissle/anti-patterns-in-software-architecture-3c8970c9c4f5)
- [Coupling and Cohesion in System Design](https://www.geeksforgeeks.org/system-design/coupling-and-cohesion-in-system-design/)
- [Design Patterns - Refactoring Guru](https://refactoring.guru/design-patterns)
