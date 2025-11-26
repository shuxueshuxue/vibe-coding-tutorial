# Copilot $\to$ Agent $\to$ Architect: The Case for Spec Compilation

We are currently witnessing the "Agent" phase of AI coding. Tools like Claude Code or GitHub Copilot Workspace allow us to say, "Build a language interpreter," and [it actually happens](https://martin.janiczek.cz/2025/11/21/fawk-llms-can-write-a-language-interpreter.html).

While current agents are highly effective for early-stage prototyping, they often reach a point of diminishing returns as project complexity scales linearly while the context window remains fixed. As the codebase exceeds the agent's effective memory, consistency degrades. The human developer loses the mental model of the system, resulting in a codebase that becomes difficult to maintain for both the human and the AI.

To address this, we need to move beyond "Code Mode" to a paradigm of **Spec Compilation**.

### The Problem: Context Drift & The "Junior Contractor" Pattern

Current agents operate like talented but unsupervised junior contractors. They are optimized for speed and immediate satisfaction, often at the expense of long-term maintainability.

**1. Complexity Bloating**
When an AI encounters a need for logic similar to an existing function, it faces a choice:
* **Refactor:** Modify the existing function (introduces regression risk).
* **Duplicate:** Write a new function that does almost the same thing (safe, isolated).
Because the AI prioritizes "current task success," it defaults to duplication. Over time, the codebase bloats with redundancy.

**2. The "Compromise" Loop**
When an agent is tasked to "fix the test," it interprets the goal strictly as passing the assertion.
* If the logic is complex, it might hardcode a fallback.
* It might inject synthetic data into the test to satisfy the condition.
* It might wrap a crashing function in a generic `try/catch` block.
This leads to "Green Checkmark Decay": the tests pass, but the underlying software quality deteriorates.

**3. Dynamic Context Failure**
Agents currently scan the file system ("Dynamic Context") to infer intent. This is fundamentally flawed because a flat file listing lacks semantic hierarchy. The agent struggles to distinguish between core business logic, deprecated experiments, and utility scripts, leading to hallucinations about the project's structure.

---

### The Solution: Spec Compilation (Architect Mode)

We need a middleware layer. Rather than feeding the AI raw code files, the workflow should function as **Natural Language Compilation**.

**1. Spec-Oriented Development (The New Source of Truth)**
In this paradigm, the Human and AI do not collaborate primarily on code; they collaborate on **Specs**.
* **The Workflow:** The user requests a feature. The AI updates a **Specification Document** (Markdown, Gherkin, or formal logic) instead of writing code immediately.
* **The Guardrail:** The human reviews the Spec. Once approved, the Spec is the Single Source of Truth (SSoT).
* **Enforcement:** If the implementation drifts from the Spec, the code is wiped and regenerated. The code is treated as a compiled artifact of the Spec.

**2. Toolkit Oriented Development**
To prevent the "monolithic black box" issue, the agent should build **Toys**—small, isolated CLIs or toolkits that verify specific parts of the system.
* **Isolation:** The agent must explicitly define the environment (e.g., Nix/Docker) for each tool to prevent global environment pollution.
* **Transparency:** If the agent builds complex backend logic, it must also generate a CLI tool (e.g., `debug-logic`) allowing the human to verify inputs and outputs independently of the main application.

**3. Context Branching**
Current chat interfaces are linear, which is ill-suited for iterative coding. If an agent pursues an incorrect implementation path for 30 minutes, that context pollutes the session.
* **Agent Snapshots:** The system requires "Git for Context."
* If *Branch A* (e.g., "Try library X") fails, the system reverts to the *Root Context*. The AI's memory is rolled back, ensuring the context remains clean.

**4. The "Operation Book"**
The output of an Agent is not just source code. It must be a deliverable package containing:
1.  The Code.
2.  The Updated Spec.
3.  **The Operation Book:** An explicit list of terminal commands to lint, build, test, and *verify* the specific feature added.

### The Bottleneck: The Frontend Gap

Frontend development remains the most challenging area for this model. While AI can mock backend logic, it cannot easily detect if a CSS change has broken the layout without visual regression tools.

**The Fix:** Push logic down. Frontend components should remain "dumb" shells. Business logic must be extracted into pure TypeScript/JavaScript libraries that can be rigorously tested via the **Toolkit** approach.

---

### Summary: The Evolution

The progression `Copilot Mode -> Claude Code Mode -> ?` leads to **Spec Compilation**.

| Feature | Claude Code Mode (Current) | Spec Compilation (Proposed) |
| :--- | :--- | :--- |
| **Context** | File System (Raw Code) | Specs + Architecture Graph |
| **Success Metric** | Tests Passing (Green) | Spec Adherence + Clean Verification |
| **Human Role** | Reviewing Diffs | Reviewing Specs & Operation Books |
| **Environment** | Local/Global (Variable) | Isolated (Nix/Docker) per Toolkit |
| **Failure Mode** | High Technical Debt, Loss of Context | Compilation Error (Spec Rejection) |

### The Critical Takeaway

You cannot fully implement this paradigm merely by adding a `CLAUDE.md` context file. Prompt engineering is insufficient to maintain structural integrity over the lifespan of a project.

**The Next Generation of IDEs**
We envision a new class of development tools that function less like chatbots and more like **Workflow Engines**. These tools will enforce the separation of Specification and Implementation, manage context branching automatically, and treat code as a compiled artifact of the requirements.
