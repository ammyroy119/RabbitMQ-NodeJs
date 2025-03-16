# RabbitMQ-NodeJs
# RabbitMQ Guide

## 1. Introduction

### What is RabbitMQ?
RabbitMQ is a message broker: it accepts and forwards messages. You can think about it as a post office: when you put the mail that you want posting in a post box, you can be sure that the letter carrier will eventually deliver the mail to your recipient. In this analogy, RabbitMQ is a post box, a post office, and a letter carrier.

### Some General Terms in RabbitMQ:

![image](https://github.com/user-attachments/assets/c4def2f6-8380-4e62-b379-900e33f23d70)


- **Producer**: Producing means nothing more than sending. A program that sends messages is a producer.
- **Queue**: A queue acts as a memory palace on the host machine where the producer can add messages for the consumer.
- **Consumer**: A consumer is a program that waits to receive messages.

---

## 2. Installing RabbitMQ

### Option 1: Install RabbitMQ Locally (Ubuntu/Linux)
#### Install Erlang (RabbitMQ Dependency)
```sh
sudo apt update
sudo apt install -y erlang
```
#### Install RabbitMQ Server
```sh
sudo apt install -y rabbitmq-server
```
#### Enable and Start RabbitMQ
```sh
sudo systemctl enable rabbitmq-server
sudo systemctl start rabbitmq-server
```
#### Check RabbitMQ Status
```sh
sudo systemctl status rabbitmq-server
```

### Option 2: Install RabbitMQ Using Docker (Recommended)
Run RabbitMQ with the management plugin (UI for monitoring):
```sh
docker run -d --name rabbitmq \
  -p 5672:5672 -p 15672:15672 \
  rabbitmq:3-management
```
- **Port 5672** → RabbitMQ listens for messages here.
- **Port 15672** → Web UI for managing RabbitMQ.

#### Open RabbitMQ Dashboard:
Visit: [http://localhost:15672/](http://localhost:15672/)
- **Default Username**: guest
- **Default Password**: guest
---

![image](https://github.com/user-attachments/assets/12252669-01f2-443f-b550-060439bef01c)


## 3. Connecting Node.js with RabbitMQ

### Step 1: Install the Required Package
```sh
npm install amqplib
```

### Step 2: Create a Producer (Send Messages)
📌 Create `producer.js`
```javascript
const amqp = require("amqplib");
async function sendMessage() {
  const connection = await amqp.connect("amqp://localhost");
  const channel = await connection.createChannel();
  const queue = "hello";

  await channel.assertQueue(queue, { durable: false });

  const message = "Hello RabbitMQ!";
  channel.sendToQueue(queue, Buffer.from(message));

  console.log(`Sent: ${message}`);

  setTimeout(() => {
    connection.close();
  }, 500);
}

sendMessage().catch(console.error);
```

### Understanding `{ durable: false }`
- `durable: false` → The queue will be deleted if RabbitMQ restarts.
- `durable: true` → The queue persists even after RabbitMQ restarts.

#### Example:
```javascript
channel.assertQueue('myQueue', { durable: false });
```
If RabbitMQ restarts, `myQueue` will be lost.
To make a queue durable:
```javascript
channel.assertQueue('myQueue', { durable: true });
```
When publishing messages, also set:
```javascript
channel.sendToQueue('myQueue', Buffer.from('Hello!'), { persistent: true });
```

**When to use `durable: false`?**
- For temporary queues that don't need to survive restarts.
- When processing transient data that can be regenerated if lost.

---

### Step 3: Create a Consumer (Receive Messages)
📌 Create `consumer.js`
```javascript
const amqp = require("amqplib");
async function receiveMessage() {
  const connection = await amqp.connect("amqp://localhost");
  const channel = await connection.createChannel();
  const queue = "hello";

  await channel.assertQueue(queue, { durable: false });

  console.log(`Waiting for messages in ${queue}. To exit, press CTRL+C`);

  channel.consume(queue, (msg) => {
    console.log(`Received: ${msg.content.toString()}`);
  }, { noAck: true });
}

receiveMessage().catch(console.error);
```

---

## 4. Running the Producer & Consumer
### Start the Consumer (Receiver)
```sh
node consumer.js
```
You should see:
```
Waiting for messages in hello. To exit, press CTRL+C
```
### Send a Message with the Producer
```sh
node producer.js
```
The consumer should print:
```
Received: Hello RabbitMQ!
```
🎉 Congratulations! You have successfully sent and received a message using RabbitMQ in Node.js!

---

## 5. Acknowledging Messages (Manual & Auto)

### 🔹 Manual Acknowledgment (Recommended)
- Prevents message loss.
- Ensures message is removed only after processing.
- Uses `channel.ack(msg);`.

```javascript
channel.consume("test_queue", (msg) => {
  console.log(`✅ Processed: ${msg.content.toString()}`);
  channel.ack(msg); // Acknowledge manually
});
```

### 🔹 Automatic Acknowledgment
- The message is immediately removed after delivery.
- If the consumer crashes, the message is lost.

```javascript
channel.consume("test_queue", (msg) => {
  console.log(`✅ Auto-processed: ${msg.content.toString()}`);
}, { noAck: true }); // Auto ACK
```

✅ **Use Case**: Logging, analytics, real-time notifications.
⚠️ **Warning**: Messages are lost if the consumer crashes.

## 6. Work Queues
We will create a work queue to distribute time-consuming tasks among multiple workers (consumers).

#### Note: The main reason behind a work queue is to avoid executing a resource-intensive task immediately and waiting for it to complete. Instead, the task is scheduled to be processed later.

![image](https://github.com/user-attachments/assets/1f2c793d-107d-4ea3-88ad-b0c277467297)


### Producer: Sending 100 Messages
This producer sends 100 messages to the queue. Since we don’t have a real task (like image processing, PDF generation, or network calls), we simulate processing with setTimeout. The processing time varies based on the message order.

📌 producer.js

```javascript
const amqp = require("amqplib");
async function sendMessage() {
  const connection = await amqp.connect("amqp://localhost");
  const channel = await connection.createChannel();
  const queue = "hello";

  await channel.assertQueue(queue, { durable: true });
  const message = "Hello RabbitMQ!";

  for (let i = 1; i <= 100; i++) {
    channel.sendToQueue(queue, Buffer.from(`${i}-${message}`), { persistent: true });
  }
  console.log(`Sent: all messages`);

  setTimeout(() => {
    connection.close();
  }, 500);
}

sendMessage().catch(console.error);
```
### Consumer: Processing Messages
The consumer processes messages from the queue. It uses setTimeout to simulate varying processing times(because we don't have the real task like- image processing, network call etc) based on whether the message number is odd or even. Odd-numbered messages take longer to process than even-numbered ones.

📌 consumer.js
```javascript
const amqp = require("amqplib");
async function receiveMessage() {
  const connection = await amqp.connect("amqp://localhost");
  const channel = await connection.createChannel();
  const queue = "hello";

  await channel.assertQueue(queue, { durable: true });

  console.log(`Waiting for messages in ${queue}. To exit, press CTRL+C`);

  channel.consume(queue, (msg) => {
    const nthMessage = msg.content.toString().split('-')[0];
    console.log(nthMessage);

    let timeout = 2000;
    if (Number(nthMessage) % 2 !== 0) {
      timeout = 4000;
    }

    setTimeout(() => {
      console.log('message processed');
      channel.ack(msg);
    }, timeout);
  });
}

receiveMessage().catch(console.error);
```
#### How to Execute the Work Queue
Run one or more consumers (workers): for more workers open more terminal and run the same below command.
shell-1/Terminal-1
```sh
node consumer.js
```

shell-2/Terminal-2
```sh
node consumer.js
```
Then, run the producer:
```sh
node producer.js
```
When multiple workers are running, messages will be distributed in a round-robin manner. For example, if you run two workers, one might process messages like 1st, 3rd, 5th, etc., while the other handles 2nd, 4th, 6th, etc. This setup allows heavy tasks to be processed in parallel.

### Fair Dispatch
By default, RabbitMQ dispatches messages as they enter the queue without considering the load on each consumer. This means that if odd-numbered messages are heavy (for example, if you increase the timeout to simulate a long task), one worker might get overwhelmed while another remains idle.

![image](https://github.com/user-attachments/assets/c8a18a9c-1cc4-4118-8b4a-bf5d5745ebc0)


To avoid this, use the prefetch method:
```javascript
channel.prefetch(1);
```
This tells RabbitMQ to send only one message to a worker at a time until it has finished processing and acknowledged the previous message.

Note about queue size: If all workers are busy, the queue can fill up. Monitor the queue and add more workers or use another strategy if needed.



---

📌 _This README will be updated with the next pending RabbitMQ topics._

