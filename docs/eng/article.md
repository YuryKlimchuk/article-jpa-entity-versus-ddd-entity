# JPA Entity != DDD Entity

## Introduction

Most Java developers believe that a DDD Entity is simply a JPA Entity with a few business methods added on top. That's exactly how the vast majority of examples on the internet present it. Unfortunately, this misconception is often where the gradual degradation of a domain model begins.

When you first start learning Domain-Driven Design, one of the earliest traps you'll encounter is confusing a DDD Entity with a JPA Entity. Almost every tutorial uses the same class for both: it is simultaneously the domain model and the database representation. It looks convenient, but in reality, it's a serious mistake.

The problem isn't obvious at first. As the project grows, however, the domain inevitably becomes polluted with infrastructure concerns, invariants become weaker, and refactoring turns into a nightmare. Let's take a closer look at why JPA Entities and DDD Entities are fundamentally different concepts—and why combining them into a single class is a bad idea.

## What Is a JPA Entity, Really?

A JPA Entity is a projection of a relational database table into an object in your application. It is purely a persistence model whose sole responsibility is to read and write data.

Here's a typical example:

```java
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue
    private Long id;

    @Column(name = "status")
    private String status;

    @Column(name = "total")
    private BigDecimal total;

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }

    public BigDecimal getTotal() { return total; }
    public void setTotal(BigDecimal total) { this.total = total; }
}
```

This is an anemic model: getters, setters, and no behavior. It is entirely driven by the rules of the JPA provider—`@Entity`, `@Table`, and `@Column` define how the class is mapped to the database. There is no place for business rules here, only data and access to it.

JPA solves a purely infrastructure problem: converting objects to database rows and back again. Its purpose is to minimize the amount of persistence code you have to write. DDD, on the other hand, addresses a completely different concern—modeling business behavior. When a single class tries to solve both problems at the same time, it ends up depending on two entirely different sets of requirements.

## What Is a DDD Entity, Really?

According to Eric Evans in *Domain-Driven Design* (2003), an Entity is a tactical pattern that represents an object defined primarily by its identity rather than its attributes. That identity persists over time and across different representations of the object. No matter how its attributes change, it is still the same object.

In practice, a DDD Entity is a plain Java object (POJO) with no dependency on frameworks. No `@Entity`, `@Table`, or `@Column` annotations—only business rules and invariants.

**Invariants** are conditions that must always hold true. An entity is responsible for protecting its own consistency and preventing external code from putting it into an invalid state.

**Business rules** belong inside the entity, not in external services. You shouldn't be able to change the `status` directly—you should have to go through a method that validates whether the transition is allowed.

Here's the same `Order`, but this time modeled as a DDD Entity:

```java
public class Order {
    private final OrderId id;
    private OrderStatus status;
    private Money total;
    private final List<OrderItem> items;

    public Order(OrderId id, List<OrderItem> items) {
        if (items == null || items.isEmpty()) {
            throw new IllegalArgumentException("Order must have at least one item");
        }
        this.id = id;
        this.items = new ArrayList<>(items);
        this.status = OrderStatus.NEW;
        this.total = calculateTotal();
    }

    public void markAsPaid() {
        if (status != OrderStatus.NEW) {
            throw new IllegalStateException("Only NEW orders can be marked as paid");
        }
        this.status = OrderStatus.PAID;
    }

    public void markAsShipped() {
        if (status != OrderStatus.PAID) {
            throw new IllegalStateException("Only PAID orders can be shipped");
        }
        this.status = OrderStatus.SHIPPED;
    }

    // Invariant: total must always equal the sum of all order items
    private Money calculateTotal() {
        return items.stream()
                .map(OrderItem::subTotal)
                .reduce(Money.ZERO, Money::add);
    }

    public OrderId id() { return id; }
    public OrderStatus status() { return status; }
    public Money total() { return total; }
    public List<OrderItem> items() { return Collections.unmodifiableList(items); }
}
```

What has changed compared to the JPA version?

- **No annotations** — it's a pure POJO with no dependency on infrastructure.
- **Invariants are protected** — the constructor guarantees that an order can never be created without at least one item.
- **Business rules live inside the entity** — `markAsPaid()` and `markAsShipped()` validate state transitions, so you can't simply call `setStatus()`.
- **Identity has its own type** — `OrderId` isn't just a database-generated `Long`; it's a Value Object with domain meaning.

Most importantly, this model can be tested **without a database**. No `@DataJpaTest`, no containers, no infrastructure. Just plain JUnit—fast, simple, and reliable.

