# Singularity.IO - Autonomous AI Research Agent Framework

## 1. Overview

Singularity.IO is a Python-based, asynchronous framework for building, deploying, and orchestrating autonomous AI agents. It is engineered for complex research and analysis tasks that require multi-step reasoning, interaction with diverse data sources, and a deep, semantic understanding of software repositories.

For the expert practitioner, Singularity.IO offers a solution to a distinct class of problems beyond simple prompt-chaining: automating high-latency, complex cognitive tasks. This includes technical due diligence, competitive analysis, large-scale codebase refactoring assessments, and synthesizing research from dozens of unstructured sources. The framework's core value lies in its ability to combine a structured, tool-based execution loop with the fluid, semantic reasoning capabilities of modern LLMs.

The system is architected around an asynchronous core (`asyncio`), ensuring that I/O-bound operations (API calls, database interactions, file access) are handled concurrently for maximum throughput. Its modular, registry-based design facilitates extension and customization, allowing developers to integrate bespoke tools, proprietary models, and specialized agentic logic with minimal friction.

## 2. Key Features: A Technical Deep-Dive

#### 2.1. Extensible, Tool-Based System
The agent's capabilities are not hardcoded but are instead defined by a collection of "Tools."

- **Implementation:** Each tool adheres to a base protocol (e.g., an abstract base class) requiring `name`, `description`, and an `execute` method. The `description` is critical, as it is used by the agent's LLM planner to determine which tool is appropriate for a given sub-task.
- **Extensibility:** New tools can be added by simply implementing this protocol and registering the class with the central `Registry`. This design pattern avoids tight coupling and allows for a rich ecosystem of capabilities to be developed independently of the agent's core logic.

#### 2.2. Deep Codebase Analysis
This is the framework's cornerstone feature, enabling agents to reason about software architecture and implementation details.

- **Indexing Pipeline:** When a repository is targeted, it triggers an indexing pipeline that:
    1.  **Clones** the repository using a sparse checkout for efficiency.
    2.  **Parses** the file tree, respecting `.gitignore` and other exclusion rules.
    3.  **Chunks Documents:** Source code files are intelligently chunked using a `RecursiveCharacterTextSplitter`. This method attempts to preserve semantic context by splitting on logical separators (classes, functions, etc.) before resorting to fixed-size chunks.
    4.  **Generates Embeddings:** Each chunk is passed through a text embedding model (e.g., `text-embedding-ada-002` or a locally-hosted Sentence Transformer) to create a high-dimensional vector representation.
    5.  **Stores in Vector DB:** These vectors, along with their corresponding metadata (file path, line numbers), are stored in a FAISS index. FAISS is used for its high-speed in-memory similarity search capabilities, making it ideal for the iterative query-refinement loop of an agent.
- **Search & Synthesis:** The agent can then query this index with natural language, retrieve relevant code snippets, and use them as context for an LLM to answer questions, identify bugs, or suggest architectural improvements.

#### 2.3. Asynchronous, Non-Blocking Core
The entire framework is built on `asyncio`.

- **Benefit:** This is crucial for orchestrating agents that perform many I/O-bound tasks. While one tool is waiting for a response from a web API, the event loop can switch to another task, such as querying the database or processing a file. This results in significantly higher performance compared to a synchronous, threaded model, especially as the number of concurrent operations grows.
- **Implementation:** All core components, from the research process orchestrator to individual tools that perform network requests (using `aiohttp`), are implemented as `async` functions.

#### 2.4. Decoupled Modular Architecture
The `Registry` class is the heart of the system's modularity.

- **Function:** It acts as a service locator, maintaining a mapping from string identifiers to registered classes (Tools, LLMs, Agents). This eliminates the need for direct imports and allows components to be configured and swapped at runtime.
- **Impact:** This design allows an engineer to, for example, replace OpenAI's `gpt-4` with a local, fine-tuned Llama model by simply implementing the `LLM` protocol and updating the registration entry, with no changes required to the agents or logic that consume it.

