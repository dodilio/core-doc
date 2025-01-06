#  Flow
## Overview
The concept of data flow in this context refers to creating a step-by-step workflow or “flow” that orchestrates the collection/mutation of all necessary data for a higher-level entity.
this approach is useful in collecting complex data from LLMs or structured migrations
Let us take an example, while creating an order, you might need to collect:
1.	Basic order information (e.g. status, total amount).
2.	Multiple order items.
3.	A single delivery address.

Dodil’s Data Flow feature allows you to group these steps into a cohesive workflow:
-	Compositional: Each step references a separate schema (e.g., ORDER, ORDERITEM, DELIVERY_ADDRESS).
-	Sequential or Parallel: Steps can be strictly ordered (you must do step 1 before step 2) or configured for more flexible paths.
-	Partial Data Handling: At each step, partial data can be saved or validated before moving on, minimizing errors and rework.

This modular system can significantly reduce complexity in large-scale data collection scenarios.


## Concept
1.	Schema
      Defines a data model in Dodil, including validation rules (e.g., required fields, allowed enums).
2.	Flow Steps (flowStep)
      A single step referencing a schema and describing how data is collected (e.g., one-to-many relationship).
3.	Flow (dodil.flow)
      Combines multiple flow steps into a high-level data collection process for your entire entity (e.g., an Order).
4.	Compositor
      A runtime object that tracks which step the user/LLM is on, what data has been collected, and whether the flow is complete.


## Composing the Data Flow

@dodil allows you to group these schemas into a flow, meaning that the user (or LLM) must provide certain data in order before proceeding to subsequent steps.

For instance, your process to create an order might look like:
1.	Collect order details.
2.	Collect multiple order item entries.
3.	Collect a single delivery address.

Once all steps are finished, you finalize the order.

## Defining Schema
```JavaScript
const dodil = require('@dodil');
// 1. Define the Order schema
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

// 2. Define the OrderItem schema
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

// 4. Define the Order schema
const dodilDeliveryAddress = dodil.schema('DELIVERY_ADDRESS', {
  orderId: {
    type: 'id',
    refer: 'ORDER'
  },
  street: {
    type: 'string',
    required: true
  },
  country: {
    enum: ['eg', 'usa',...etc],
    required: true
  },
  mobile: {
    type: "string",
  },
});
```


## Example with LLMs

### lets define the flow
```JavaScript
const { schema, compose, flow } = require('@dodil');

// flow class decorator extends core functions to the class
// Example:
// .input() -> a polymorphic function to input data from the current stage 
// .isCompleted() -> indicate the flow is complete
// .currentlyMissing()/allMissing -> what are the missing data on this stage 

@flow.class()
class orderAcqusitionFlow {
      constructor() {
      // Composite Schema
         const orderComposite = compose(dodilOrder, {
            items: { from: dodilOrderItems, relation: 'many', minItems: 1 },
            deliverAddress: { from: dodilDeliveryAddress, relation: 'one', required: true },
            payment: { from : dodilPayment, relation: 'one', required: true }
         })
         this.orderCompose = new orderComposite();
      }
      
      async load(cacheId) {
         await this.orderCompose.loadFromChache(cacheId)
      }
     
      // a router middle where 
      // main concept is to help 
      @flow.router('route')
      async determineStage(input) {
         if(!this.orderCompose.isSatisfied('items'));
            this.routeTo(this.addItems);
         else if(!this.orderCompose.isSatisfied('deliveryAddress'))
            this.routeTo(this.addDeliveryAdress);
         else if (!this.orderCompose.isSatisfied('payment'))
            this.routeTo(this.addPayment);
      }
      
      @flow.input()
      async add(inputData) {
         return this.currentStage(inputData);
      }
      
      @flow.sequance.defaultStart()
      async addItems(data) {
        const i = dodilOrderItem(data);
        this.orderCompose.add('items', i);
        return this.orderCompose.length >= 1 && 
                  this.orderCompose.isSatisfied('items');
      }
      
      
      @flow.sequance.startOn(addItems)
      async addDeliveryAdress(data) {
        const address = dodilAddress(data);
        this.orderCompose.add('deliveryAddress', address);
        return this.orderCompose.isSatisfied('deliveryAddress');
      }
      
      @flow.sequance.startOn(addDeliveryAdress)
      addPayment(data) {
        const payment = dodilPayment(data);
        this.orderCompose.add('payment', payment);
        return this.orderCompose.isSatisfied('payment');
      }
      
      @flow.sequance.finishOn(addDeliveryAdress)
      async store() {
        await this.orderCompose.save()
      }
}
```

