# Dodil in Practice

<!--Writerside adds this topic when you create a new documentation project.
You can use it as a sandbox to play with Writerside features, and remove it from the TOC when you don't need it anymore.-->

## Write with Dodil
Below illustrative example of a simple Node.js application demonstrating all major Dodil components—Relations, Augmentation, State Management, Reactivity, and Prompt Management. It’s meant to highlight the concepts rather than serve as production-ready code. Feel free to adapt it to your specific tech stack
```JavaScript
/**************************************************
 * 1) IMPORT & SETUP
 **************************************************/
const dodil = require('@dodil');

/**
 * Connect to your data store of choice.
 * Example: 'mongo-db', 'sql', or a custom connector.
 */
const dodilStore = dodil.dataStore('mongo-db', 'mongodb://localhost:27017/mydb');


/**************************************************
 * 2) DEFINE SCHEMAS (Relations)
 **************************************************/
// -- Parent Schema: ORDER
const dodilOrder = dodil.schema('ORDER', {
  status: {
    type: 'string',
    enum: ['pending', 'in_process', 'completed'],
    default: 'pending'
  },
  totalAmount: {
    type: 'number',
    default: 0
  }
});

// -- Child Schema: ORDERITEM
const dodilOrderItem = dodil.schema('ORDERITEM', {
  orderId: { type: 'id', refer: 'ORDER' }, // Relationship (parent-child)
  productName: { type: 'string', required: true },
  quantity: { type: 'number', min: 1 },
  status: {
    type: 'string',
    enum: ['pending', 'in_process', 'completed'],
    default: 'pending'
  }
});


/**************************************************
 * 3) AUGMENTATION
 **************************************************/
/**
 * Let's ensure that creating an ORDER always requires
 * at least one ORDERITEM. We'll define that in "slf" mode.
 */
dodilOrder.augment
  .toCreate('slf', {
    select: '!status',  // Exclude 'status' from the creation payload (auto-handled or can be set automatically)
    refer: {
      schema: 'ORDERITEM',
      as: 'items',
      minItems: 1
    }
  });


/**************************************************
 * 4) STATE MANAGEMENT
 **************************************************/
/**
 * We'll lock (immutable) the quantity field if
 * the ORDERITEM.status is 'in_process' or 'completed'.
 */
dodilOrderItem.state.create(
  {
    field: 'status',
    operator: 'in',
    value: ['in_process', 'completed']
  },
  {
    on: 'quantity',
    effectAs: { immutable: true }
  }
);

/**
 * We can also lock (immutable) *all* fields in ORDERITEM
 * if the parent ORDER is 'completed'.
 */
dodilOrderItem.state.create(
  {
    field: '$parent.status',
    operator: 'eq',
    value: 'completed'
  },
  {
    on: '*',
    effectAs: { immutable: true }
  }
);


/**************************************************
 * 5) REACTIVITY
 **************************************************/
/**
 * Whenever an ORDERITEM.status changes to 'completed', we add a listener in the parent OrderId
 * we check if all items under that ORDER are completed.
 * If yes, we set ORDER.status to 'completed'.
 */
// Reactivity Rule on ORDER
dodilOrder.reaction.create(
  "child",                   // Trigger originates from child entities (ORDERITEM)
  "modified",                // Event type (e.g., when ORDERITEM is updated)
  {
    "$modifiedFields": "status",           // React only when status is modified
    "$eventObject.status": ["completed"]   // React only when the new status is 'completed'
  },
  {
    action: "custom",                      // Custom logic to handle cascading updates
    function: "checkAllItemsAndSetOrder"
  }
);

dodil.addCustomAction("checkAllItemsAndSetOrder", async (eventObject) => {
  const orderId = eventObject.parentId;  // Assume the eventObject provides the parent ID (Order)
  
  // Fetch all items for this Order
  const allItems = await dodilOrderItem.find({ orderId });

  // Check if all items are completed
  const allCompleted = allItems.every(item => item.status === "completed");

  // Update Order status if all items are completed
  if (allCompleted) {
    await dodilOrder.update(orderId, { status: "completed" });
    console.log(`Order ${orderId} status set to completed.`);
  }
});


/**************************************************
 * 6) PROMPT MANAGEMENT (AI Integration)
 **************************************************/
/**
 * We'll demonstrate a simple example where a user/LLM
 * might say: "Find all completed orders."
 * Using `dodil.ai.detectSql`, we can parse the prompt
 * into a valid query grounded in the actual schema.
 */
async function aiQueryExample() {
  const prompt = "Find all completed orders";
  const { query, parameters } = dodil.ai.detectSql(prompt);

  console.log("Generated Query:", query);
  console.log("Parameters:", parameters);

  // Hypothetical direct query (depending on your DB setup):
  // const results = await dodilOrder.queryRaw(query, parameters);
  // console.log("Query Results:", results);
}


/**************************************************
 * 7) USAGE DEMO
 **************************************************/
(async () => {
  try {
    // 7.1 Creating an ORDER + ORDERITEM(s)
    // The augmentation rule says we must supply at least 1 item
    const newOrderData = {
      totalAmount: 150,        // from user input or logic
      items: [                 // child records for augmentation
        { productName: 'Widget A', quantity: 2 },
        { productName: 'Widget B', quantity: 1 }
      ]
    };
    const createdOrder = await dodilOrder.create(newOrderData);
    console.log("Created ORDER + items:", createdOrder);

    // 7.2 Modifying an ORDERITEM
    // If we set the item to 'completed', reactivity might check if all are done
    const firstItemId = createdOrder.items[0]._id;
    const updatedItem = await dodilOrderItem.update(firstItemId, {
      status: 'completed'
    });
    console.log("Updated first ORDERITEM to 'completed':", updatedItem);

    // 7.3 Trying the AI prompt for reading data
    await aiQueryExample();

  } catch (error) {
    console.error("Error in demonstration:", error);
  }
})();
```

