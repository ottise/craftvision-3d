# CraftVision 3D - System Architecture & Design Document

## 1. High-Level Architecture
The system is designed as a **Microservice-inspired Monorepo**, with clear separation of concerns:
- **Frontend (Next.js 15):** Handles the user interface, calls APIs via the Gateway, and receives Realtime notifications via SignalR.
- **Gateway (YARP):** Acts as a Reverse Proxy, handling routing, Rate Limiting, and Authentication forwarding.
- **Backend Service (.NET 9):** Manages core business logic (Auth, User, Order, QR, NFC, Product...). Applies Clean Architecture and CQRS Lite.
- **AI Service (.NET 9):** An independent service responsible for calling LLM APIs (Gemini/OpenAI), managing prompts, and orchestrating AI chat flows.
- **Infrastructure:** Asynchronous communication via **Kafka**, Caching and Websocket Backplane via **Redis**, and data storage via two separate **PostgreSQL** databases.

## 2. Complete Monorepo Folder Tree
```text
craftvision-3d/
│
├── backend/                              # All .NET backend source code
├── frontend/                             # Next.js 15 Frontend
├── docs/                                 # Project Documentation
│   ├── SRS/                              # Software Requirements Specification
│   ├── SDS/                              # Software Design Specification
│   ├── Architecture/                     # Architecture Diagrams & ADRs
│   ├── ERD/                              # Entity Relationship Diagrams
│   ├── API/                              # API Contracts & Swagger docs
│   ├── UseCases/                         # Use case documents
│   ├── SequenceDiagram/                  # UML Sequence Diagrams
│   ├── ClassDiagram/                     # UML Class Diagrams
│   ├── Deployment/                       # Deployment architecture notes
│   └── MeetingMinutes/                   # Team meeting notes
│
├── databases/                            # Database scripts and migrations
│   ├── backend/
│   ├── ai/
│   └── migrations/
│
├── docker/                               # Docker configuration files
│   ├── backend/
│   ├── ai-service/
│   ├── gateway/
│   ├── postgres/
│   ├── redis/
│   ├── kafka/
│   ├── zookeeper/
│   └── nginx/
│
├── monitoring/                           # Observability configuration
│   ├── Serilog/
│   ├── HealthChecks/
│   └── Dashboards/
│
├── postman/                              # Postman Collections & Environments
├── scripts/                              # CI/CD and utility shell scripts
│
├── docker-compose.yml                    # Local development orchestration
├── craftvision-3d.sln                    # Visual Studio Master Solution
├── agent.md                              # Local prompt context for agent
└── README.md                             # Basic project startup guide
```

### 2.1. Root Level Directories Explained
- **`backend/`**: Contains all .NET microservices, Gateway, and BuildingBlocks (detailed extensively in Section 4).
- **`frontend/`**: Contains the Next.js 15 application codebase, using App Router, Tailwind CSS, and Zustand for state management.
- **`docs/`**: Centralized documentation for the entire team.
  - `SRS/` & `SDS/`: System Requirements and Design Specifications.
  - `Architecture/`: High-level architecture and Architecture Decision Records (ADRs).
  - `ERD/`, `SequenceDiagram/`, `ClassDiagram/`: Database schema and UML diagrams.
  - `API/`: Postman documentation exports or OpenAPI/Swagger specs.
- **`databases/`**: SQL scripts for initializing databases and managing raw migrations outside of EF Core if needed.
- **`docker/`**: Dockerfiles and environment-specific configuration files for containerizing each service (backend, AI, gateway, Postgres, Redis, Kafka).
- **`monitoring/`**: Configurations for system observability.
  - `Serilog/`: Centralized logging configurations.
  - `HealthChecks/`: Custom health check scripts.
  - `Dashboards/`: Grafana/Kibana dashboard JSON exports for monitoring metrics.
- **`postman/`**: Shared Postman collections and environment variables for testing APIs as a team.
- **`scripts/`**: Useful shell/PowerShell scripts for automating local setup, cleaning solutions, or CI/CD pipelines.

