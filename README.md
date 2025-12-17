# RabbitMQ Complete Documentation Guide

## Overview
RabbitMQ is an open-source message broker that implements the Advanced Message Queuing Protocol (AMQP). Written in Erlang, it facilitates communication between distributed systems through asynchronous messaging patterns.

**Current Version:** 4.2.2 (Latest as of December 2024)

---

## Table of Contents
1. [Getting Started](#getting-started)
2. [Core Concepts](#core-concepts)
3. [AMQP 0-9-1 Model](#amqp-0-9-1-model)
4. [Exchanges](#exchanges)
5. [Queues](#queues)
6. [Bindings](#bindings)
7. [Messages](#messages)
8. [Consumers](#consumers)
9. [Publishers](#publishers)
10. [Client Libraries](#client-libraries)
11. [Configuration](#configuration)
12. [Clustering](#clustering)
13. [High Availability](#high-availability)
14. [Monitoring](#monitoring)
15. [Security](#security)
16. [Advanced Features](#advanced-features)
17. [Best Practices](#best-practices)

---

## Getting Started

### Installation

**Docker (Recommended for Development):**
```bash
# Latest RabbitMQ 4.x with management plugin
docker run -d --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  rabbitmq:4-management

# Access management UI at http://localhost:15672
# Default credentials: guest/guest
```

**Ubuntu/Debian:**
```bash
sudo apt-get update
sudo apt-get install rabbitmq-server
sudo systemctl enable rabbitmq-server
sudo systemctl start rabbitmq-server
```

**RHEL/CentOS/Fedora:**
```bash
# Add repository
curl -s https://packagecloud.io/install/repositories/rabbitmq/rabbitmq-server/script.rpm.sh | sudo bash

# Install
sudo yum install rabbitmq-server
sudo systemctl enable rabbitmq-server
sudo systemctl start rabbitmq-server
```

**macOS:**
```bash
brew install rabbitmq
brew services start rabbitmq
```

**Windows:**
```bash
# Using Chocolatey
choco install rabbitmq
```

### Enable Management Plugin
```bash
rabbitmq-plugins enable rabbitmq_management

# Access at http://localhost:15672
# Default: username=guest, password=guest
```

### Basic Commands
```bash
# Start server
rabbitmq-server

# Stop server
rabbitmqctl stop

# Server status
rabbitmqctl status

# List queues
rabbitmqctl list_queues

# List exchanges
rabbitmqctl list_exchanges

# List connections
rabbitmqctl list_connections

# List channels
rabbitmqctl list_channels
```

---

## Core Concepts

### Key Components

1. **Producer** - Application that sends messages
2. **Consumer** - Application that receives messages
3. **Queue** - Buffer that stores messages
4. **Exchange** - Routes messages to queues
5. **Binding** - Link between exchange and queue
6. **Routing Key** - Message attribute for routing
7. **Channel** - Virtual connection inside a connection
8. **Virtual Host (vhost)** - Isolated environment
9. **Connection** - TCP connection to RabbitMQ
10. **Message** - Data being transmitted

### Message Flow

```
Producer → Exchange → Binding → Queue → Consumer
            ↓
      [Routing Key]
```

1. Producer publishes message to an exchange
2. Exchange receives message and routing key
3. Exchange routes message to queue(s) based on bindings
4. Queue stores message until consumed
5. Consumer receives and processes message
6. Consumer sends acknowledgment

---

## AMQP 0-9-1 Model

AMQP (Advanced Message Queuing Protocol) is the primary protocol used by RabbitMQ.

### Key Concepts

**Programmable Protocol:**
- Applications declare their own topology (queues, exchanges, bindings)
- No pre-configured static routing
- Dynamic and flexible architecture

**Message Acknowledgments:**
- Manual or automatic acknowledgment
- Prevents message loss
- Ensures reliable delivery

**Virtual Hosts:**
- Logical grouping of resources
- Complete isolation between vhosts
- Multi-tenant support

### AMQP Model Flow

```
[Publisher] → [Exchange] → [Binding] → [Queue] → [Consumer]
                   ↓
           [Routing Rules]
```

**Key Properties:**
- Messages are published to exchanges (not directly to queues)
- Exchanges use routing rules to distribute messages
- Messages wait in queues until consumed
- Acknowledgments ensure reliability
- Networks can be unreliable; protocol handles this

---

## Exchanges

Exchanges receive messages from producers and route them to queues based on rules.

### Exchange Types

#### 1. Direct Exchange

Routes messages to queues based on exact routing key match.

**Use Cases:**
- Task distribution to specific workers
- Direct messaging
- Command routing

**Example:**
```
Routing Key: "pdf.create" → Queue: pdf_queue
Routing Key: "email.send" → Queue: email_queue
```

**Declaration:**
```bash
# Command line
rabbitmqadmin declare exchange name=orders type=direct durable=true

# AMQP method
exchange.declare(exchange="orders", type="direct", durable=True)
```

#### 2. Topic Exchange

Routes based on pattern matching with wildcards.

**Wildcards:**
- `*` (asterisk) - matches exactly one word
- `#` (hash) - matches zero or more words
- Words are separated by dots (.)

**Use Cases:**
- Log routing by severity and module
- News distribution by category
- Multi-criteria routing

**Examples:**
```
Pattern: "stock.*.nyse" matches:
  ✓ "stock.usd.nyse"
  ✓ "stock.eur.nyse"
  ✗ "stock.usd.nasdaq"

Pattern: "logs.#" matches:
  ✓ "logs.error"
  ✓ "logs.info.database"
  ✓ "logs.warn.api.auth"
```

**Declaration:**
```python
channel.exchange_declare(
    exchange='logs',
    exchange_type='topic',
    durable=True
)
```

#### 3. Fanout Exchange

Routes messages to ALL bound queues (broadcasts).

**Use Cases:**
- Broadcasting events
- Real-time notifications
- System-wide updates
- Game leaderboards
- Sport score updates

**Characteristics:**
- Ignores routing key completely
- Fastest exchange type
- One-to-many messaging

**Example:**
```
Message → Fanout Exchange → Queue A
                         → Queue B
                         → Queue C
```

**Declaration:**
```go
err := ch.ExchangeDeclare(
    "broadcasts",   // name
    "fanout",       // type
    true,           // durable
    false,          // auto-delete
    false,          // internal
    false,          // no-wait
    nil,            // arguments
)
```

#### 4. Headers Exchange

Routes based on message header attributes instead of routing key.

**Use Cases:**
- Complex routing logic
- Multiple criteria routing
- When routing key is insufficient

**Matching:**
- `x-match: all` - All headers must match
- `x-match: any` - At least one header must match

**Example:**
```python
# Binding with headers
channel.queue_bind(
    exchange='images',
    queue='resize_queue',
    arguments={
        'x-match': 'all',
        'format': 'jpg',
        'size': 'large'
    }
)

# Publishing with headers
properties = pika.BasicProperties(
    headers={'format': 'jpg', 'size': 'large'}
)
channel.basic_publish(
    exchange='images',
    routing_key='',
    body=message,
    properties=properties
)
```

### Default Exchange

Special nameless exchange (empty string "").

**Characteristics:**
- Pre-declared by broker
- Type: Direct
- Cannot be deleted
- Queues auto-bound with queue name as routing key
- Allows "direct" publishing to queues by name

**Example:**
```python
# Publishing to default exchange routes directly to queue name
channel.basic_publish(
    exchange='',           # Default exchange
    routing_key='myqueue', # Queue name
    body='Hello'
)
```

### Exchange Properties

**Durability:**
- `durable=true` - Survives broker restart
- `durable=false` - Deleted on restart

**Auto-Delete:**
- Deleted when last binding is removed
- Useful for temporary exchanges

**Internal:**
- Cannot be published to directly
- Only for exchange-to-exchange bindings

**Arguments:**
- `alternate-exchange` - Routes unroutable messages
- Custom extensions and plugins

### Exchange-to-Exchange Bindings (E2E)

Bind one exchange to another for complex routing.

**Example:**
```python
# Bind destination exchange to source exchange
channel.exchange_bind(
    destination='dest_exchange',
    source='source_exchange',
    routing_key='pattern.*'
)
```

**Use Cases:**
- Hierarchical routing
- Message filtering chains
- Complex topologies

---

## Queues

Queues store messages until they are consumed.

### Queue Types

#### 1. Classic Queues

Traditional queue implementation.

**Features:**
- FIFO ordering (mostly)
- Lazy mode for memory efficiency
- Message TTL
- Priority queues

**Declaration:**
```python
channel.queue_declare(
    queue='tasks',
    durable=True,
    exclusive=False,
    auto_delete=False,
    arguments={'x-queue-type': 'classic'}
)
```

#### 2. Quorum Queues

Replicated, highly available queues (recommended for production).

**Features:**
- Raft-based replication
- Data safety guarantees
- Automatic leader election
- Poison message handling

**Declaration:**
```python
channel.queue_declare(
    queue='orders',
    durable=True,
    arguments={
        'x-queue-type': 'quorum',
        'x-quorum-initial-group-size': 3
    }
)
```

**Use Cases:**
- Critical data
- High availability requirements
- Production workloads

#### 3. Streams

Immutable, append-only log (like Kafka).

**Features:**
- Non-destructive consumption
- Multiple consumers can read same messages
- Offset-based consumption
- Replay capability
- High throughput

**Declaration:**
```python
channel.queue_declare(
    queue='events',
    durable=True,
    arguments={
        'x-queue-type': 'stream',
        'x-max-length-bytes': 20000000000,  # 20GB
        'x-max-age': '7D'  # 7 days retention
    }
)
```

**Use Cases:**
- Event sourcing
- Audit logs
- Time-series data
- Large-scale fan-out

### Queue Properties

**Name:**
- Must be unique within vhost
- Empty string = server-generated name

**Durability:**
- `durable=true` - Survives restart (queue declaration persists)
- Messages need separate durability flag

**Exclusivity:**
- `exclusive=true` - Deleted when connection closes
- Used by single consumer only

**Auto-Delete:**
- Deleted when last consumer unsubscribes

**Arguments (x-arguments):**
```python
arguments = {
    'x-message-ttl': 60000,              # Message TTL (ms)
    'x-max-length': 1000,                # Max messages
    'x-max-length-bytes': 1048576,       # Max bytes (1MB)
    'x-dead-letter-exchange': 'dlx',     # DLX exchange
    'x-dead-letter-routing-key': 'dead', # DLX routing key
    'x-max-priority': 10,                # Priority levels
    'x-queue-mode': 'lazy',              # Lazy mode
    'x-single-active-consumer': True,    # Single consumer
}
```

### Server-Named Queues

Let RabbitMQ generate unique queue names.

```python
# Empty string = auto-generated name
result = channel.queue_declare(queue='', exclusive=True)
queue_name = result.method.queue  # Get generated name
```

**Use Cases:**
- Temporary queues
- RPC reply queues
- Consumer-specific queues

### Message Ordering

**FIFO Guarantee:**
- Messages from single channel to single queue are ordered
- Multiple consumers can break ordering on redelivery

**Preserving Order:**
1. Use streams (immutable log)
2. Enable Single Active Consumer
3. Use one consumer per queue
4. Use message priorities carefully

---

## Bindings

Bindings connect exchanges to queues with routing rules.

### Binding Keys

The pattern/key used to match routing keys.

**Direct Exchange:**
```python
channel.queue_bind(
    exchange='direct_logs',
    queue='errors',
    routing_key='error'  # Exact match
)
```

**Topic Exchange:**
```python
channel.queue_bind(
    exchange='logs',
    queue='critical_logs',
    routing_key='*.critical'  # Pattern match
)
```

**Fanout Exchange:**
```python
channel.queue_bind(
    exchange='broadcasts',
    queue='mobile_app',
    routing_key=''  # Ignored
)
```

**Headers Exchange:**
```python
channel.queue_bind(
    exchange='images',
    queue='process_queue',
    arguments={
        'x-match': 'all',
        'format': 'png',
        'size': 'large'
    }
)
```

### Multiple Bindings

A queue can have multiple bindings:

```python
# Queue receives from multiple routing keys
channel.queue_bind(exchange='logs', queue='all_logs', routing_key='info')
channel.queue_bind(exchange='logs', queue='all_logs', routing_key='warning')
channel.queue_bind(exchange='logs', queue='all_logs', routing_key='error')
```

### Unbinding

```python
channel.queue_unbind(
    queue='all_logs',
    exchange='logs',
    routing_key='info'
)
```

---

## Messages

### Message Properties

**Delivery Mode:**
- `1` - Non-persistent (faster)
- `2` - Persistent (survives restart)

**Priority:**
- 0-255 (queue must support priorities)

**Expiration:**
- Per-message TTL in milliseconds

**Headers:**
- Custom key-value metadata

**Content Type:**
- MIME type (e.g., "application/json")

**Content Encoding:**
- Encoding (e.g., "utf-8")

**Correlation ID:**
- For request-reply pattern

**Reply To:**
- Queue name for replies

**Message ID:**
- Unique identifier

**Timestamp:**
- Message creation time

**Type:**
- Message type identifier

**User ID:**
- Creating user (validated by broker)

**App ID:**
- Creating application

### Publishing Messages

**Python (pika):**
```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

# Declare exchange
channel.exchange_declare(
    exchange='logs',
    exchange_type='topic',
    durable=True
)

# Publish message
channel.basic_publish(
    exchange='logs',
    routing_key='info.user.login',
    body='User logged in',
    properties=pika.BasicProperties(
        delivery_mode=2,  # Persistent
        content_type='text/plain',
        headers={'source': 'web-app'},
        timestamp=int(time.time())
    )
)

connection.close()
```

**Go (amqp091-go):**
```go
conn, _ := amqp.Dial("amqp://guest:guest@localhost:5672/")
defer conn.Close()

ch, _ := conn.Channel()
defer ch.Close()

// Publish
err := ch.Publish(
    "logs",               // exchange
    "info.user.login",    // routing key
    false,                // mandatory
    false,                // immediate
    amqp.Publishing{
        ContentType:  "text/plain",
        Body:         []byte("User logged in"),
        DeliveryMode: amqp.Persistent,
    },
)
```

### Publisher Confirms

Ensure messages are received by broker.

**Python:**
```python
channel.confirm_delivery()

try:
    channel.basic_publish(
        exchange='orders',
        routing_key='new',
        body='order data',
        mandatory=True
    )
    print("Message delivered")
except pika.exceptions.UnroutableError:
    print("Message was returned")
```

**Go:**
```go
ch.Confirm(false)
confirms := ch.NotifyPublish(make(chan amqp.Confirmation, 1))

ch.Publish(/* ... */)

confirmed := <-confirms
if confirmed.Ack {
    log.Println("Message confirmed")
} else {
    log.Println("Message nacked")
}
```

### Message TTL

**Queue-Level TTL:**
```python
channel.queue_declare(
    queue='expiring_queue',
    arguments={'x-message-ttl': 60000}  # 60 seconds
)
```

**Per-Message TTL:**
```python
channel.basic_publish(
    exchange='',
    routing_key='queue',
    body='expires soon',
    properties=pika.BasicProperties(expiration='60000')
)
```

---

## Consumers

### Basic Consume

**Python:**
```python
def callback(ch, method, properties, body):
    print(f"Received: {body}")
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_consume(
    queue='tasks',
    on_message_callback=callback,
    auto_ack=False  # Manual acknowledgment
)

channel.start_consuming()
```

**Go:**
```go
msgs, err := ch.Consume(
    "tasks",  // queue
    "",       // consumer tag
    false,    // auto-ack
    false,    // exclusive
    false,    // no-local
    false,    // no-wait
    nil,      // arguments
)

for msg := range msgs {
    log.Printf("Received: %s", msg.Body)
    msg.Ack(false)
}
```

### Acknowledgment Modes

#### 1. Automatic Acknowledgment (Auto-Ack)

Message acknowledged immediately upon delivery.

```python
channel.basic_consume(
    queue='tasks',
    on_message_callback=callback,
    auto_ack=True  # Risky!
)
```

**Risk:** Message lost if consumer crashes.

#### 2. Manual Acknowledgment (Recommended)

Consumer explicitly acknowledges after processing.

```python
def callback(ch, method, properties, body):
    try:
        # Process message
        process(body)
        # Acknowledge success
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception:
        # Reject and requeue
        ch.basic_nack(
            delivery_tag=method.delivery_tag,
            requeue=True
        )
```

**Acknowledgment Methods:**
- `basic.ack` - Message successfully processed
- `basic.nack` - Reject one or more messages
- `basic.reject` - Reject single message

### Prefetch Count (QoS)

Limit unacknowledged messages per consumer.

```python
# Process max 10 messages at a time
channel.basic_qos(prefetch_count=10)
```

**Benefits:**
- Fair distribution among consumers
- Prevents consumer overload
- Better resource utilization

### Consumer Priority

Ensure high-priority consumers receive messages first.

```python
channel.basic_consume(
    queue='tasks',
    on_message_callback=callback,
    arguments={'x-priority': 10}
)
```

### Consumer Cancellation

**Graceful:**
```python
consumer_tag = channel.basic_consume(
    queue='tasks',
    on_message_callback=callback
)

# Cancel later
channel.basic_cancel(consumer_tag)
```

**Automatic:**
- Connection/channel closed
- Queue deleted
- Exclusive consumer disconnects

### Single Active Consumer

Only one consumer receives messages at a time.

```python
channel.queue_declare(
    queue='orders',
    arguments={'x-single-active-consumer': True}
)
```

**Use Cases:**
- Ordered processing
- Stateful consumers
- Exclusive access needed

---

## Publishers

### Connection Management

**Python:**
```python
import pika

# Connection parameters
credentials = pika.PlainCredentials('user', 'password')
parameters = pika.ConnectionParameters(
    host='localhost',
    port=5672,
    virtual_host='/',
    credentials=credentials,
    heartbeat=600,
    blocked_connection_timeout=300,
)

connection = pika.BlockingConnection(parameters)
channel = connection.channel()

# Use channel...

connection.close()
```

**Go:**
```go
config := amqp.Config{
    Heartbeat: 10 * time.Second,
    Locale:    "en_US",
}

conn, err := amqp.DialConfig("amqp://user:pass@localhost:5672/", config)
defer conn.Close()

ch, err := conn.Channel()
defer ch.Close()
```

### Publisher Best Practices

1. **Use Publisher Confirms**
```python
channel.confirm_delivery()
```

2. **Handle Returns** (unroutable messages)
```python
channel.add_on_return_callback(handle_return)
```

3. **Connection Pooling**
```go
// Maintain pool of connections
var connectionPool []*amqp.Connection
```

4. **Retry Logic**
```python
for attempt in range(max_retries):
    try:
        channel.basic_publish(...)
        break
    except Exception as e:
        if attempt == max_retries - 1:
            raise
        time.sleep(retry_delay)
```

5. **Use Persistent Messages for Important Data**
```python
properties=pika.BasicProperties(delivery_mode=2)
```

---

## Client Libraries

### Official Clients

#### Java
```xml
<dependency>
    <groupId>com.rabbitmq</groupId>
    <artifactId>amqp-client</artifactId>
    <version>5.20.0</version>
</dependency>
```

```java
ConnectionFactory factory = new ConnectionFactory();
factory.setHost("localhost");
Connection connection = factory.newConnection();
Channel channel = connection.createChannel();

channel.exchangeDeclare("logs", "topic", true);
channel.basicPublish("logs", "info.app", null, message.getBytes());

connection.close();
```

#### .NET/C#
```bash
dotnet add package RabbitMQ.Client
```

```csharp
var factory = new ConnectionFactory() { HostName = "localhost" };
using var connection = factory.CreateConnection();
using var channel = connection.CreateModel();

channel.ExchangeDeclare("logs", "topic", true);
channel.BasicPublish("logs", "info.app", null, body);
```

#### Python (pika)
```bash
pip install pika
```

```python
import pika

connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost')
)
channel = connection.channel()

channel.exchange_declare(exchange='logs', exchange_type='topic')
channel.basic_publish(exchange='logs', routing_key='info.app', body='Hello')

connection.close()
```

#### Node.js (amqplib)
```bash
npm install amqplib
```

```javascript
const amqp = require('amqplib');

const connection = await amqp.connect('amqp://localhost');
const channel = await connection.createChannel();

await channel.assertExchange('logs', 'topic', { durable: true });
channel.publish('logs', 'info.app', Buffer.from('Hello'));

await connection.close();
```

#### Go (amqp091-go)
```bash
go get github.com/rabbitmq/amqp091-go
```

```go
import amqp "github.com/rabbitmq/amqp091-go"

conn, _ := amqp.Dial("amqp://guest:guest@localhost:5672/")
defer conn.Close()

ch, _ := conn.Channel()
defer ch.Close()

ch.ExchangeDeclare("logs", "topic", true, false, false, false, nil)
ch.Publish("logs", "info.app", false, false, amqp.Publishing{
    Body: []byte("Hello"),
})
```

#### PHP (php-amqplib)
```bash
composer require php-amqplib/php-amqplib
```

```php
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();

$channel->exchange_declare('logs', 'topic', false, true, false);
$msg = new AMQPMessage('Hello');
$channel->basic_publish($msg, 'logs', 'info.app');

$channel->close();
$connection->close();
```

---

## Configuration

### Configuration File

**Location:** `/etc/rabbitmq/rabbitmq.conf`

**Format:** Sysctl (new) or Erlang term (legacy)

**Basic Configuration:**
```ini
# Network
listeners.tcp.default = 5672

# Management plugin
management.tcp.port = 15672
management.tcp.ip = 0.0.0.0

# Limits
vm_memory_high_watermark.relative = 0.4
disk_free_limit.absolute = 50GB

# Default user
default_vhost = /
default_user = admin
default_pass = secret
default_permissions.configure = .*
default_permissions.read = .*
default_permissions.write = .*

# Logging
log.file.level = info
log.console = true
log.console.level = info
```

### Environment Variables

```bash
# Node name
RABBITMQ_NODENAME=rabbit@hostname

# Data directory
RABBITMQ_MNESIA_BASE=/var/lib/rabbitmq/mnesia

# Log directory
RABBITMQ_LOG_BASE=/var/log/rabbitmq

# Config file
RABBITMQ_CONFIG_FILE=/etc/rabbitmq/rabbitmq.conf

# Enabled plugins
RABBITMQ_ENABLED_PLUGINS_FILE=/etc/rabbitmq/enabled_plugins
```

### Memory Management

```ini
# Memory limit (40% of available RAM)
vm_memory_high_watermark.relative = 0.4

# Absolute limit
vm_memory_high_watermark.absolute = 2GB

# Paging threshold
vm_memory_high_watermark_paging_ratio = 0.75
```

### Disk Space

```ini
# Free disk space limit
disk_free_limit.absolute = 50GB

# Percentage-based
disk_free_limit.relative = 2.0
```

---

## Clustering

### Why Cluster?

- **High Availability** - Redundancy
- **Scalability** - Distribute load
- **Throughput** - Parallel processing

### Cluster Setup

**Node 1:**
```bash
# Start RabbitMQ
rabbitmq-server -detached

# Get cookie (same on all nodes)
cat /var/lib/rabbitmq/.erlang.cookie
```

**Node 2:**
```bash
# Copy cookie from node 1
echo "SAME_COOKIE" > /var/lib/rabbitmq/.erlang.cookie
chmod 400 /var/lib/rabbitmq/.erlang.cookie

# Start node
rabbitmq-server -detached

# Join cluster
rabbitmqctl stop_app
rabbitmqctl join_cluster rabbit@node1
rabbitmqctl start_app
```

**Node 3:**
```bash
rabbitmqctl stop_app
rabbitmqctl join_cluster rabbit@node1
rabbitmqctl start_app
```

### Cluster Status

```bash
rabbitmqctl cluster_status
```

**Output:**
```
Cluster status of node rabbit@node1
[{nodes,[{disc,[rabbit@node1,rabbit@node2,rabbit@node3]}]},
 {running_nodes,[rabbit@node3,rabbit@node2,rabbit@node1]}]
```

### Node Types

**Disk Nodes:**
- Persist metadata to disk
- At least one disk node required
- Recommended for metadata safety

**RAM Nodes:**
- Store metadata in RAM only
- Better performance
- Must have disk node peer

**Changing Node Type:**
```bash
rabbitmqctl stop_app
rabbitmqctl change_cluster_node_type ram
rabbitmqctl start_app
```

### Leave Cluster

```bash
# From node leaving
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl start_app

# Or remove from another node
rabbitmqctl forget_cluster_node rabbit@node2
```

---

## High Availability

### Queue Mirroring (Classic Queues - Deprecated)

**Note:** Use Quorum Queues instead for HA.

**Policy:**
```bash
rabbitmqctl set_policy ha-all "^ha\." \
  '{"ha-mode":"all","ha-sync-mode":"automatic"}'
```

### Quorum Queues (Recommended)

Replicated using Raft consensus algorithm.

**Declaration:**
```python
channel.queue_declare(
    queue='orders',
    durable=True,
    arguments={
        'x-queue-type': 'quorum',
        'x-quorum-initial-group-size': 3
    }
)
```

**Features:**
- Automatic leader election
- Data replication across nodes
- Consistent in network partitions
- Poison message handling

**Requirements:**
- At least 3 nodes recommended
- Odd number of replicas
- Durable messages only

### Federation

Connect brokers across datacenters/regions.

**Use Cases:**
- Geo-distribution
- WAN links
- Cloud/on-prem hybrid

**Setup:**
```bash
# On downstream
rabbitmqctl set_parameter federation-upstream upstream1 \
  '{"uri":"amqp://server1","expires":3600000}'

# Create federation policy
rabbitmqctl set_policy federate-me "^federated\." \
  '{"federation-upstream-set":"all"}'
```

### Shovel

Move messages between brokers/queues.

**Use Cases:**
- Data migration
- Cross-datacenter sync
- Queue forwarding

**Dynamic Shovel:**
```bash
rabbitmqctl set_parameter shovel my-shovel \
'{
  "src-uri": "amqp://localhost",
  "src-queue": "source",
  "dest-uri": "amqp://remote-server",
  "dest-queue": "destination"
}'
```

---

## Monitoring

### Management UI

**Access:** http://localhost:15672

**Features:**
- Overview dashboard
- Queue/exchange stats
- Connection management
- User management
- Policy management
- Message publishing/consumption testing

### Command Line

```bash
# List queues with messages
rabbitmqctl list_queues name messages

# Connection details
rabbitmqctl list_connections name state channels

# Memory usage
rabbitmqctl status | grep memory

# Channel stats
rabbitmqctl list_channels connection number prefetch_count

# Consumer stats
rabbitmqctl list_consumers
```

### Metrics API

**HTTP API:**
```bash
# Overview
curl -u guest:guest http://localhost:15672/api/overview

# Node stats
curl -u guest:guest http://localhost:15672/api/nodes

# Queue stats
curl -u guest:guest http://localhost:15672/api/queues

# Connection stats
curl -u guest:guest http://localhost:15672/api/connections
```

### Prometheus Integration

**Enable Plugin:**
```bash
rabbitmq-plugins enable rabbitmq_prometheus
```

**Scrape Endpoint:**
```