## Dodil vs Traditional SQL
- Dodil Sample: ~70–100 lines
- Traditional SQL (plus custom validations, triggers, AI logic): ~150–300 lines (or 2–3× more)

Using Dodil often cuts your line count significantly—not only saving development time but also yielding code that’s more maintainable and easier to evolve.

### Traditional SQL-Based Approach (Likely 2–3× More Lines)
If you were to build the same functionality using a standard SQL approach without a unifying framework like Dodil, you’d typically write:
1.	Schemas/Models:
 - Define database tables (DDL) and possibly an ORM or model class for each table.
 - Then replicate “child references” in code or via migrations.
2.	API Routes & Validation:
   * Separate controllers or service layers to handle createOrder, addItems, updateItemStatus, etc.
   * Manual validations (e.g., “ensure at least one item is added when creating an Order”).
3.	Triggers or Event Handlers (for reactivity):
   * Might require SQL triggers or custom code to detect changes in ORDERITEM.status and update the parent ORDER.
   * Or you’d do this in your service layer, explicitly checking all child items each time one changes.
4.	LLM Integration:
   * You’d need to parse AI prompts separately, generate SQL strings carefully, and ensure they don’t try to access non-existent columns. 
   * You’d handle “partial creation” logic manually—detecting missing fields or child records and prompting the user again.
5. 	State Management:
   * You’d likely code these rules in multiple places (in the service layer or controllers): “If status is in ‘in_process’ or ‘completed’, lock quantity.”

Collectively, you’d write more boilerplate—especially for validations, reactive triggers, and LLM prompt handling. This might easily double or triple your lines of code to 150–300 lines (or more), depending on:
*	How you handle triggers (SQL triggers vs. code-based event watchers).
*	How you orchestrate partial user input (like requiring one item at order creation).
*	How you integrate an LLM or other AI system (writing a custom parser or manually checking the schema in your code).