## 3. Visual Studio Solution Organization
```text
Solution 'craftvision-3d'
│
├── 📁 Gateway
│   └── CraftVision.Gateway
│
├── 📁 BuildingBlocks
│   └── CraftVision.BuildingBlocks
│
├── 📁 Backend
│   ├── CraftVision.Domain
│   ├── CraftVision.Application
│   ├── CraftVision.Infrastructure
│   └── CraftVision.Presentation
│
└── 📁 AI Service
    ├── CraftVision.AI.Domain
    ├── CraftVision.AI.Application
    ├── CraftVision.AI.Infrastructure
    └── CraftVision.AI.Presentation
```

## 4. Detailed Folder Purpose Breakdown (Down to the smallest branches)

### 4.1. `backend/BuildingBlocks/` (Shared utilities)
*Contains shared source code for the entire system to avoid code duplication between the Backend and AI Service.*
- **`Constants/`**: Stores system-wide static values, error codes, and magic strings to avoid hardcoding.
- **`Contracts/`**: Standardized data structures (e.g., generic API response wrappers, pagination models).
- **`Exceptions/`**: Custom application exceptions (e.g., `NotFoundException`, `ValidationException`) handled by global middleware.
- **`Messaging/`**: Event schemas (DTOs) used for communication over Kafka (e.g., `OrderPlacedEvent`).
- **`Security/`**, **`Utils/`**, **`Web/`**, **`Extensions/`**: Reusable extension methods, security helpers (hashing), and web-specific utilities.

### 4.2. `backend/Gateway/CraftVision.Gateway/` (YARP Proxy)
*The single entry point for all frontend requests.*
- **`Configurations/`**: Contains proxy route definitions loaded from `appsettings.json`.
- **`Extensions/`**: Extension methods for configuring YARP services in the Dependency Injection container.
- **`Middleware/`**: Custom ASP.NET Core middleware to intercept requests for logging or global rate limiting before routing.
- **`HealthChecks/`**: Endpoints to monitor if the gateway service itself is operational.

### 4.3. `backend/Backend/CraftVision.Domain/` (Core Logic)
*The heart of the system. Has **NO** dependencies on external libraries.*
- **`Common/`**: Base classes inherited by entities (e.g., `BaseEntity`, `IAuditable` for tracking created/updated dates).
- **`Entities/`**: Database table representations (e.g., `User`, `Order`, `Product`).
- **`Enums/`**: Fixed sets of values (e.g., `OrderStatus`, `RoleType`).
- **`ValueObjects/`**: Immutable objects representing descriptive aspects of the domain with no identity (e.g., `Address`, `Money`).
- **`Events/`**: Domain events triggered when the internal state of an aggregate root changes.

### 4.4. `backend/Backend/CraftVision.Application/` (Use Cases)
*Business logic and orchestration. Depends **ONLY** on the Domain.*
- **`Interfaces/`**: Abstractions that the Infrastructure layer will implement.
  - `IHelpers/`: Utility interfaces.
  - `IRepositories/`: Database access contracts (e.g., `IUserRepository`).
  - `IServices/`: External service contracts (e.g., `IEmailService`).
  - `IValidation/`: Validation logic contracts.
- **`Features/`**: Contains the CQRS modules (e.g., `Auth`, `User`, `Order`). **Inside every feature folder:**
  - **`Commands/`**: Objects containing data for write operations (Create, Update, Delete).
  - **`Queries/`**: Objects containing data for read operations (Get, List).
  - **`Handlers/`**: The actual business logic (Use Cases) that execute the Commands/Queries via MediatR.
  - **`Validators/`**: FluentValidation rules ensuring data integrity before reaching the Handler.
  - **`DTOs/`**: Data Transfer Objects used to receive requests and send responses specific to this feature.
- **`Mappings/`**: AutoMapper profiles to map between Entities and DTOs.
- **`Services/`**: Minimal folder for shared, cross-cutting business services (e.g., `JwtTokenGenerator`).
- **`Behaviors/`**: MediatR pipeline behaviors intercepting every request.
  - `ValidationBehavior/`: Automatically runs FluentValidation.
  - `LoggingBehavior/`: Logs request/response execution times.
  - `PerformanceBehavior/`: Detects and logs slow-running queries.
