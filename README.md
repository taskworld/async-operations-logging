# Logging of asynchronous operations in JavaScript.

Ok, let’s assume that we are building an application loads some data from the database (asynchronously) and generates a report like this:

```js
// version0.js
import Promise from 'bluebird'

// The use-case function that does the business logic.
async function processRequest () {
  const [ a, b ] = await Promise.all([ loadDataA(), loadDataB() ])
  return await combineData(a, b)
}

// This function works industriously to obtain the number 1 from the database.
async function loadDataA () {
  await Promise.delay(300)
  return 1
}

// This function works sedulously to obtain the number 2 from the cloud.
async function loadDataB () {
  await Promise.delay(600)
  return 2
}

// This function works diligently to obtain the sum of two numbers.
async function combineData (a, b) {
  await Promise.delay(1000)
  return a + b
}

// Main code that receives request, delegates work to service functions,
// and output the result back to the requester. For instance, this may be
// an API endpoint or whatev’r, but here we’re simply implementing it
// as a console program.
function main () {
  Promise.resolve(processRequest())
  .then(answer => console.log('Answer is %d!', answer))
  .done()
}

main()
```

Let’s run this code and wait a while, and it answers:

```
Answer is 3!
```

Then you wonder, why it took so long to just compute a simple sum of two numbers. To know what to do, it’s best to profile your application to see which part of our code took the most time. Then we know where to look next.

So we decided that we need to perform logging — at the start and the end of each asynchronous operation.

Since this service will be used by thousand of clients simultaneously, we need some kind of “transaction ID” (generated per request) to distinguish different requests from each other.


## Attempt 1: Pass transaction ID to each async operation.

This is the most straightforward solution.
The request handler generates a new transaction ID and passes it to the service,
which passes it on to every function that requires logging.

```js
// version1.js
import Promise from 'bluebird'
import invariant from 'invariant'
import util from 'util'
import { uniqueId } from 'lodash'

// A function to help ensure that a transaction ID is passed.
// So that we don’t log out `transactionId=undefined`.
function ensureTransactionIdSent (transactionId) {
  invariant(transactionId, 'You forgot to pass in a transactionId.')
}

// A function that logs the operation.
function log (...args) {
  console.log(`[${Date.now()}] ${util.format(...args)}`)
}

async function processRequest (transactionId) {
  ensureTransactionIdSent(transactionId)
  try {
    log('Begin processRequest, transactionId=%s', transactionId)
    const [ a, b ] = await Promise.all([
      loadDataA(transactionId),
      loadDataB(transactionId)
    ])
    return await combineData(a, b, transactionId)
  } finally {
    log('Finish processRequest, transactionId=%s', transactionId)
  }
}

async function loadDataA (transactionId) {
  ensureTransactionIdSent(transactionId)
  try {
    log('Begin loadDataA, transactionId=%s', transactionId)
    await Promise.delay(300)
    return 1
  } finally {
    log('Finish loadDataA, transactionId=%s', transactionId)
  }
}

async function loadDataB (transactionId) {
  ensureTransactionIdSent(transactionId)
  try {
    log('Begin loadDataB, transactionId=%s', transactionId)
    await Promise.delay(600)
    return 2
  } finally {
    log('Finish loadDataB, transactionId=%s', transactionId)
  }
}

async function combineData (a, b, transactionId) {
  ensureTransactionIdSent(transactionId)
  try {
    log('Begin combineData, transactionId=%s', transactionId)
    await Promise.delay(1000)
    return a + b
  } finally {
    log('Finish combineData, transactionId=%s', transactionId)
  }
}

function main () {
  const transactionId = uniqueId('transaction')
  Promise.resolve(processRequest(transactionId))
  .then(answer => console.log('Answer is %d!', answer))
  .done()
}

main()
```

Ok, let’s run this code to verify that it works.

```
[1461915275057] Begin processRequest, transactionId=transaction1
[1461915275058] Begin loadDataA, transactionId=transaction1
[1461915275060] Begin loadDataB, transactionId=transaction1
[1461915275362] Finish loadDataA, transactionId=transaction1
[1461915275665] Finish loadDataB, transactionId=transaction1
[1461915275667] Begin combineData, transactionId=transaction1
[1461915276670] Finish combineData, transactionId=transaction1
[1461915276670] Finish processRequest, transactionId=transaction1
Answer is 3!
```

Ok, it works.

At least, we now know which process took the most time (combineData takes 1 second!). We now have data which tells you where to improve the performance.

__But go back and look at the code. Is it easy to read like the first version?__ All I see is transactionId `transactionId` _transactionId_ __transactionId__ everywhere!

The main logic of this function is buried deep inside the logging ceremony… Although it works, this is not good at all.

__There must be a better way.__


## Attempt 2: Create a Transaction object

Seeing a lot of repetitive code above, let’s try again, this time we’ll create a `Transaction` object that has the ability to run an asynchronous function and perform logging.

```js
// Transaction.js
import util from 'util'
import { uniqueId } from 'lodash'

export class Transaction {
  constructor () {
    this._id = uniqueId('transaction')
  }
  async run (taskName, work) {
    try {
      log('Begin %s, transactionId=%s', taskName, this._id)
      return await work()
    } finally {
      log('Finish %s, transactionId=%s', taskName, this._id)
    }
  }
}

function log (...args) {
  console.log(`[${Date.now()}] ${util.format(...args)}`)
}

export default Transaction
```

