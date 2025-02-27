# cluster-messages
A helpful Node module to make it easy to send messages between the
master and workers with `callbacks` or `async/await`

Events can be emitted by any process and received by any process.

# Usage

Require the package:
```javascript
let ClusterMessages = require('cluster-messages');
let messages = new ClusterMessages();
```

## Async/Await

Setting up event listeners :
```javascript
messages.on('multiplication', (data, sendResponse) => {
  sendResponse(data.x * data.y);
});
```

Emitting events:
```javascript
let data = {
  x: Math.round(Math.random() * 100),
  y: Math.round(Math.random() * 100),
};

const response = await messages.sendAwait('multiplication', data);

console.log(`${data.x} * ${data.y} = ${response}`)
```

## Callback

Setting up event listeners :
```javascript
messages.on('multiplication', (data, sendResponse) => {
  sendResponse(data.x * data.y);
});
```

Emitting events:
```javascript
let data = {
  x: Math.round(Math.random() * 100),
  y: Math.round(Math.random() * 100),
};

messages.send('multiplication', data, response => {
  console.log(`${data.x} * ${data.y} = ${response}`)
})
```

---

## .send(eventName, data, callback)

- **eventName** (string) - the name of the event being emitted
- **data** (object) - the object passed to the event listener
- **callback** (function) - callback invoked
by the event listener, single parameter containing the response

```javascript
messages.send('print', { name: 'John' }, (response) => {
  console.log(response); // 'Hi John'
});
```

This function will emit an event, the callback is a single parameter
that is passed, this means inside the event listener only one parameter
can be passed to `sendResponse` so it needs to be an object containing
all the data.

When sent from the master:
- the event will get sent to all workers
- each time the worker calls `sendResponse` (the callback inside
the event listener), the `callback` will be invoked
- e.g. if there are 3 workers, and all of them call `sendResponse`,
the callback will be invoked 3 times

When sent from the worker:
- the event is sent to the master only

## .sendAwait(eventName, data)

- **eventName** (string) - the name of the event being emitted
- **data** (object) - the object passed to the event listener

```javascript
const response = await messages.send('print', { name: 'John' });

console.log(response); // 'Hi John'
```

This function will emit an event. Inside the event listener only one parameter
can be passed to `sendResponse` so it needs to be an object containing
all the data.

When sent from the master:
- the event will get sent to all workers
- the first worker which calls `sendResponse` will resolve the promise

When sent from the worker:
- the event is sent to the master only

## .on(eventName, callback)

- **eventName** (string) - the name of the event to listen for
- **data** (object) - single object sent by the event emitter (`.send`)
- **callback** (function) - takes two parameters, `data` and
`sendResponse` (function). `data` is the data sent by the event emitter,
`sendResponse` is a function that you invoke and pass the data you want
to send back to the callback on the event emitter

```javascript
messages.on('print', (data, sendResponse) => {
  sendResponse('Hi ' + data.name);
});
```

This is the event listener, the callback takes two parameters (`data`
and `sendResponse`). `sendResponse` only takes one parameter and is
sent back to wherever the event originated.

---

When initiating ClusterMessages you can pass it some options:
```javascript
const ClusterMessages = require('cluster-messages');

const options = {
  metaKey: '__metadata__',
  callbackTimeout: 1000 * 60 * 10,
};

const messages = new ClusterMessages(options);
```

#### metaKey
Messages are sent around as usual and can be
received through `cluster.on('message')` or
`process.on('message')` as expected, so the module adds a property containing
meta data that is passed around with each event. As this meta data
property is essentially exposed in every single `message` event emission
within `cluster` or `process`, the `metaKey` option allows you to define
the name of the meta data property to ensure it does not conflict with
your application.

#### responseTimeout
When a callback/promise is passed to `messages.send`, it is stored, but in the
case that for some reason due to either Node or the OS that the message
is dropped when being sent from one process to the other, the callback/promise
will never get deleted so the callback/promise is automatically deleted after
a set amount of time (default is 10 minutes). This option
allows you to specify a timeout in milliseconds. This option is not
really necessary at all, but the point of this logic is to prevent
memory leaks.
