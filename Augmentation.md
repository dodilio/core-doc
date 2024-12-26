# Augmentation

In DODIL, Augmentation defines additional rules and requirements around data when performing operations such as create, view, or edit. It allows you to:
1.	Specify required data that must be present at certain stages (e.g., automatically creating child records when creating a parent).
2.	Combine multiple schemas into a single “augmented” data structure for streamlined handling.
3.	Enforce multi-layer validation:
4. Augmented validation: Rules you define for the combined data structure.
5.	Per-schema validation: The existing schema constraints on each individual schema.

Only if both layers pass do the operations (create/view/edit) succeed.
## Key Concepts
1.	Augmentation Targets:
   - **toCreate** – When new data is being created
   - **toView** – When data is being read or displayed
   - **toEdit** – When existing data is being updated.

2.	Augmentation Contexts:
   - **slf**: The augmentation is triggered directly on the schema itself.
   - **nested**: The augmentation is triggered when the schema is created/viewed/edited as part of another schema’s augmentation process.

3.	Combined Schema:
When an augmented operation occurs, DODIL merges the parent’s augmentation rules with the child schemas’ definitions, ensuring both the parent and child-level validations are enforced.

4.	select and refer:
   - select: Which fields in the current schema are requested or manipulated (e.g., !status means “all fields except status”).
   - refer: Points to another schema that must be included or created as part of this operation (e.g., referencing an ORDERITEM within an ORDER).


## Simple Example: Order Creation Must Include Order Items

### Scenario

You want to ensure that every time a new Order is created, at least one Order Item is also created. We’ll define this rule on the Order schema using Augmentation.

```javascript
    // 1. Import DODIL
const dodil = require('@dodil');

// 2. Define the Order schema
const dodilOrder = dodil.schema('ORDER', {
  status: {
    type: 'string',
    enum: ['pending', 'inprocess', 'completed'],
  },
  totalAmount: {
    type: 'number',
    default: 0,
  },
});

// 3. Define the OrderItem schema
const dodilOrderItem = dodil.schema('ORDERITEM', {
  orderId: {
    type: 'id',
    refer: 'ORDER'
  },
  productName: {
    type: 'string',
    required: true,
  },
  quantity: {
    type: 'number',
    min: 1,
  },
  status: {
    type: 'string',
    enum: ['pending', 'inprocess', 'completed'],
  },
});
   ```

#### Adding Augmentation Rules
Now, we configure Order’s toCreate augmentation so that Order Items are required at creation time. DODIL merges these rules to create a single combined schema for the create operation.

```javascript
dodilOrder.augment
  .toCreate('slf', {
    // Select all fields of ORDER except 'status'
    // This means during creation, the 'status' field might be excluded or auto-handled
    select: '!status',

    // Refer to the ORDERITEM schema
    refer: {
      schema: 'ORDERITEM',  // must exist in DODIL
      as: 'items',          // how we name it in the combined data
      minItems: 1,         // at least one ORDERITEM must be provided
    },
  });
   ```

##### Why slf?
We specify 'slf' (self) because this augmentation rule applies when we directly create an ORDER. If an ORDER is created within another schema’s process, we might define 'nested' rules—but that’s a more advanced scenario.


##### What Happens at Runtime?
1. Client Submission: When you call dodilOrder.create(...), you provide both Order fields and a list/array of Order Items under items.
2. Combined Validation:
   * DODIL first checks if your request meets the augmentation rules (at least one ORDERITEM, etc.).
   * Then, DODIL validates each item against the ORDERITEM schema rules.
   * If all validations pass, the system proceeds to create the Order and each associated OrderItem.


## Example Flow: Creating an Order with Items
```JavaScript
    // Example payload you might send to DODIL
const requestBody = {
  totalAmount: 120.5,
  items: [
    {
      productName: 'Widget A',
      quantity: 2,
      status: 'pending'
    },
    {
      productName: 'Widget B',
      quantity: 1,
      status: 'pending'
    }
  ]
};

try {
  // This uses the augmentation rules defined in `toCreate('slf')`
  const result = await dodilOrder.create(requestBody);

  // DODIL automatically:
  // 1. Validates `ORDER` fields (excluding status).
  // 2. Validates each `ORDERITEM` provided in `items`.
  // 3. Inserts ORDER and ORDERITEM(s) if everything passes.

  console.log('Order and OrderItems created:', result);
} catch (err) {
  console.error('Failed to create:', err);
}
```

## View Augmentation (toView)

### Why Use toView?
You do not need to specify on every sql query you make the join - it handles that for you
* You might want to automatically include related data (like items) whenever you fetch an ORDER
* Or limit which fields appear when retrieving an entity (e.g., hiding sensitive fields).


Example: Viewing an Order with Items

```JavaScript
   // Reusing the same ORDER schema, we define:
dodilOrder.augment
  .toView('slf', {
    // We'll select all ORDER fields
    select: '*',

    // We'll also fetch its order items
    refer: {
      schema: 'ORDERITEM',
      as: 'items',
      fields: ['productName', 'quantity', 'status'], 
      // We only want these fields from the ORDERITEM
    }
  });
```

Now, whenever you run:

```JavaScript
   const orderWithItems = await dodilOrder.view(orderId);
```

DODIL will:
1. Fetch the ORDER by orderId.
2. Include (as: "items") the relevant ORDERITEM documents referencing that order.
3. Return them in a single combined result.


## Nested Augmentation Example

### Scenario: toCreate('nested')

“Nested” augmentation is triggered when you’re creating a schema as part of another schema’s augmentation flow. For instance, if a Shipment schema automatically creates a new Order inside it, you might define how ORDER must be created in that context.

```JavaScript
// We have a SHIPMENT schema
const dodilShipment = dodil.schema('SHIPMENT', {
  trackingNumber: { type: 'string' },
  carrier: { type: 'string' },
});

// Suppose we want to create an ORDER as part of SHIPMENT creation
dodilShipment.augment
  .toCreate('slf', {
    select: '*',
    refer: {
      schema: 'ORDER',
      as: 'orderData',
      required: true,
    }
  });

// Now, inside the ORDER schema, define what happens if it's created in a nested scenario
dodilOrder.augment
  .toCreate('nested', {
    select: 'status, totalAmount', // or define a custom selection for a nested creation
    refer: {
      schema: 'ORDERITEM',
      as: 'items',
      minItems: 1
    }
  });
```

* When you call dodilShipment.create(...), it triggers nested creation of ORDER. Because the SHIPMENT augmentation rules say “We’re creating an ORDER inside me,” the ORDER schema then uses its nested rules (toCreate('nested')) instead of the slf rules.

## Two Layers of Validation
1. Combined Schema Validation
   * The augmented (merged) rules that specify cross-schema constraints (like minItems on the child array).
2. Individual Schema Validation
   * Each schema’s own constraints (required, enum, min, max, etc.).

Both must pass for the operation to complete.

### Example of Validation Failure
1. If an Order is created with minItems: 1 in the augmentation but you provide no items, you fail the combined schema validation.
2. If an OrderItem is provided with a negative quantity, you fail the individual schema validation.


## Conclusion
Augmentation is a powerful tool that transforms how you handle multi-schema operations. Whether you’re ensuring every Order has at least one OrderItem, or automatically fetching nested details when viewing an entity, DODIL’s augmentation capabilities streamline these processes.
