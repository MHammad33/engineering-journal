# Engineering Patterns & Blueprints 

This directory serves as a library of reusable architectural patterns, standardized workflows, and "Golden Rules" that I apply across my technical projects.

## Purpose
The goal of this section is to move beyond "problem-solving" into "systematizing." It contains abstracted logic that ensures consistency, scalability, and maintainability in any codebase I touch.

## Catalog of Patterns

### AI & LLM Orchestration
- **[RAG Pipeline Standard](./ai/rag-standard.md):** My baseline architecture for Retrieval-Augmented Generation, focusing on chunking strategies and vector embeddings.
- **[Agentic Workflow Logic](./ai/agent-patterns.md):** Patterns for multi-step LLM reasoning and tool-calling loops.

### Frontend & State Management
- **[Custom Hook Architecture](./frontend/hook-patterns.md):** Strategies for separating business logic from UI components in React/Next.js.
- **[Performance Guardrails](./frontend/rendering-optimization.md):** Standardized approaches to memoization and minimizing expensive re-renders.

### Backend & Infrastructure
- **[RBAC Implementation](./backend/auth-rbac.md):** A scalable pattern for Role-Based Access Control in MERN/PostgreSQL environments.
- **[Error Handling Middleware](./backend/error-strategy.md):** Global error-catching patterns for Express and Next.js API routes.

---

## How to Use This Library
When starting a new feature or project:
1. **Reference:** Check these patterns to avoid "reinventing the wheel."
2. **Implement:** Apply the abstracted logic to the specific business requirements.
3. **Refine:** If a pattern is improved during a task, the changes are back-ported here to keep the "Golden Rule" up to date.

---
*"Good engineering is about minimizing the number of unique problems you have to solve."*
