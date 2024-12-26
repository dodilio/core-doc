# State Management

In DODIL, State Management allows you to dynamically enforce rules on fields depending on conditions. For example:
1.	Locking (immutable) a field if a related status is set to a certain value.
2.	Making a field required or optional based on other field values.
3.	Restricting valid choices (enumSubset) for a field once certain criteria are met.

At runtime, DODIL computes these states into a special object, `$state, which indicates how each field behaves under current conditions.

example 
- simple example: when status of order Items is changed 
- quantity of orderItems can be changed once the status of order is moved to "['pending', 'in_process']"

## Key Concepts
1.	Conditions
   * Each condition checks a field or a parent’s field using an operator (eq, in, lt, etc.) and a value
   * If the condition is true, the specified effect is applied to target fields.
2. Effects
   * immutable: true: You cannot change this field value after the condition is met.
   * required: true: This field must have a value
   * enumSubset: Restricts the valid enum values for that field.
3. on Field
   * Indicates which field(s) to apply the effect to (e.g., quantity, status, or even * for all fields).
4. References to Parent/child Fields
   * You can reference a parent schema’s field via "$parent|parent(nth)|parent('schema', nth).fieldName" or "$parent|child(nth)|child('schema', nth).fieldName".
   * Allows child/parent constraints based on the relation’s state.
5. Computed `$state`
   * At runtime, each field’s final state is compiled into a \$state.fields.{fieldName} object.
   * This includes any combination of immutable, required, hidden, enumSubset, etc

## Defining State Rules
You define state rules by calling:
```JavaScript
    YourDodilSchema.state.create(
      // condition(s),
      // effect(s)
    );
);
```
You can chain multiple create() calls to add multiple conditions. When conditions overlap, their effects merge, potentially creating a combined or more restrictive state.

## Basic Example: Same-Schema Conditions
Consider an OrderItem schema where the quantity field must be immutable once the item status is either in_process or completed.

```JavaScript
// 1. Define the schema
const dodilOrderItem = dodil.schema('ORDERITEM', {
  orderId: { type: 'id', refer: 'ORDER' },
  productName: { type: 'string', required: true },
  quantity: { type: 'number', min: 1 },
  status: {
    type: 'string',
    enum: ['pending', 'in_process', 'completed'],
  },
});

// 2. Add a state rule:
dodilOrderItem.state.create(
  // Condition
  {
    field: 'status',
    operator: 'in',
    value: ['in_process', 'completed'],
  },
  // Effect
  {
    on: 'quantity',
    effectAs: { immutable: true },
  }
);
```

### What Happens at Runtime?
* Whenever you query or edit an **ORDERITEM**, DODIL evaluates whether status is in `['in_process', 'completed']`.
* If true, the quantity field becomes immutable—it cannot be changed.
* This rule is reflected in `$state.fields.quantity.immutable = true`.


## Parent-Child Relationship Example
In some cases, a child’s fields are controlled by the parent schema’s state. For instance, if the Order (parent) is already completed, you may want to lock down all fields in the OrderItem (child).
```JavaScript
// 1. The ORDER schema
const dodilOrder = dodil.schema('ORDER', {
  status: {
    type: 'string',
    enum: ['pending', 'in_process', 'completed'],
  },
  totalAmount: { type: 'number', default: 0 },
});

// 2. The ORDERITEM schema
const dodilOrderItem = dodil.schema('ORDERITEM', {
  orderId: { type: 'id', refer: 'ORDER' },
  productName: { type: 'string', required: true },
  quantity: { type: 'number', min: 1 },
  status: {
    type: 'string',
    enum: ['pending', 'in_process', 'completed'],
  },
});

// 3. State rule referencing the parent’s status
dodilOrderItem.state.create(
  {
    field: '$parent.status',
    operator: 'eq',
    value: 'completed',
  },
  {
    on: '*',
    effectAs: { immutable: true },
  }
);
```

### Explanation
* Here, field: '$parent.status' checks the status field of the parent ORDER.
* If it’s completed, all fields (on: '*') in ORDERITEM become immutable.
* You cannot change quantity, productName, or status once the parent is completed.

## Multiple Conditions and Effects
You can define multiple conditions with multiple effects. For example, you might make a field both required and immutable if certain conditions are met.

```JavaScript
dodilOrderItem.state.create(
  [
    { field: 'status', operator: 'eq', value: 'in_process' },
    { field: 'quantity', operator: 'gt', value: 5 },
  ],
  {
    on: 'productName',
    effectAs: { required: true, immutable: true },
  }
);
```

* This example enforces that if status == in_process AND quantity > 5, then productName is both required and cannot be changed (immutable).

## Enum Subsets
The enumSubset property refines allowable values for a field at runtime. Suppose your schema’s status field has an overall set ['pending', 'in_process', 'completed'], but once the quantity > 10, you can only choose 'in_process' or 'completed'.

```JavaScript
    dodilOrderItem.state.create(
  {
    field: 'quantity',
    operator: 'gt',
    value: 10,
  },
  {
    on: 'status',
    effectAs: {
      enumSubset: ['in_process', 'completed'],
    },
  }
);
```
* Normally, status can be any value in `['pending', 'in_process', 'completed']`.
* If quantity > 10, the only valid states become `['in_process', 'completed']`.

## How DODIL Computes \$state
Whenever you create, view, or edit data, DODIL evaluates all applicable conditions in the schema’s State Management definitions. For each field, it compiles the results into:
```JavaScript
{
  $state: {
    fields: {
      quantity: {
        immutable: true,
        required: false,
        // possibly other flags if defined
      },
      status: {
        enumSubset: ['in_process', 'completed'],
      },
      // ...
    }
  }
}
```

## Merging Rules
* if multiple conditions target the same field, DODIL merges them.
  * For example, if one rule says required: true and another says immutable: true, the field ends up both required and immutable.
* If rules contradict each other (e.g., one says required: true, another says required: false), the more restrictive rule generally takes precedence.


## Practical Usage in API Calls

Typically, you fetch or edit items through something like:
```JavaScript
// Fetch an ORDERITEM
const item = await dodilOrderItem.view(itemId);
// The 'item' response might include:
// {
//   _id: 'item123',
//   orderId: 'orderABC',
//   productName: 'Widget X',
//   quantity: 10,
//   status: 'in_process',
//   $state: {
//     fields: {
//       quantity: { immutable: true },
//       productName: { required: true },
//       // ...
//     }
//   }
// }
```
When you try to update the item (e.g., changing quantity when $state.fields.quantity.immutable = true), DODIL will throw an error if your update violates the immutable constraint.

## Conclusion
Use State Management to turn your application’s dynamic requirements into clear, declarative rules. Whether you’re locking a field after payment is processed, restricting status changes based on quantity, or making a child record dependent on its parent’s state—DODIL’s flexible condition and effect system has you cover