# Agentic AI: An Overview

This post provides an overview of Agentic AI based on key points outlined in my notes. The content is structured to explain the concepts clearly and concisely, incorporating relevant diagrams for illustration.

![Agentic AI Thumbnail](https://raw.githubusercontent.com/XyonX/portfolio-content/main/Blogs/agenti-ai/images/thumb.webp)

## Definition of Agentic AI

Agentic AI refers to applications that utilize large language models (LLMs) to perform tasks. It does not possess independent thinking capabilities but excels in task execution. Agentic AI systems rely on LLMs to determine and generate tasks. The process involves submitting a task to the LLM, which responds with subtasks for the agent to complete. The agent executes these, and the LLM generates a response based on the results.

This approach enables AI to handle practical applications beyond mere text generation.

![What is Agentic AI](https://raw.githubusercontent.com/XyonX/portfolio-content/main/Blogs/agenti-ai/images/what-is-agentic-ai.webp)

## Rationale for Using Agents Alongside LLMs

LLMs are fundamentally functions that process text input to produce text output; they cannot perform actions independently. Agents serve as intermediaries that interpret LLM outputs and execute the required tasks, such as API calls or data processing. This integration allows LLMs to extend their utility into actionable scenarios.

## Distinction Between Agentic AI and AI Agents

An AI agent is a standalone software component that interacts with an LLM to manage tasks. In contrast, Agentic AI describes a system comprising multiple agents collaborating to achieve complex objectives. Examples include a master agent coordinating with specialized agents for web searches, personal assistance, document handling, or coding support.

![Agentic AI Diagram](https://raw.githubusercontent.com/XyonX/portfolio-content/main/Blogs/agenti-ai/images/agentic-ai-diagram.jpeg)

## Operational Mechanism of AI Agents

AI agents function through a structured process:

1. Receive the input task or message.
2. Compile a list of available tools.
3. Create a payload including the message, conversation history, and tool list, then submit it to the LLM.
4. The LLM evaluates the payload and determines if a tool is needed, specifying parameters if applicable (e.g., for a weather query, it may call a weather tool with location details).
5. Execute the tool, which could be local or external, and return the result to the LLM.
6. The LLM processes the result and either generates a final response or requests additional tool calls, continuing in a loop until resolution.

This iterative method ensures comprehensive task handling.

![Agentic AI Walkthrough](https://raw.githubusercontent.com/XyonX/portfolio-content/main/Blogs/agenti-ai/images/agentic-walkthrough.png)

## Essential Features of AI Agents

Effective AI agents incorporate several core features:

- **Reasoning and Planning**: Agents use LLMs to decompose tasks, prioritize steps, and develop strategies.
- **Tool Use and Calling**: Enables interaction with external resources to gather or process information.
- **Memory**:
  - **Short-term Memory**: Maintains context for the current session, functioning as a temporary storage.
  - **Long-term Memory**: Retains historical data for improved future performance.
- **Action Execution**: Performs the operations specified by tool calls.
- **Learning and Adaptation**: Advanced agents refine approaches based on prior outcomes.
- **Communication**: Facilitates natural language interactions with users or other agents.

These features enhance the reliability and versatility of Agentic AI systems.

## Conclusion

Agentic AI represents a significant advancement in applying LLMs to real-world tasks through agent-based systems. It focuses on execution and collaboration to achieve efficient results. For those interested in AI development, exploring agent implementation can provide valuable insights.