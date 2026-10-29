---
title: "Free FastAPI Course: Pydantic v2, Dependency Injection, Async, SQLAlchemy, Auth, Testing & Deployment"
description: "Free FastAPI course: 40 lessons on Pydantic v2, dependency injection, async, SQLAlchemy 2.x, Alembic, OAuth2 and JWT, testing and production deployment."
---

# FastAPI Mastery Path

<div class="mp-hero-lockup">
  <div class="mp-hero-globe">
    <svg viewBox="0 0 160 160" class="mp-globe-svg mp-mark">
      <circle cx="80" cy="80" r="74" fill="var(--md-default-bg-color, #FAF7EF)" stroke="#2FA98D" stroke-width="5"/>
      <rect class="mp-bar mp-bar-1" x="36" y="90" width="17" height="24" rx="5" fill="#2FA98D"/>
      <rect class="mp-bar mp-bar-2" x="59" y="74" width="17" height="40" rx="5" fill="#7FC9B7"/>
      <rect class="mp-bar mp-bar-3" x="82" y="56" width="17" height="58" rx="5" fill="#E0B457"/>
      <rect class="mp-bar mp-bar-4" x="105" y="38" width="17" height="76" rx="5" fill="#C97B4A"/>
      <circle class="mp-dot" cx="44.5" cy="82" r="7" fill="#22261f"/>
    </svg>
  </div>
  <div class="mp-hero-divider"></div>
  <div class="mp-hero-text">
    <div class="mp-hero-title">Mastery Path</div>
    <div class="mp-hero-sub">BOOTCAMP</div>
    <div class="mp-hero-rule"></div>
    <div class="mp-hero-byline">VINOD BHANUJA</div>
  </div>
</div>

FastAPI turns ordinary Python functions with type hints into a validated, documented
HTTP API. Declare `book_id: int` and a request for `/books/forty-two` is answered with a
precise 422 before your code runs; declare a response model and a password hash can't
leak into the JSON; the OpenAPI document and the interactive `/docs` page come for free.
That's why FastAPI is quick to start with. It's also why its failures are recognisable:
an `async def` endpoint that calls a blocking library and freezes every request in the
process, a list endpoint that loads a whole table, a commit that fails after the client
was already told "saved", an API that checks who you are but not whether the record is
yours. This course teaches you to build with FastAPI and to understand the machinery
well enough to avoid those mistakes.

**Level 1** follows one request from the ASGI server to your function and back — path
and query parameters, Pydantic request bodies and response models, Pydantic v2 in depth,
errors and exception handlers, forms and uploads, routers — and ends with a tested
in-memory notes API. **Level 2** adds real data: dependency injection, yield dependencies
and their teardown timing, `async def` versus `def` measured under load, SQLAlchemy 2.x
sync and async, N+1 queries, Alembic migrations, typed settings, isolated database
tests and pagination, ending with a bookshelf API. **Level 3** makes it multi-user and
real-time: Argon2 password hashing, OAuth2 and JWT with rotating refresh tokens, scopes
and object-level authorization, middleware and CORS, lifespan and background tasks,
WebSockets and Server-Sent Events, OpenAPI and generated clients, async tests, caching,
ETags and rate limits, ending with an authenticated bookmarks service. **Level 4** is
production: profiling and load testing, worker processes and graceful shutdown, reverse
proxies, logs, metrics and FastAPI's built-in OpenTelemetry, OWASP API hardening,
versioning, job queues, upgrades, and a capstone booking API that sells exactly the
seats it has under concurrent load from four processes.

Every lesson has a **How It Actually Works** section — what FastAPI does with your
function signature at import time, why validation errors list every problem at once, why
a lazy-loaded relationship raises `MissingGreenlet` in async code, why `allow_origins=["*"]`
with credentials reflects any origin, why an exception handler can't see a request ID
stored in a context variable.

!!! note "Run, not imagined"
    Code, commands, responses and error messages in this course come from real runs on
    **FastAPI 0.143.0**, **Pydantic 2.14.0**, **Starlette 1.7.0**, **Uvicorn 0.54.0** and
    **SQLAlchemy 2.1.4** with Python 3.14, mostly against SQLite. Several lessons show
    first attempts that failed — a test-database setup that broke three different ways, a
    migration that left a temporary table behind, a codemod whose output didn't import —
    because you'll meet those too. Each level page lists what was run and what wasn't
    (no Docker, nginx, PostgreSQL or Redis were available). Timings are one laptop's
    results, not benchmarks.

## How the program is organized

