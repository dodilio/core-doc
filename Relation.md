# Relation
In many data models, especially those involving document stores or relational databases, “parent-child” relationships (or “one-to-many” relationships) are foundational. For instance:
- An Order (parent) can have multiple Order Items (children)
- A Category (parent) can have multiple Subcategories (children)
- A User (parent) can have multiple Addresses (children)

DODIL provides a concise way to define these relationships by referencing one schema from within another using the { type: 'id', refer: 'SCHEMA_NAME' } pattern.

## Basic One-to-Many Example
### Order → OrderItem
Below is a simple example where each OrderItem references a single Order. This pattern supports a one-to-many relationship: one Order can have multiple OrderItems.
```javascript
    // 1. Import DODIL
const dodil = require('@dodil');

// 2. Initialize the datastore connector (Mongo, SQL, etc.)
const dodilStore = dodil.dataStore('mongo-db', 'localhost:27017');

// 3. Define the Parent Schema
const dodilOrder = dodil.schema('ORDER', {
  status: {
    type: 'string',
    enum: ['pending', 'inprocess', 'completed'],
  },
  totalAmount: {
    type: 'number',
  },
});

// 4. Define the Child Schema referencing the Parent
const dodilOrderItem = dodil.schema('ORDERITEM', {
  orderId: { 
    type: 'id', 
    refer: 'ORDER' // References the ORDER schema
  },
  productName: {
    type: 'string',
  },
  quantity: {
    type: 'number',
  },
  status: {
    type: 'string',
    enum: ['pending', 'inprocess', 'completed'],
  },
})
```

	•	Parent: ORDER
	•	Child: ORDERITEM
	•	Relationship: The orderId field in ORDERITEM references the parent’s schema, making it easy to see which items belong to which order.




## Multiple Children Example

Sometimes an Order can have multiple child schemas—let’s say OrderItems and Payments. Each child references the same parent in different ways.

```javascript
const dodilPayment = dodil.schema('PAYMENT', {
  orderId: {
    type: 'id',
    refer: 'ORDER',
  },
  amount: {
    type: 'number',
  },
  method: {
    type: 'string',
    enum: ['credit_card', 'paypal', 'wire_transfer'],
  },
});
```

    In this scenario:
    - Parent: ORDER
    - Children: ORDERITEM and PAYMENT

Each child collection references the same parent by storing orderId. This pattern keeps the parent schema simpler while allowing specialized child schemas.

## Nested Parent-Child-Grandchild Example

Complex applications can have deeper hierarchical data. For instance, an Order has many OrderItems, and each OrderItem might have many SerialNumbers (e.g., if the item is a product that needs a unique identifier).

```JavaScript
// Grandchild Schema: ORDERITEM_SERIAL
const dodilOrderItemSerial = dodil.schema('ORDERITEM_SERIAL', {
  orderItemId: { 
    type: 'id', 
    refer: 'ORDERITEM' // references the child
  },
  serialNumber: {
    type: 'string',
  },
});
```
Now the chain looks like:
- Parent: ORDER
- Child: ORDERITEM (references ORDER)
- Grandchild: ORDERITEM_SERIAL (references ORDERITEM)

This nested approach is particularly common when each level in the hierarchy adds distinct properties or constraints.

## One-to-One Relationship Example
Although less common, you might also define a one-to-one relationship. For example, an Order might have a ShippingDetail that contains a single shipping address, method, and tracking info.

```JavaScript
  const dodilShippingDetail = dodil.schema('SHIPPING_DETAIL', {
  orderId: { 
    type: 'id',
    refer: 'ORDER',
    unique: true // Ensures each orderId can only appear once in SHIPPING_DETAIL
  },
  address: { 
    type: 'string',
  },
  trackingNumber: {
    type: 'string',
  },
});

// usage
const shipping = await dodilShippingDetail.create({
  orderId: 'order123',
  address: '123 Elm Street',
  trackingNumber: 'TRACK-001',
});
```
By using unique: true, you effectively establish a one-to-one constraint between ORDER and SHIPPING_DETAIL at the database level (assuming the underlying datastore supports unique indexes).

## Many-to-Many Relationship Example
To handle a many-to-many scenario, you typically introduce a junction (or link) schema. For instance, consider an Article that can have many Tags, and a Tag that can belong to many Articles.

```JavaScript
    const dodilArticle = dodil.schema('ARTICLE', {
  title: { type: 'string' },
  content: { type: 'string' },
});

const dodilTag = dodil.schema('TAG', {
  name: { type: 'string' },
});

// Junction Schema
const dodilArticleTag = dodil.schema('ARTICLE_TAG', {
  articleId: { 
    type: 'id', 
    refer: 'ARTICLE' 
  },
  tagId: { 
    type: 'id', 
    refer: 'TAG' 
  },
});
```

To link them, you insert rows/documents in ARTICLE_TAG, each entry referencing one articleId and one tagId. This is a classic relational approach that DODIL handles seamlessly.