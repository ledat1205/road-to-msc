### Python Programming

Expect questions on core concepts, best practices, and problem-solving, possibly with live coding (e.g., LeetCode-style problems like string manipulation, data structures, or async tasks). Focus on backend relevance: concurrency, APIs, databases.

**Key Topics to Review:**

- Syntax and idioms: List comprehensions, decorators, generators/iterators, context managers.
- Advanced: Asyncio/coroutines for I/O-bound tasks, multiprocessing for CPU-bound.
- Libraries: Standard (collections, functools), web (FastAPI/Flask), data (Pandas/Numpy if relevant from your CV), ORM (SQLAlchemy).
- Best practices: PEP 8, error handling, testing (unittest/pytest), performance optimization (profiling with cProfile).
- Backend-specific: REST/GraphQL APIs, microservices patterns, caching (Redis), queueing (Celery).

**Common Interview Questions/Examples:**

| Question Type | Sample Questions                                                          | Tips/How to Prepare                                                                                                   |
| ------------- | ------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| Conceptual    | Explain the GIL and how to handle concurrency in Python.                  | Review: GIL limits threads for CPU tasks; use multiprocessing or asyncio. Practice: Write a simple async web scraper. |
| Coding        | Implement a function to merge intervals (e.g., [[1,3],[2,6]] -> [[1,6]]). | Practice on LeetCode (medium difficulty). Sort intervals, merge overlapping.                                          |
| Backend       | How would you design a rate limiter for an API?                           | Use token bucket (e.g., with Redis). Tie to your MoMo high-traffic experience.                                        |
| From Your CV  | How did you optimize PostgreSQL in Tiki?                                  | Explain micro-batching and query refactoring; prepare code snippets if possible.                                      |
| Advanced      | Difference between **init** and **new**?                                  | **new** creates instance, **init** initializes. Practice metaclasses if time.                                         |

Practice 5-10 problems on HackerRank or LeetCode in Python, focusing on arrays, trees, and dynamic programming.

### Kotlin Programming

Kotlin questions may cover interoperability with Java, modern features, and backend use (e.g., in Spring/Vert.x). Expect coding on null safety, coroutines, or collections.

**Key Topics to Review:**

- Basics: Null safety (nullable types, safe calls), extension functions, data classes, lambdas.
- Advanced: Coroutines for async (suspend functions, flows), sealed classes, inline functions.
- Backend: Ktor/Spring Boot for APIs, JVM interop, dependency injection (Koin/Dagger).
- Collections: Higher-order functions (map, filter, fold), immutable vs mutable.
- Best practices: Idiomatic Kotlin over Java-style, error handling (Result/try-catch), testing (JUnit/Kotest).

**Common Interview Questions/Examples:**

|Question Type|Sample Questions|Tips/How to Prepare|
|---|---|---|
|Conceptual|How does Kotlin handle null safety compared to Java?|Elvis operator (?:), safe calls (?.), let/also. Practice: Rewrite a Java null-check in Kotlin.|
|Coding|Write a function to find the longest palindrome substring.|Use dynamic programming or expand-around-center. Practice on LeetCode in Kotlin.|
|Backend|How would you implement coroutines in a Vert.x service for async DB calls?|Use suspend functions with Dispatchers.IO. Link to your MoMo Vert.x work.|
|From Your CV|Describe redesigning storage in Heo Dat (HBase to PostgreSQL).|Focus on Kotlin/Java code for migration logic, data integrity.|
|Advanced|What are delegated properties?|By lazy, observable. Practice: Implement a lazy-initialized cache.|

Use IntelliJ for practice; solve problems on Kotlin Playground or LeetCode's Kotlin support.

### Preparation for AWS Cloud

Focus on services in the JD (elastic infrastructure, S3, Aurora/PostgreSQL). Questions may be scenario-based, like designing scalable systems.

**Key Topics to Review:**

- Core Services: EC2 (instances), S3 (storage), RDS/Aurora (databases), Lambda (serverless).
- Networking/Security: VPC, IAM roles/policies, Security Groups.
- Scalability: Auto Scaling Groups, ELB (load balancers), EKS (Kubernetes on AWS).
- Monitoring: CloudWatch, X-Ray for tracing.
- Best Practices: Cost optimization, high availability (multi-AZ), CI/CD with CodePipeline.
- From JD: Elastic infrastructure—review Elastic Beanstalk or ECS/EKS for microservices.

**Common Interview Questions/Examples:**

|Question Type|Sample Questions|Tips/How to Prepare|
|---|---|---|
|Conceptual|Explain S3 consistency models and when to use Glacier.|Eventual for reads, strong for new objects. Tie to your TMA data syncs.|
|Design|How to architect a scalable microservice on AWS with PostgreSQL?|Use ECS/EKS for containers, RDS Aurora for DB, S3 for statics, Auto Scaling. Draw a simple diagram mentally.|
|Hands-On|What IAM policy allows read-only S3 access?|JSON policy with s3:GetObject. Practice in AWS console (free tier).|
|From Your CV|How did you use AWS in TMA for ingestion pipelines?|Emphasize scalability (FAISS embeddings), security (multi-tenant).|
|Advanced|Difference between RDS and Aurora?|Aurora is serverless-compatible, faster replication. Review for JD's persistence stack.|

Hands-on: Use AWS free tier to deploy a simple Python/Kotlin app (e.g., via Elastic Beanstalk). Read AWS Well-Architected Framework pillars (reliability, performance).

### General Interview Tips

- **Structure Answers**: Use STAR (Situation, Task, Action, Result) for behavioral questions, tying back to CV metrics (e.g., "Reduced MTTD by 90% at Tiki").
- **Practice Mock Interviews**: Time yourself on Pramp or Interviewing.io; focus on explaining code aloud.
- **Questions to Ask**: "How does the team handle on-call for SRE?" or "What's the roadmap for Thought Machine integrations?" Shows interest.
- **Logistics**: Since it's in HCMC and you're local, confirm hybrid details. Review basics like time complexity (O(n)) for coding.
- **Mindset**: Be confident in your 2+ years of relevant exp; they're looking for quick learners. If stuck, think aloud and ask clarifying questions.