# Ai Prompt Management

As AI agents become more capable of interpreting user requests and responding with actions, there’s a growing need for a safe, structured, and reliable way to connect these agents to real-world data systems. Ai-Prompt-Management fills this gap by leveraging DODIL’s augmentation and state management rules while integrating the LLM into your existing database schemas.

## Key Goals:
1.	*Reduce Hallucination*: LLMs often invent SQL queries or misinterpret schemas. With Ai-Prompt-Management, queries are validated and generated using real DODIL schema definitions.
2.	*Enforce Business Rules*: Augmentation (e.g., requiring certain child records) and State Management (e.g., immutable fields) automatically apply to AI-generated read/write requests, ensuring data consistency.
3.	*Streamline AI-Driven Workflows*: Whether the user wants to query, create, or update data, the module generates the necessary SQL or prompts to the user for missing info—without manually crafting complex logic.

## Core Concepts
### Detecting SQL via dodil.ai.detectSql(prompt)
One of the biggest challenges with LLMs is “hallucination”—where the AI invents fields, tables, or syntax that doesn’t exist. detectSql mitigates this risk:
```JavaScript
const { query, parameters } = dodil.ai.detectSql(prompt);
```
*	prompt: A user’s natural language request, e.g., “Find all orders that are pending.”
*	detectSql: Parses the prompt, consults DODIL’s known schemas, and outputs a relevant, valid SQL statement (plus parameters).
*	Reduced Hallucination: Because it’s grounded in actual schema definitions, the AI is less likely to produce invalid or fictitious fields/queries.

### Conformance to Augmentation & State Management
Ai-Prompt-Management seamlessly applies the same rules you’ve defined in DODIL’s Augmentation and State Management:
*	Augmentation: If your schema requires certain child records (e.g., Order must have at least one OrderItem), the module makes sure any create or read requests from the LLM match those requirements.
*	State Management: If a field is immutable when the order is in_process, any LLM attempt to edit that field triggers an error or a prompt for further action.

This allows your LLM to respect all the constraints you’ve designed for your data—without custom-coded logic in the AI layer.

#### Example Workflows

##### Reading Data (SQL Generation)

Scenario: User wants to “View the Order #123 along with its items.”
1.	LLM Prompt: “Show me the details of order #123, including items.”
2.	Ai-Prompt-Management:
   *	Uses detectSql(prompt) to parse the request.
   *	Finds relevant schemas (ORDER, ORDERITEM), checks augmentation rules (e.g., auto-include items if ORDER is augmented to always fetch them).
   *	Generates a valid SQL query (e.g., SELECT ... FROM ORDER JOIN ORDERITEM... WHERE ORDER.id = 123;).
3.	Result: The module returns the data to the LLM, which can then present it to the user in a natural language format.

#### Editing Data While State is Locked
Scenario: User wants to edit the quantity of an OrderItem where the Order.status is in_process, and the quantity field is immutable once in this state.
1.	LLM Prompt: “Change the quantity of item #9999 to 20.”
2.	Ai-Prompt-Management:
   *	Checks the State Management rules for ORDER and ORDERITEM.
   *	Notices that since ORDER.status is in_process, ORDERITEM.quantity is immutable.
   *	Tells the LLM to inform the user: “Cannot change quantity because the order is already in process.”
   *	LLM can then decide whether to prompt the user about alternate actions or to refuse the request.

## Putting It All Together

Ai-Prompt-Management acts as a companion to DODIL, ensuring that any LLM-driven queries or updates:
*	Conform to your schema design and relationships.
*	Honor your Augmentation (e.g., mandatory child records, nested data) and State Management rules (e.g., immutable fields).
*	Reduce hallucination by grounding the AI in real data definitions when generating SQL.
*	Prompt the user for missing data or disclaimers when constraints are not met.

By integrating Ai-Prompt-Management into your AI-powered workflows, you can confidently allow LLMs to create, read, and update data without manually coding complex validations or checks—DODIL and Ai-Prompt-Management handle it behind the scenes.