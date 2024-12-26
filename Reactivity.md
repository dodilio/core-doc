# Reactivity

In DODIL, Reactivity defines event-driven rules that respond to changes in your data:
1. Reactivity should always focus on the effect location, not the trigger origin (closely resembles a pub/sub (publish-subscribe) )
2. Automatic Updates: When a field is created, updated, or deleted, other fields or records may need to be modified or synchronized.
3. Multi-Schema Coordination: Reactivity allows you to propagate changes across parent-child boundaries or between related schemas.
4. High Efficiency: Reactivity logic in DODIL is designed to be multi-phase, avoiding unnecessary database queries until conditions are met.


For example, if an Order status changes to completed, you might automatically update all OrderItems to completed. Similarly, if an OrderItem quantity changes, you might want to recalculate the Order total.

Why Reactivity?
 * Consistency: Automatically keep data in sync when changes occur, removing the burden of manual updates.
 * Maintainability: Define reactive rules in a clear, declarative manner, making the system easier to understand and extend.
 * Reduced Boilerplate: Centralize logic that would otherwise be spread across multiple services or client-side code.

## Key Concepts

1.	Reaction Rules
   * You define these using schema.reaction.create(...).
   * Each rule specifies:
     1. Which schema or target the rule belongs to.
     2. When to trigger (create, update, delete).
     3. Conditions (e.g., specific fields changed).
     4. Actions (e.g., set a field, run a custom function).
2.	Event Object `$eventObject`
	•	Represents the record or fields causing the event.
	•	If you edited an OrderItem, $eventObject contains the newly edited fields and their values.
3.	Modified Fields `$modifiedFields`
	•	An array of fields that changed during an update.
	•	Useful for checking if a specific field (e.g., status) was altered before triggering a reaction.
4.	Multi-Phase Filtering
	•	Phase 1: Filter rules by the schema and event type (created, modified, or deleted).
	•	Phase 2: Check if conditions on `$eventObject` and `$modifiedFields` are satisfied (e.g., status changed to in_process).
	•	Phase 3: Perform any needed database queries if the first two filters pass (optional, e.g., if the reaction must target parent or child records).
5.	Reaction Actions
	•	An action defines how to update or affect other fields or records.
	•	Common patterns include "action": "set", which is setting a field in the current schema
    •	You can add action : "custom" to add a function that executes
      

## Basic Example: Syncing Status from Child to Parent
Suppose we have an OrderItem referencing an Order. When an OrderItem.status changes to completed, we want to check if all OrderItems for that parent Order are completed. If they are, we automatically update the parent Order.status to completed.

### Setup Schema
```javascript
// parent
const dodilOrder = dodil.schema('ORDER', {
  status: { type: 'string', enum: ['pending', 'in_process', 'completed'] },
});

// child
const dodilOrderItem = dodil.schema('ORDERITEM', {
  orderId: { type: 'id', refer: 'ORDER' },
  status: { type: 'string', enum: ['pending', 'in_process', 'completed'] },
});
```

### Basic Flow
- When ORDERITEM.status changes to completed, a reactivity rule checks if all items for the same orderId are now completed.
- If all are completed, we update the parent ORDER.status to completed.
```javascript
// Reaction Rule in Order
dodilOrder.reaction.create(
  // Reaction on 'modified' event
  {
    event: 'modified',
  },
  // React when => condition object
  {
    "$modifiedFields": "status",       // Only if status changed
    "$eventObject.status": ["completed"],
  },
  // React how
  {
    action: 'custom',
    function: 'updateChildrenStatus'
  }
);
// Implement the Custom Action
dodil.addCustomAction('updateChildrenStatus', async (eventObject) => {
  const order = eventObject; // The updated ORDER
  const { _id } = order;

  // Update all ORDERITEMs to 'completed' as well
  await dodilOrderItem.updateMany(
    { orderId: _id },
    { status: 'completed' }
  );
});
```

## Basic Example of a Direct Set Reaction
Sometimes you just need a simple rule, like syncing a child’s status to its parent without custom logic. For example, if OrderItem.status changes to in_process, set the parent’s status to in_process.

```JAVASCRIPT
dodilOrderItem.reaction.create(
  // Reaction is on `ORDERITEM` schema
  "parent",        // or a config object specifying the target
  "modified",      // event
  {
    "$modifiedFields": "status",
    "$eventObject.status": ["in_process"]
  },
  {
    // Direct set action
    action: "set",
    fields: {
      // we want to set the parent's `status` to the child's new status
      status: "$eventObject.status"
    }
  }
);
```

Note: Depending on your DODIL setup, specifying "parent" as the first parameter may indicate you’re updating the parent schema. The exact signature can vary, but conceptually, you’re telling DODIL: “When ORDERITEM is modified, set the parent’s status to the child’s status.”


## Custom Actions
Not all reactivity rules are simple “set” or “copy” updates. You may need to run more complex logic:
- Aggregations: Summing up child quantities into a parent’s total.
- Conditional Checks: Only update a parent field if all children meet certain criteria.
- Multi-Step Updates: For instance, updating a grandparent schema after iterating through multiple child schemas.

### How to Define a Custom Action
1. Create the reaction rule with "action": "custom" and "function": "myFunctionName".
2. Implement that function using dodil.addCustomAction(myFunctionName, callback).

```JavaScript
// Reaction rule in ORDER
dodilOrder.reaction.create(
  {
    event: 'modified',
  },
  {
    "$modifiedFields": "status",
    "$eventObject.status": ["completed"],
  },
  {
    action: 'custom',
    function: 'updateChildrenStatus'
  }
);

// Custom action definition
dodil.addCustomAction('updateChildrenStatus', async (eventObject) => {
  const order = eventObject; // The updated ORDER
  const { _id } = order;

  // Update all ORDERITEMs to 'completed'
  await dodilOrderItem.updateMany({ orderId: _id }, { status: 'completed' });
});
```
This pattern separates what triggers the reaction (the reaction rule) from how it’s carried out (the custom action).


## Additional Pattern 
This pattern separates what triggers the reaction (the reaction rule) from how it’s carried out (the custom action).
1. Multiple Conditions
   You can pass arrays or multiple keys in the conditions object:
```JavaScript
{
  "$modifiedFields": ["status", "quantity"],
  "$eventObject.status": ["in_process", "completed"],
}
```

2. Delete Events
   When a record is deleted, you might want to decrement a parent’s total or remove references. Just define event: 'deleted' in your reaction filter.
3. Chaining Multiple Reaction Rules
   You can define multiple reaction.create(...) calls on the same schema to handle different scenarios. For instance, one rule might handle changes to status, while another handles changes to quantity.