## 3. Getting Started: A Practical Guide

**Prerequisites:**
- Python 3.10+
- Poetry (v1.2+) for dependency and environment management
- An active OpenAI API Key. For advanced use, keys for other services (e.g., AWS, GitHub) may be required.

**Installation & Configuration:**
1.  **Clone the repository:**
    ```bash
    git clone https://github.com/deadsg235/singularity-io.git
    cd singularity-io
    ```
2.  **Set up the environment:** Poetry will create a virtual environment and install all dependencies.
    ```bash
    poetry install
    ```
3.  **Configure Environment Variables:** Create a `.env` file in the project root. The application uses a library like `pydantic-settings` to load configuration.
    ```dotenv
    # .env
    OPENAI_API_KEY="sk-..."
    DATABASE_URL="sqlite+aiosqlite:///./research.db" # Use a local SQLite DB for first run
    # Optional: For using a local embedding model
    # EMBEDDING_MODEL_PATH="./models/all-MiniLM-L6-v2"
    ```
4.  **Activate the environment:**
    ```bash
    poetry shell
    ```

**Example: Running an In-Depth Repository Analysis**
The following script demonstrates a more advanced use case: asking the agent to analyze its own architecture and identify its main entry point.

```python
# run_self_analysis.py
import asyncio
from singularity_io.agent.researcher import run_research_process, AdvancedSearchSystem
from singularity_io.models.database import initialize_database, ResearchHistory

async def main():
    """
    Initializes the database and runs a research task to analyze the project's own architecture.
    """
    # Ensure the database and tables are created
    await initialize_database()

    objective = (
        "Perform a comprehensive analysis of the `singularity-io` codebase. "
        "Identify the primary entry point for initiating a research task. "
        "Detail the key modules involved in the `run_research_process` and "
        "explain how they interact, specifically mentioning the role of the registry and the searcher."
    )
    
    repo_url = "https://github.com/deadsg235/singularity-io.git"

    print(f"Starting research for objective: '{objective}'")
    
    # This call will trigger the full indexing and research pipeline
    final_report = await run_research_process(
        objective=objective,
        search_system=AdvancedSearchSystem.SEMANTIC_SEARCH,
        repo_url=repo_url
    )
    
    print("\n--- Research Complete ---\n")
    print(final_report)

    # You can also query the database for history
    last_run = await ResearchHistory.get_latest()
    if last_run:
        print(f"\nRetrieved from DB. Status: {last_run.status}")

if __name__ == "__main__":
    asyncio.run(main())
```

## 4. Advanced Use Cases for the Expert

- **Automated Technical Due Diligence:** Point the agent at a target company's public Git repositories to get a high-level overview of their software quality, architectural patterns, and choice of technologies.
- **Large-Scale Refactoring Assessment:** Before embarking on a major refactoring (e.g., migrating from Flask to FastAPI), task an agent to traverse the codebase, identify all usages of the legacy framework, and categorize them by complexity.
- **Automated Competitive Analysis:** Provide the agent with a list of competitor websites and APIs. The agent can use web browsing and API interaction tools to probe these services and synthesize a report on their features and performance.
- **Security Vulnerability Auditing:** Equip the agent with tools that wrap security scanners (like `semgrep` or `trufflehog`). The agent can then run these scanners, analyze the output, and use its codebase understanding to determine the potential impact of the findings.

## 5. Architectural Highlights

The architecture is explicitly designed for extensibility and maintainability. For a full breakdown, please see `ARCHITECTURE.md`.
- **Service Locator Pattern:** The `Registry` acts as a simple but powerful service locator, decoupling components.
- **Asynchronous from the Ground Up:** The choice of `asyncio` allows for high-throughput I/O for agents that interact heavily with external services.
- **Clear Separation of Concerns:** The agent's reasoning loop (`researcher.py`), capabilities (`tools/`), and deep analysis functions (`search/`) are cleanly separated, making the system easier to debug and extend.

This project is in active development. We welcome contributions, feature requests, and rigorous technical feedback.