- **`Events/`**: MediatR event handlers reacting to Domain Events.
- **`Exceptions/`**: Application-specific exceptions thrown by Handlers.

### 4.5. `backend/Backend/CraftVision.Infrastructure/` (External Integrations)
*External concerns and actual implementation of Application interfaces.*
- **`Data/`**: Database logic.
  - `Context/`: The Entity Framework Core `DbContext`.
  - `Configurations/`: EF Core Fluent API mappings for entities.
  - `Migrations/`: Code-first database migration files.
  - `Seed/`: Scripts/Classes to populate default data (admin accounts, default categories).
- **`Repositories/`**: Actual implementations of the `IRepository` interfaces from the Application layer.
- **`Caching/`**: Implementations for Redis.
  - `ApiCache/`: Logic for caching API responses.
  - `RefreshTokens/`: Logic for storing and validating JWT refresh tokens.
  - `SignalR/`: Redis Backplane configuration for scaling WebSockets.
  - `RateLimiting/`: Distributed rate limiting implementations.
- **`Messaging/`**: Implementations for Kafka.
  - `Producers/`: Logic to publish messages to Kafka topics.
  - `Consumers/`: Background workers listening to Kafka topics.
  - `Events/`, `Topics/`, `Configurations/`: Kafka configuration classes.
- **`AI/`**: Integration clients connecting to the AI Service APIs.
- **`Storage/`**: Handlers for saving physical files (avatars, 3D models) to S3 or local disk.
- **`Workers/`**: Background services (e.g., clearing expired sessions).
- **`Security/`**, **`Helpers/`**, **`Utilities/`**: Infrastructure-specific utility implementations.

### 4.6. `backend/Backend/CraftVision.Presentation/` (Entry Points)
*API endpoints and client interactions. Depends **ONLY** on Application.*
- **`Controllers/`**: HTTP REST API Endpoints. Their only job is to receive HTTP requests, map them, and send them to MediatR.
- **`Hubs/`**: SignalR WebSocket endpoints for real-time communication.
  - `ChatHub/`: Handles real-time chat with AI/Support.
  - `NotificationHub/`: Pushes alerts to users.
- **`Middleware/`**: ASP.NET Core middleware (e.g., global exception handler formatting errors as JSON).
- **`Authorization/`**, **`Filters/`**: Custom auth policies and action filters.
- **`Swagger/`**: OpenAPI generation configurations.
- **`HealthChecks/`**: Endpoints for Docker/Kubernetes liveness and readiness probes.
- **`Configurations/`**: Setup files extending `Program.cs`.

### 4.7. `backend/AI/` (AI Processing Service Ecosystem)
*Follows the same Clean Architecture layout as the core Backend, but customized for AI.*
- **`CraftVision.AI.Domain/`**: Contains entities and enums for AI conversation tracking.
- **`CraftVision.AI.Application/`**:
  - `Features/Chat/`: CQRS structure for handling chat messages (Commands, Queries, Handlers, etc.).
  - `Prompts/`: Dedicated folder managing System Prompts and few-shot examples for the LLMs.
- **`CraftVision.AI.Infrastructure/`**:
  - `LLM/`: SDK implementations connecting to Google Gemini or OpenAI.
  - `PromptTemplates/`: Service for injecting variables into prompt strings.
  - `Kafka/`: Consumers listening to Backend requests (e.g., generate gift idea).
- **`CraftVision.AI.Presentation/`**: AI-specific controllers and middleware.

## 5. Layer Organization Rules
### Infrastructure Structure Rules
- **Backend Infrastructure:** Uses a `Files/` (or `Storage/`) folder specifically for uploading avatars, product images, and QR images.
- **AI Infrastructure:** Uses a `Data/` folder (instead of Persistence) to store DB Contexts and Migrations, keeping it synchronized with the Backend convention.

## 6. Clean Architecture Dependency Rules
1. **Domain Layer**: The heart. It has **NO** dependencies on any other project. No NuGet packages for DB or HTTP allowed.
2. **Application Layer**: Depends **ONLY** on the Domain Layer. It defines interfaces that Infrastructure will implement.
3. **Infrastructure Layer**: Depends on **Application** and **Domain** layers. Contains actual implementations.
4. **Presentation Layer**: Depends **ONLY** on the **Application** layer. (Controllers only need MediatR to dispatch requests).