### lets consume the flow
```JavaScript
// assuming you created an LLM 
// Detect intent -> this person want to create order by adding cart..etc
function handleCartChange(chatID, inputData) {
  const orderFlow = orderAcqusitionFlow();
  await orderFlow.laod(chatID); // to load the composite cached data 
  
  // to determine the current need based on the compite data satisfaction
  await orderFlow.route(); 
  // to ensure we activated the flow after adjusting the pointer via the router
  await orderFlow.start(); 
  
  // Note: 
  // - uses the default flow.input acquired by the decorator
  // - validation: if the stage is items and the user added data for the delivery address it will fail
  await orderFlow.input(inputData); 
  
  // flow class have default 
  if (orderFlow.isCompleted()) {
     return {
       status: "complete",
       message: "Order creation is finished.",
       data: finalData,
     };
  }
  else {
      await orderFlow.orderCompose.cache(); // to temporary save the compose body
      const missingFields = compositor.currentlyMissing();  // use allMissing() to get all missing fields in all steps
      return {
            status: "in-progress",
            message: `We still need more information for step: ${currentStep.step}`,
            missingFields,
    };
  }
}

```

#### Example using flow to Migrate your Data

Another use case on how to use the compose and flow feature would be below used in migrating data

```JavaScript
const { schema, compose, flow, datasource } = require('@dodil');

// Assume existing schemas are already defined as per your initial setup
const mongoDB = await datasource('mongo', { ...config });
const sourceOrder = dodil.schema({
      "orderId": {
      "type": "string",
      "description": "Unique identifier for the order."
    },
    "status": {
      "type": "string",
      "enum": ["pending", "inprocess", "completed"],
      "description": "Current status of the order."
    },
    "totalAmount": {
      "type": "number",
      "default": 0,
      "description": "Total monetary value of the order."
    },
    ...etc
}, mongoDB);

@flow.class()
class DataMigrationFlow {
  constructor () {
      this.sourceId = null;
      this.batchSize = 1000;
      this.currentBatch = 0;
      
  }
  
  @flow.sequence.start()
  async initializeMigration(sourceId) {
    this.sourceId = sourceId;
    return true
  }
  
  @flow.sequence.startOn(initializeMigration) // when intialized migration
  @flow.sequence.startOn(migrateRecords) // when migrateRecords is completed
  async loadSourceData(sourceId) {
     /**
      {
        orderId: 'order123',
        status: 'pending',
        totalAmount: 150,
        items: [
          { productName: 'Widget A', quantity: 2, status: 'pending' },
          { productName: 'Widget B', quantity: 1, status: 'pending' },
        ],
        deliveryAddress: {
          street: '123 Main St',
          country: 'usa',
          mobile: '555-1234',
        },
        payment: {
          method: 'credit_card',
          transactionId: 'txn789',
        },
      },
     */
    const offset = this.batchSize *  this.currentBack
    this.batch++;
    const records = await this.sourceOrder.find().batch(this.batch).offset(offset);
    if(records.length > 0) {
      return records;
    }
    else 
      flow.stop(); // stop the flow
  }
  
  @flow.sequence.startOn(loadSourceData)
  async migrateRecords(records) {
    for (const record in records) {
     try {
      const orderCompose = new compose();
      // Validate and compose data using Dodil Compose
      const dodilOrder = dodilOrderSchema(record);
      const dodilOrderItems = record.items.map(item => dodilOrderItemSchema(item));
      const dodilDeliveryAddress = dodilDeliveryAddressSchema(record.deliveryAddress);
      const dodilPayment = dodilPaymentSchema(record.payment);

      orderCompose.add('order', dodilOrder);
      orderCompose.add('items', dodilOrderItems);
      orderCompose.add('deliveryAddress', dodilDeliveryAddress);
      orderCompose.add('payment', dodilPayment);

     if(orderCompose.isAllSatified()) {
       return await orderCompose.save();
     }
     else {
      // handle error 
     }
    }
  }
}


// use migration
const migration = new DataMigrationFlow();
await migration.start(); // to start the flow

```
