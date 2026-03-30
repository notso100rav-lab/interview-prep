# Section 2: Spring & Spring Boot

**Estimated Time:** 18–24 hours  
**Difficulty:** Intermediate to Advanced  
**Format:** 190+ scenario-based Q&A with code examples

---

## What's Covered

This section covers the Spring ecosystem as experienced in real backend engineering roles. Each chapter presents scenario-based questions that mirror actual interview conversations at top-tier tech companies.

---

## Progress Checklist

| # | Chapter | Topics | Questions | Status |
|---|---------|---------|-----------|--------|
| 01 | [Spring Core & Dependency Injection](./01-spring-core-and-di.md) | @Component, @Autowired, injection types, circular deps | 20+ | ☐ |
| 02 | [Bean Lifecycle & Scopes](./02-bean-lifecycle-and-scopes.md) | @PostConstruct, @PreDestroy, singleton/prototype/request | 20+ | ☐ |
| 03 | [Spring Boot Auto-Configuration](./03-spring-boot-auto-configuration.md) | @SpringBootApplication, @Conditional, profiles, externalized config | 20+ | ☐ |
| 04 | [REST API Design & Validation](./04-rest-api-design-and-validation.md) | @RestController, @Valid, versioning, HATEOAS, content negotiation | 20+ | ☐ |
| 05 | [Exception Handling & @ControllerAdvice](./05-exception-handling-controller-advice.md) | @ExceptionHandler, RFC 7807, global error responses | 20+ | ☐ |
| 06 | [Transactions & Propagation](./06-transactions-and-propagation.md) | @Transactional, propagation types, isolation, self-invocation | 20+ | ☐ |
| 07 | [Hibernate & JPA Internals](./07-hibernate-and-jpa-internals.md) | EntityManager, entity states, dirty checking, relationships | 20+ | ☐ |
| 08 | [Lazy vs Eager Loading](./08-lazy-vs-eager-loading.md) | FetchType, LazyInitializationException, @EntityGraph | 20+ | ☐ |
| 09 | [The N+1 Problem](./09-n-plus-one-problem.md) | Detection, JOIN FETCH, @BatchSize, @EntityGraph solutions | 20+ | ☐ |
| 10 | [Caching & Performance Tuning](./10-caching-and-performance-tuning.md) | @Cacheable, providers, second-level cache, connection pools | 20+ | ☐ |
| 11 | [Spring Security & JWT](./11-spring-security-jwt.md) | SecurityFilterChain, JWT, UserDetailsService, @PreAuthorize | 20+ | ☐ |

---

## How to Use This Section

1. **Read the question** — treat it like a real interview; formulate your answer mentally first.
2. **Click to reveal** — compare your answer with the detailed explanation.
3. **Study the code** — every answer includes working Java/Spring code.
4. **Note the callouts** — 🚨 mistakes and ✅ best practices are highlighted.
5. **Check the tips** — > 💡 Interviewer Tip blocks show what interviewers really look for.

---

## Difficulty Guide

| Tag | Meaning |
|-----|---------|
| 🟢 Easy | Junior-level; foundational knowledge |
| 🟡 Medium | Mid-level; requires understanding of internals |
| 🔴 Hard | Senior-level; architecture decisions and deep internals |

---

## Recommended Study Order

For most backend roles, prioritize in this order:

1. **Spring Core & DI** → foundation for everything else
2. **Transactions & Propagation** → extremely common interview topic
3. **N+1 Problem** → database performance is always tested
4. **Spring Security & JWT** → almost every backend role asks this
5. **Bean Lifecycle** → frequently tested at senior levels
6. **Hibernate Internals** → deep ORM knowledge differentiates candidates

---

## Key Technologies Referenced

- Spring Framework 6.x
- Spring Boot 3.x
- Spring Security 6.x
- Hibernate 6.x / Jakarta Persistence (JPA) 3.x
- Java 17+

---

*Next Section: [03 - Docker for Java Backend](../03-docker/README.md)*