## 7. CQRS Folder Organization Strategy
Organized strictly by **Feature** to maintain high cohesion.
```text
CraftVision.Application/Features/{FeatureName}/
│
├── Commands/          # State-changing operations
├── Queries/           # Read-only operations
├── Handlers/          # Handlers = Use Cases (core business logic)
├── DTOs/              # Data Transfer Objects specific to this feature
└── Validators/        # FluentValidation rules for commands/queries
```

**Note on `Services/` folder in Application:** 
Because this architecture uses CQRS, the Handler IS the Use Case. Therefore, the `Services/` folder will be kept minimal and reserved ONLY for cross-cutting utility services like `JwtService`, `NotificationService`, or `EmailService`.

## 8. MediatR Organization Strategy
- **Commands/Queries**: Segregated inside the `Features/` folder.
- **Behaviors**: Placed in `Application/Behaviors/` (e.g., `ValidationBehavior`, `LoggingBehavior`, `PerformanceBehavior`).
- **Domain Events**: Dispatched via MediatR. Handlers for these events are placed in `Application/Events/`.

## 9. Project Reference Rules
### Backend & AI Service Strict Rules:
- `Presentation.csproj` → references `Application.csproj`
- `Infrastructure.csproj` → references `Application.csproj` and `Domain.csproj`
- `Application.csproj` → references `Domain.csproj`
- `Domain.csproj` → references **Nothing**

## 10. Git Branching Strategy (GitHub Flow)
- `main`: Production-ready, stable code.
- `dev`: Main integration branch for the team.
- `feature/[name]`: For new features (e.g., `feature/nfc-integration`).
- `bugfix/[name]`: For fixing issues.
- **Rule**: All merges to `dev` or `main` MUST go through a Pull Request (PR).

## 11. Development Workflow
1. **Task Assignment**: Developer picks a task from Jira/Trello.
2. **Branching**: Checkout new branch from `dev`.
3. **Implementation**:
   - Create Domain Entities.
   - Define Command/Query, Handler, DTO, and Validator.
   - Implement Infrastructure Repo/Service (if needed).
   - Expose via Presentation Controller.
4. **Testing**: Run local Docker environment.
5. **PR & Review**: Push branch, open PR. Reviewer checks Clean Architecture rules and CQRS structure.
6. **Merge**: Merge to `dev`.

## 12. Docker Deployment Overview
- **Local Dev**: Use `docker-compose.yml` at the root. Running `docker-compose up -d` will spin up PostgreSQL, Redis, Kafka, Zookeeper, and optionally the .NET services.
- **Containers**: `craftvision-backend`, `craftvision-ai`, `craftvision-gateway`, `craftvision-postgres`, `craftvision-redis`, `craftvision-kafka`, `craftvision-frontend`.

## 13. Kafka Communication Overview
- **Topics**: `ai-generation-requests`, `ai-generation-completed`.
- **Flow**: Backend publishes `GiftIdeaRequestedEvent` -> AI Service consumes and processes with LLM -> AI publishes `GiftIdeaCompletedEvent` -> Backend consumes, updates DB, and notifies via SignalR.
- **Internal Backend Events**: Kafka is also used to publish events within the backend (e.g., Domain Events like `OrderPlacedEvent`) to trigger background tasks and asynchronous side-effects.

## 14. Redis Usage Overview
- Caching API responses, managing Refresh Tokens, SignalR backplane, and Rate Limiting for Gateway.

## 15. Gateway Routing Overview
- **`Route: /api/{**catch-all}`** ➔ Backend Service.
- **`Route: /ai/{**catch-all}`** ➔ AI Service.
- **`Route: /hubs/{**catch-all}`** ➔ Backend Service (SignalR).

## 16. Frontend Architecture Overview
- Next.js 15 App Router. `Zustand` for global state. `Axios` and `@microsoft/signalr` for network and real-time updates. Components are feature-based (`components/nfc/`, `components/product/`).
