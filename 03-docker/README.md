# Section 3: Docker for Java Backend Engineers

Docker is an essential skill for modern Java backend engineers. Whether you're packaging a Spring Boot microservice, tuning JVM memory inside a Kubernetes pod, or debugging a container restart loop at 2 AM, solid Docker fundamentals separate senior engineers from the rest. This section covers everything from basic image building to production-grade container hardening — all with a Java-first lens.

---

## 📚 What You'll Learn

- **Docker image building, layering, and caching** — understand how Docker layers work, why order matters, and how to keep builds fast
- **Multi-stage builds for Java** — produce lean production images by separating build (Maven/Gradle + JDK) from runtime (JRE or distroless)
- **JVM memory tuning inside containers** — avoid OOMKilled pods by mastering `-XX:MaxRAMPercentage`, container-aware GC settings, and `UseContainerSupport`
- **Production best practices** — avoid the 10 most common Docker mistakes Java teams make in production (root user, `:latest` tag, secrets in ENV, etc.)

---

## 📋 Progress Checklist

- [ ] Chapter 1: Docker Fundamentals for Java
- [ ] Chapter 2: Multi-Stage Docker Builds
- [ ] Chapter 3: JVM Memory Tuning in Containers
- [ ] Chapter 4: Common Production Mistakes

---

## ⏱️ Estimated Study Time

**1 week** — approximately 1–2 hours per chapter, with extra time to practice hands-on.

---

## 📂 Files in This Section

| File | Topic |
|------|-------|
| [`01-docker-fundamentals-for-java.md`](./01-docker-fundamentals-for-java.md) | Dockerfile anatomy, images vs containers, volumes, networking, Docker Compose |
| [`02-multi-stage-docker-builds.md`](./02-multi-stage-docker-builds.md) | Multi-stage Maven/Gradle builds, layer caching, distroless images |
| [`03-jvm-memory-tuning-in-containers.md`](./03-jvm-memory-tuning-in-containers.md) | `UseContainerSupport`, `MaxRAMPercentage`, GC selection, OOMKilled debugging |
| [`04-common-production-mistakes.md`](./04-common-production-mistakes.md) | 10 production anti-patterns with WRONG ❌ vs CORRECT ✅ examples |

---

## 🔗 Related Sections

- [01-core-java](../01-core-java/README.md) — Java fundamentals
- [02-spring-and-spring-boot](../02-spring-and-spring-boot/README.md) — Spring Boot applications you'll be containerizing
- [04-cloud-and-devops](../04-cloud-and-devops/README.md) — Kubernetes, CI/CD pipelines, and cloud deployments

---

## 💡 How to Use This Section

1. Read each chapter in order — later chapters build on earlier concepts.
2. Open each `<details>` block only **after** attempting to answer the question yourself.
3. Spin up a local Docker environment and run the examples — reading alone isn't enough.
4. Revisit the Common Production Mistakes chapter before any system design interview.

---

> **Interview tip:** Interviewers at FAANG+ companies often ask Docker questions in the context of a specific failure scenario ("a pod is OOMKilled — walk me through your investigation"). Know the *why*, not just the commands.
