# CraftVision 3D

## Project Overview
An AI-powered handmade gift platform that helps users:
- Generate handmade gift ideas
- Estimate materials and costs
- Get tutorials
- Generate personalized QR experiences
- Manage NFC-enabled physical gifts
- Chat with AI assistant

## Technology Stack
- **Frontend:** Next.js 15, TypeScript, TailwindCSS, Zustand
- **Backend Service:** .NET 9, Clean Architecture, CQRS Lite, MediatR, SignalR
- **AI Service:** .NET 9, Gemini/OpenAI Integrations
- **Gateway:** YARP (Reverse Proxy)
- **Infrastructure:** PostgreSQL, Redis, Kafka, Docker

## Monorepo Structure
- `backend/`: Contains the .NET 9 ecosystem (Backend Service, AI Service, Gateway, BuildingBlocks).
- `frontend/`: Contains the Next.js 15 frontend application.
- `docs/`: System architecture, SRS, ERD, and other documents.
- `databases/`: Database initialization and migration scripts.
- `docker/`: Dockerfiles and environment configurations.

## Getting Started
To spin up the local development environment:
```bash
docker-compose up -d
```