Now our code can be like this:

```js
// version2.js
import Promise from 'bluebird'
import Transaction from './Transaction'

async function processRequest (transaction) {
  return transaction.run('processRequest', async () => {
    const [ a, b ] = await Promise.all([
      loadDataA(transaction),
      loadDataB(transaction)
    ])
    return await combineData(a, b, transaction)
  })
}

async function loadDataA (transaction) {
  return transaction.run('loadDataA', async () => {
    await Promise.delay(300)
    return 1
  })
}

async function loadDataB (transaction) {
  return transaction.run('loadDataB', async () => {
    await Promise.delay(600)
    return 2
  })
}

async function combineData (a, b, transaction) {
  return transaction.run('combineData', async () => {
    await Promise.delay(1000)
    return a + b
  })
}

function main () {
  const transaction = new Transaction()
  Promise.resolve(processRequest(transaction))
  .then(answer => console.log('Answer is %d!', answer))
  .done()
}

main()
```

Note that I removed the `ensureTransactionIdSent` function,
because the error message `“Fatal TypeError: Cannot read property 'run' of undefined”`, is good enough to tell us that we forgot to send in a transaction. At least we cannot accidentally print out “transactionId=undefined”.

Let’s run it:

```js
[1461915965332] Begin processRequest, transactionId=transaction1
[1461915965333] Begin loadDataA, transactionId=transaction1
[1461915965335] Begin loadDataB, transactionId=transaction1
[1461915965640] Finish loadDataA, transactionId=transaction1
[1461915965939] Finish loadDataB, transactionId=transaction1
[1461915965940] Begin combineData, transactionId=transaction1
[1461915966946] Finish combineData, transactionId=transaction1
[1461915966946] Finish processRequest, transactionId=transaction1
Answer is 3!
```

Now this is indeed _much_ better.

But still, I have to pass the transaction object __everywhere__. There are 5 mentions of `transaction` in the `processRequest`.

Is this good? Is this necessary? Why do I need to do that? Well, so that the child task can also be run against the same transaction — that’s why we need to know which transaction it is: so that we can tell the child task to run itself on the same transaction — really?

Is it my responsibility to _know_ and _pass the transaction forward_ to everyone else? Can’t that be done transparently so I can focus more on my core logic?

__There must be a better way.__


## Solution: Introduce a concept of Task.

From the reflection above, we conclude that a Task is an asynchronous process that:

- Can be run against a transaction.
- Can spawn sub-tasks, which will be run in the same transaction.

Therefore, we can implement a Task and use it like this:

```js
// Task.js
export class Task {
  constructor (name, work) {
    this._name = name
    this._work = work
  }

  // The only way to execute a task is to run it inside some transaction.
  // The work function is given a function to run a subtask.
  // It will return a promise that resolves to the task’s result.
  runInTransaction (transaction) {
    const runSubtask = subtask => subtask.runInTransaction(transaction)
    return transaction.run(this._name, () => {
      return this._work(runSubtask)
    })
  }
}

export default Task
```

Now our code can be a function that returns a Task instead of being an async function.

```js
// version3.js
import Promise from 'bluebird'
import Transaction from './Transaction'
import Task from './Task'

function processRequest () {
  return new Task('processRequest', async (run) => {
    const [ a, b ] = await Promise.all([ run(loadDataA()), run(loadDataB()) ])
    return await run(combineData(a, b))
  })
}

function loadDataA () {
  return new Task('loadDataA', async () => {
    await Promise.delay(300)
    return 1
  })
}

function loadDataB () {
  return new Task('loadDataB', async () => {
    await Promise.delay(600)
    return 2
  })
}

function combineData (a, b) {
  return new Task('combineData', async () => {
    await Promise.delay(1000)
    return a + b
  })
}

function main () {
  const task = processRequest()
  Promise.resolve(runTask(task))
  .then(answer => console.log('Answer is %d!', answer))
  .done()
}

// Utility: Runs a task in a new transaction.
function runTask (task) {
  const transaction = new Transaction()
  return task.runInTransaction(transaction)
}

main()
```

__Now, your code is no longer aware of the underlying transaction.__
And you can now focus on your core logic.
This is the power of abstraction.


## Tracking child tasks

Also, you can modify `Task.js` a little bit and get parent-child task tracking for free:

```js
  runInTransaction (transaction, prefix='') {
    const name = prefix + this._name
    const runSubtask = subtask => subtask.runInTransaction(transaction, `${name} → `)
    return transaction.run(name, () => {
      return this._work(runSubtask)
    })
  }
```

```
[1461919069214] Begin processRequest, transactionId=transaction1
[1461919069216] Begin processRequest → loadDataA, transactionId=transaction1
[1461919069219] Begin processRequest → loadDataB, transactionId=transaction1
[1461919069526] Finish processRequest → loadDataA, transactionId=transaction1
[1461919069824] Finish processRequest → loadDataB, transactionId=transaction1
[1461919069824] Begin processRequest → combineData, transactionId=transaction1
[1461919070825] Finish processRequest → combineData, transactionId=transaction1
[1461919070825] Finish processRequest, transactionId=transaction1
Answer is 3!
```