| Level | Focus | Modules |
|-------|-------|---------|
| [Level 1 · Entry](level-1/index.md) | ASGI and the request lifecycle, first app, path and query parameters, request bodies, response models, Pydantic v2, errors, forms and uploads, routers | 9 topics + 1 project (an in-memory notes API) |
| [Level 2 · Intermediate](level-2/index.md) | Dependency injection, yield dependencies, async vs sync, SQLAlchemy 2.x sync and async, Alembic, settings, testing, pagination | 9 topics + 1 project (a bookshelf API with a database and tests) |
| [Level 3 · Advanced](level-3/index.md) | Password hashing, OAuth2 and JWT, authorization, middleware and CORS, lifespan and background tasks, WebSockets and streaming, OpenAPI, async tests, caching and rate limits | 9 topics + 1 project (a multi-user authenticated API) |
| [Level 4 · Master](level-4/index.md) | Performance, deployment, reverse proxies, observability, security hardening, versioning, job queues, advanced patterns, upgrading | 9 topics + 1 capstone (a production-ready event booking API) |

## How to use this site

- **Know your Python first.** FastAPI relies on type hints, classes, decorators and
  `async`/`await` more than most frameworks. If `Annotated[int, Query(ge=1)]` or
  `async def` looks unfamiliar, start with
  [Python Mastery Path](https://sigilipelli.github.io/python-mastery-path/).
- **Run every example.** Each lesson's code is complete enough to paste into a file and
  run; the outputs shown are what it printed. If yours differ, check the versions — the
  lessons point out the places where versions matter.
- **Read the 422s and the SQL.** Validation errors and the ORM's queries tell you most of
  what you need to know when something's wrong. Several lessons start by reading them.
- **Related courses.** For API design beyond FastAPI, see
  [REST API Mastery Path](https://sigilipelli.github.io/rest-api-mastery-path/); for a
  batteries-included Python web framework,
  [Django Mastery Path](https://sigilipelli.github.io/django-mastery-path/); for testing in
  depth, [Python Testing Mastery Path](https://sigilipelli.github.io/python-testing-mastery-path/);
  for SQL itself, [SQL Mastery Path](https://sigilipelli.github.io/sql-mastery-path/); for
  containers, [Docker Mastery Path](https://sigilipelli.github.io/docker-mastery-path/).
- Do the **Exercise** at the end of each lesson; the module-10 projects build on them.

Start here → [Level 1 · Entry](level-1/index.md)


🎥 **Prefer video?** Watch the [Mastery Path video series](https://youtube.com/@sigilipelli) on YouTube — Shorts and full walkthroughs of lessons across the series.

## More from the Mastery Path series

Free, structured, module-wise training across 97 other languages, platforms and disciplines — expand a category to browse:

<div class="mastery-tracks">
<details class="mastery-card">
<summary>Languages <span class="mastery-count">20</span></summary>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/python-mastery-path/">🐍 Python</a>
  <a href="https://sigilipelli.github.io/java-mastery-path/">☕ Java</a>
  <a href="https://sigilipelli.github.io/javascript-mastery-path/">🟨 JavaScript</a>
  <a href="https://sigilipelli.github.io/typescript-mastery-path/">🔷 TypeScript</a>
  <a href="https://sigilipelli.github.io/csharp-mastery-path/">🔵 C#</a>
  <a href="https://sigilipelli.github.io/shell-mastery-path/">🐚 Shell/Bash</a>
  <a href="https://sigilipelli.github.io/powershell-mastery-path/">💻 PowerShell</a>
  <a href="https://sigilipelli.github.io/c-mastery-path/">🇨 C</a>
  <a href="https://sigilipelli.github.io/cpp-mastery-path/">➕ C++</a>
  <a href="https://sigilipelli.github.io/go-mastery-path/">🐹 Go</a>
  <a href="https://sigilipelli.github.io/rust-mastery-path/">🦀 Rust</a>
  <a href="https://sigilipelli.github.io/sql-mastery-path/">🗄️ SQL</a>
  <a href="https://sigilipelli.github.io/ruby-mastery-path/">💎 Ruby</a>
  <a href="https://sigilipelli.github.io/php-mastery-path/">🐘 PHP</a>
  <a href="https://sigilipelli.github.io/kotlin-mastery-path/">🟣 Kotlin</a>
  <a href="https://sigilipelli.github.io/swift-mastery-path/">🐦 Swift</a>
  <a href="https://sigilipelli.github.io/dart-mastery-path/">🎯 Dart</a>
  <a href="https://sigilipelli.github.io/scala-mastery-path/">🔴 Scala</a>
  <a href="https://sigilipelli.github.io/r-mastery-path/">📊 R</a>
  <a href="https://sigilipelli.github.io/matlab-mastery-path/">🟧 MATLAB</a>
</div>
</details>
<details class="mastery-card">
<summary>Web Frameworks <span class="mastery-count">10</span></summary>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/react-mastery-path/">⚛️ React</a>
  <a href="https://sigilipelli.github.io/nextjs-mastery-path/">▲ Next.js</a>
  <a href="https://sigilipelli.github.io/angular-mastery-path/">🅰️ Angular</a>
  <a href="https://sigilipelli.github.io/vue-mastery-path/">💚 Vue.js</a>
  <a href="https://sigilipelli.github.io/html-css-mastery-path/">🎨 HTML & CSS</a>
  <a href="https://sigilipelli.github.io/tailwind-mastery-path/">🌬️ Tailwind CSS</a>
  <a href="https://sigilipelli.github.io/django-mastery-path/">🎸 Django</a>
  <a href="https://sigilipelli.github.io/graphql-mastery-path/">🕸️ GraphQL</a>
  <a href="https://sigilipelli.github.io/nodejs-mastery-path/">🟩 Node.js</a>
  <a href="https://sigilipelli.github.io/spring-boot-mastery-path/">🍃 Spring Boot</a>
</div>
</details>
<details class="mastery-card">
<summary>Mobile <span class="mastery-count">2</span></summary>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/flutter-mastery-path/">💙 Flutter</a>
  <a href="https://sigilipelli.github.io/react-native-mastery-path/">📲 React Native</a>
</div>
</details>
<details class="mastery-card">
<summary>Testing & QA <span class="mastery-count">6</span></summary>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/java-testing-mastery-path/">🧪 Java Testing</a>
  <a href="https://sigilipelli.github.io/cpp-testing-mastery-path/">🧪 C/C++ Testing</a>
  <a href="https://sigilipelli.github.io/python-testing-mastery-path/">🧪 Python Testing</a>
  <a href="https://sigilipelli.github.io/automotive-testing-mastery-path/">🚗 Automotive Testing</a>
  <a href="https://sigilipelli.github.io/selenium-mastery-path/">🧭 Selenium</a>
  <a href="https://sigilipelli.github.io/cypress-mastery-path/">🌲 Cypress</a>
</div>
</details>
<details class="mastery-card">
<summary>Security <span class="mastery-count">2</span></summary>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/cybersecurity-mastery-path/">🛡️ Cybersecurity</a>
  <a href="https://sigilipelli.github.io/ethical-hacking-mastery-path/">🎯 Ethical Hacking</a>
</div>
</details>
<details class="mastery-card">
<summary>Cloud Platforms <span class="mastery-count">5</span></summary>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/aws-mastery-path/">☁️ AWS</a>
  <a href="https://sigilipelli.github.io/azure-mastery-path/">☁️ Azure</a>
  <a href="https://sigilipelli.github.io/gcp-mastery-path/">☁️ GCP</a>
  <a href="https://sigilipelli.github.io/ibm-cloud-mastery-path/">☁️ IBM Cloud</a>
  <a href="https://sigilipelli.github.io/adobe-mastery-path/">🎨 Adobe</a>
</div>
</details>
<details class="mastery-card">
<summary>Data & Analytics <span class="mastery-count">7</span></summary>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/data-engineering-mastery-path/">🛠️ Data Engineering</a>
  <a href="https://sigilipelli.github.io/data-science-mastery-path/">📈 Data Science</a>
  <a href="https://sigilipelli.github.io/tableau-mastery-path/">📊 Tableau</a>
  <a href="https://sigilipelli.github.io/excel-mastery-path/">📗 Excel</a>
  <a href="https://sigilipelli.github.io/pyspark-mastery-path/">⚡ PySpark</a>
  <a href="https://sigilipelli.github.io/etl-datalake-mastery-path/">🏞️ ETL & Data Lake</a>
  <a href="https://sigilipelli.github.io/power-bi-mastery-path/">📉 Power BI</a>
</div>
</details>
<details class="mastery-card">
<summary>Databases <span class="mastery-count">5</span></summary>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/postgresql-mastery-path/">🐘 PostgreSQL</a>
  <a href="https://sigilipelli.github.io/mongodb-mastery-path/">🍃 MongoDB</a>
  <a href="https://sigilipelli.github.io/redis-mastery-path/">🟥 Redis</a>
  <a href="https://sigilipelli.github.io/snowflake-mastery-path/">❄️ Snowflake</a>
  <a href="https://sigilipelli.github.io/databricks-mastery-path/">🧱 Databricks</a>
</div>
</details>
<details class="mastery-card">
<summary>AI / ML / LLM <span class="mastery-count">11</span></summary>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/ai-ml-mastery-path/">🤖 AI/ML</a>
  <a href="https://sigilipelli.github.io/llm-dev-mastery-path/">🧠 LLM Dev</a>
  <a href="https://sigilipelli.github.io/rag-mastery-path/">📚 RAG</a>
  <a href="https://sigilipelli.github.io/edge-ai-mastery-path/">📱 Edge AI</a>
  <a href="https://sigilipelli.github.io/claude-training-mastery-path/">🔶 Claude Training</a>
  <a href="https://sigilipelli.github.io/ai-tools-mastery-path/">🧰 AI Tools</a>
  <a href="https://sigilipelli.github.io/ml-math-mastery-path/">➗ ML Math Foundations</a>
  <a href="https://sigilipelli.github.io/prompt-engineering-mastery-path/">✍️ Prompt Engineering</a>
  <a href="https://sigilipelli.github.io/ai-agents-mastery-path/">🕹️ AI Agents</a>
  <a href="https://sigilipelli.github.io/mlops-mastery-path/">🔁 MLOps</a>
  <a href="https://sigilipelli.github.io/computer-vision-mastery-path/">👁️ Computer Vision</a>
</div>
</details>
<details class="mastery-card">
<summary>Embedded Systems <span class="mastery-count">5</span></summary>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/embedded-mastery-path/">🔌 Embedded</a>
  <a href="https://sigilipelli.github.io/embedded-linux-mastery-path/">🐧 Embedded Linux</a>
  <a href="https://sigilipelli.github.io/embedded-python-mastery-path/">🐍 Embedded Python</a>
  <a href="https://sigilipelli.github.io/freertos-mastery-path/">⏱️ FreeRTOS</a>
  <a href="https://sigilipelli.github.io/s32k-mastery-path/">🔧 S32K</a>
</div>
</details>
<details class="mastery-card">
<summary>Leadership & Management <span class="mastery-count">5</span></summary>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/product-manager-mastery-path/">📋 Product Manager</a>
  <a href="https://sigilipelli.github.io/product-lead-mastery-path/">🧭 Product Lead</a>
  <a href="https://sigilipelli.github.io/project-manager-mastery-path/">📅 Project Manager</a>
  <a href="https://sigilipelli.github.io/ai-manager-mastery-path/">🤖 AI Manager</a>
  <a href="https://sigilipelli.github.io/servant-leadership-mastery-path/">🤝 Servant Leadership</a>
</div>
</details>
<details class="mastery-card">
<summary>Professional Skills <span class="mastery-count">2</span></summary>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/english-fluency-mastery-path/">🗣️ English Fluency & IELTS</a>
  <a href="https://sigilipelli.github.io/workday-mastery-path/">🧑‍💼 Workday</a>
</div>
</details>
<details class="mastery-card">
<summary>Careers & Interviews <span class="mastery-count">2</span></summary>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/dsa-interview-mastery-path/">🧩 DSA & Coding Interviews</a>
  <a href="https://sigilipelli.github.io/system-design-mastery-path/">🏗️ System Design</a>
</div>
</details>
<details class="mastery-card">
<summary>Process & APIs <span class="mastery-count">3</span></summary>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/agile-mastery-path/">🔄 Agile/Scrum/Kanban</a>
  <a href="https://sigilipelli.github.io/rest-api-mastery-path/">🔗 REST API</a>
  <a href="https://sigilipelli.github.io/playwright-mastery-path/">🎭 Playwright</a>
</div>
</details>
<details class="mastery-card">
<summary>Infrastructure & Ops <span class="mastery-count">12</span></summary>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/server-ops-mastery-path/">🖥️ Server Ops</a>
  <a href="https://sigilipelli.github.io/linux-mastery-path/">🐧 Linux</a>
  <a href="https://sigilipelli.github.io/nodemcu-mastery-path/">📶 NodeMCU/IoT</a>
  <a href="https://sigilipelli.github.io/terraform-mastery-path/">🌍 Terraform</a>
  <a href="https://sigilipelli.github.io/kubernetes-mastery-path/">☸️ Kubernetes</a>
  <a href="https://sigilipelli.github.io/docker-mastery-path/">🐳 Docker</a>
  <a href="https://sigilipelli.github.io/github-mastery-path/">🐙 GitHub & Git</a>
  <a href="https://sigilipelli.github.io/cicd-mastery-path/">🔁 CI/CD</a>
  <a href="https://sigilipelli.github.io/ansible-mastery-path/">🤖 Ansible</a>
  <a href="https://sigilipelli.github.io/kafka-mastery-path/">📨 Kafka</a>
  <a href="https://sigilipelli.github.io/microservices-mastery-path/">🧩 Microservices</a>
  <a href="https://sigilipelli.github.io/observability-mastery-path/">📈 Observability</a>
</div>
</details>
</div>

