# kafkjajs-buffer

Plugin for [kafkajs](https://github.com/tulios/kafkajs) to buffer messages and send them in batches, inspired by [node-rdkafka](https://github.com/Blizzard/node-rdkafka)

# Overview

kafkajs-buffer adds queue/buffer capabilities to a kafkajs producer. It allows buffering output messages. It splits the buffer into batches and sends them to Kafka, optimizing the number of requests. Delivered messages will be notified in a callback function, avoiding the need to await the requests and improving streaming times.

# Why to use it
When we publish messages to Kafka, it's crucial to control the size and the moment of the requests we send. This library solves two common problems:
Grouping in batches is essential. We can't send messages one by one, it would produce overload if we send them simultaneously, and it will be prolonged if we send them one by one, waiting for Kafka knowledge (roundtrip).

- Split into optimal batches the number of messages to send to Kafka. Kafka doesn't accept messages heavier than a prefix size, depending on the configuration. Kafkajs-buffer allows setting the size of the batches to fit the Kafka configured size.

- Batching in blocks of the proper size and sending them to Kafka will lead to specific logic in our code to postpone the sending of messages, group them and send later based on a reached max messages buffer length or a given time. Kafkajs-buffer solves this problem transparently.

# Usage

You can install the kafkajs-buffer module like any other module:

```
npm install kafkajs-buffer
```

To use the module, you must require and instance it.

```typescript
import { KafkajsBuffer } from "kafkajs-buffer";
```

```typescript
const producerBuffer = new KafkajsBuffer(producer, options);
```

To send the messages push them in the buffer, similar the way you would send it using Kafkajs.

```typescript
  producerBuffer.push({
    "topic-1",
    messages: [
      {
        key: "m1",
        value: "message 1",
      },
      {
        key: "m2",
        value: "message 2",
      },
    ],
  });
```

You can also push messages for different topics.

```typescript
producerBuffer.push([
  {
    topic: "topic-1",
    messages: [
      {
        key: "m1",
        value: "message 1",
      },
    ],
  },
  {
    topic: "topic-2",
    messages: [
      {
        key: "m2",
        value: "message 2",
      },
      {
        key: "m3",
        value: "message 3",
      },
    ],
  },
]);
```

You can programatically request to send the buffer messages to kafka. This avoids reaching the max buffer size. Depending on the time from the last sending, the messages in the buffer queue will be sent immediately or postponed.

```typescript
producerBuffer.poll();
```

In addition you can set the producer to poll automatically on an interval.

```typescript
producerBuffer.startAutoPolling(100);
```

Don't forget stop the autopolling before your program execution ends.

```typescript
producerBuffer.stopAutoPolling();
```

To receive the confirmation when the messages are published to kafka use the callback functions 'onBatchDelivered' and/or 'onMessageDelivered'.

```typescript
// This function is called everytime a message is successfully sent to Kafka
const onMessageDelivered = (messageDelivered) => {
  messagesDeliveredCount += 1;
};
```

```typescript
// This function is called everytime a batch is successfully sent to Kafka
const onBatchDelivered = (messagesDelivered: IDeliveredMessage[]) => {
  messagesDeliveredCount += messagesDelivered.length;
};
```

In addition you can add extra information to the messages that won't be sent to kafka but will be received in the callback function.

```typescript
type Info = {
  timestamp: number;
};

const producerBuffer = new KafkajsBuffer<Info>(producer, options);

const messageToSend: IMessageWithInfo<Info> = {
  key: "1",
  value: "message value",
  info: {
    timestamp: Date.now(),
  },
};

const onMessageDelivered = (messageDelivered: IDeliveredMessage<Info>) => {
  console.log(
    `Message created at ${messageDelivered.info?.timestamp} was delivered to kafka`
  );
};
```

To greacefully shutdonwn your process you must call and await 'flush'. It will wait until a possible onprogress sending ends and/or will send any pending message still in the buffer.

```typescript
await producerBuffer.flush();
```

# Configuration

```typescript
const options = {
  batchNumMessages: 1000, // The buffer is sent to kafka splitted in batches of this size.
  queueBufferingMaxMs: 1000, // Time the messages are buffered before sending. Polling actions will be trigger the sending after this time.
  queueBufferingMaxMessages: 100000, // Max number of messages allowed in the buffer. When more messages are pushed it will throw an error.
  onMessageDelivered: () => {}, // Callback confirmation when a message is delivered to kafka.
  onBatchDeliverd: () => {}, // Callback confirmation when a batch is delivered to kafka.
  onSendError: (err) => {}, // Callback with error when the messages are tried to be sent after a poll and fail
  messageAcks: -1, // Control the number of required acks (https://kafka.js.org/docs/producing)
  responseTimeout: 30000, // The time to await a response in ms (https://kafka.js.org/docs/producing)
  messageCompression: CompressionTypes.None, // Compression codec (https://kafka.js.org/docs/producing)
};
```
