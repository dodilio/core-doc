# Dodil
Dodil is an augmented data framework designed to accelerate the development of general and LLM-powered applications. It supports any programming language and integrates with any data source.

## Motivation
Large Language Models (LLMs) have emerged as powerful tools for natural language understanding and human-AI interaction. While LLMs excel at generating insights and responding to user queries, significant challenges arise when they are tasked with interacting with complex application data systems. Unlike basic database querying, real-world applications involve intricate architectures, non-standardized interfaces, and domain-specific semantics, making integration with LLMs a non-trivial task.

## Advantage of the Framework:
Dodil accelerates both traditional and AI-powered development, reduces operational overhead, and enforces data integrity at every step. It’s not just another ORM or query builder—it’s a holistic data framework that transforms how you design, manage, and evolve your application’s data model in an AI-driven world

- Cross-Platform & Cross-DB: Works with any language or datastore, ensuring maximum adaptability.
- Low Code & High Velocity: Greatly reduces API code, letting you build and ship features faster.
- Rules-Driven Consistency: Centralizes business constraints, validations, and reactive updates for clean, maintainable architectures.
- Native LLM Support: Bridges the gap between AI prompts and real-world data, cutting down hallucinations and ensuring safe operations.
- Flexible & Extensible: Designed to evolve with your tech stack, scaling from a single database app to a multi-service AI ecosystem.

### Concept
we designed Dodil to enhance the data with 5 fundamental concepts
- [Relation](Relation.md): Identifies connections, such as parent-child relationships and data grouping.
- [Augmentation](Augmentation.md): Defines what needs to be added when creating, viewing, or editing specific data
- [State Management](State-Management.md): Manages the state (e.g., immutability, enumerations) of fields, depending on other fields within the same schema or related schemas (parent or child)
- [Reactivity](Reactivity.md): Ensures that changes to one field trigger appropriate updates to related fields (within the same schema, parent, or child)
- [Prompt Management](Prompt-Management.md): Leverages the above concepts to refine user prompts, translating them into accurate data executions and reducing LLM hallucinations