```java
@Test
void shouldNotAllowShippingUnpaidOrder() {
    Order order = new Order(new OrderId(UUID.randomUUID()), List.of(anItem()));

    assertThrows(IllegalStateException.class, order::markAsShipped);
}
```

The moment you introduce a business rule into your entity, you can immediately write a unit test for it. No repositories, no services, no infrastructure.

With the traditional JPA-centric approach, testing the same rule often means implementing a repository first, then a service, and only then writing a test—most likely an integration test backed by a real database. That's three additional layers and significantly more effort.

Writing tests isn't most developers' favorite activity. The more complicated and time-consuming the process becomes, the easier it is to skip them altogether. A DDD Entity removes those barriers: writing a test takes seconds, running it takes seconds, and the likelihood that the test will actually be written increases dramatically.

## Why Mixing Them in the Same Class Is Dangerous

At some point, it becomes tempting to put everything from the DDD Entity into the JPA Entity. It may seem to work at first—but only for a while. Eventually, the problems begin to pile up.

### One Class, Two Masters

JPA and DDD have fundamentally different expectations.

JPA wants a no-argument constructor, getters and setters, and unrestricted field access for persistence and proxying.

DDD, on the other hand, wants encapsulation, a minimal public API, and strict protection of invariants.

When you try to satisfy both, you end up with something like this:

```java
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue
    private Long id;

    @Column(name = "status")
    @Enumerated(STRING)
    private OrderStatus status;

    @Column(name = "total")
    private BigDecimal total;

    // Required by JPA
    protected Order() {
    }

    // Intended for DDD
    public Order(List<OrderItem> items) {
        if (items == null || items.isEmpty()) {
            throw new IllegalArgumentException("Order must have at least one item");
        }
        this.status = OrderStatus.NEW;
    }

    // Business methods try to protect invariants...
    public void markAsPaid() {
        if (status != OrderStatus.NEW) {
            throw new IllegalStateException("Only NEW orders can be marked as paid");
        }
        this.status = OrderStatus.PAID;
    }

    // ...but setters are still publicly available
    public void setStatus(OrderStatus status) { this.status = status; }
    public void setTotal(BigDecimal total) { this.total = total; }

    // getters, getters, getters...
}
```

This class immediately suffers from several problems.

**Invariants are no longer protected.** Yes, you have `markAsPaid()`, but any developer can simply call `setStatus()` instead. The compiler won't complain. Static analysis won't complain. Your invariant is broken—and nobody notices.

**The domain now depends on infrastructure.** Annotations such as `@Entity`, `@Table`, and `@Column` define how the object is stored in the database. Change the database schema, and you may have to modify the domain model. Change the domain model, and you risk breaking persistence. The Single Responsibility Principle is violated because the class now has two completely different reasons to change.

**Testing business rules becomes more expensive.** To verify a business rule, you often end up needing an `EntityManager`, a transaction, and a database. Tests become slower, more complicated, and the temptation to skip them grows with every new business rule.

**It only works while the project is small.** At first it seems harmless—"it's only three fields, what's the big deal?" But business logic has a tendency to grow. Six months later, your entity contains dozens of methods, some state transitions happen through business methods, others through `setStatus()` in a service, and untangling that mess becomes nearly impossible.

## Conclusion

JPA Entities and DDD Entities may look similar at first glance, but they solve fundamentally different problems.

A JPA Entity describes **how data is stored**.

A DDD Entity describes **how the business behaves**.

When a project is small, combining these two responsibilities into a single class feels convenient—even natural. But as the system grows, that convenience becomes increasingly expensive. The domain model becomes coupled to infrastructure, invariants are violated more frequently, and testing and refactoring require ever more effort.

This doesn't mean every project must separate the domain model from the persistence model. For a simple CRUD application, using a single JPA Entity can be a perfectly reasonable trade-off.

However, if you're building a system with genuinely complex business logic, it's worth drawing a clear line between the domain model and the persistence model as early as possible.

A simple rule of thumb is this:

> If you find yourself adding annotations, setters, or other elements to a domain entity solely because your ORM requires them, you're probably mixing two different responsibilities.

DDD doesn't discourage the use of JPA.

It simply reminds us not to confuse the persistence mechanism with the domain model.

And the more complex your business becomes, the more valuable that separation turns out to be.

> JPA is a way to persist data.
>
> DDD is a way to model business.

As long as a single class tries to do both, everything seems simple. But as the business grows more complex, that simplicity turns out to be temporary.

That's why experienced DDD teams separate the persistence model from the domain model—not because they're obsessed with "clean architecture," but because it allows the system to remain understandable, maintainable, and adaptable as it evolves.
