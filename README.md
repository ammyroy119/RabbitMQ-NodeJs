# RabbitMQ-NodeJs
# RabbitMQ Guide

## 1. Introduction

### What is RabbitMQ?
RabbitMQ is a message broker: it accepts and forwards messages. You can think about it as a post office: when you put the mail that you want posting in a post box, you can be sure that the letter carrier will eventually deliver the mail to your recipient. In this analogy, RabbitMQ is a post box, a post office, and a letter carrier.

### Some General Terms in RabbitMQ:
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

---

📌 _This README will be updated with the next pending RabbitMQ topics._

