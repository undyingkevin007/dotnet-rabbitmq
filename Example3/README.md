# Publish / Subscribe

## Contents

- [Overview](#overview)
- [Characteristics](#characteristics)
- [Example Solution](#example-solution)
- [Running the Example](#running-the-example)

---

## Overview

![rmq-pubsub](https://user-images.githubusercontent.com/33935506/98722034-e02df500-23f5-11eb-88f4-982b2b3621ad.png)

---

## Characteristics

- TODO

---

## Example Solution

There are 2 parts to the solution. A _Producer_ and a _Consumer_. The _Producer_ is a .Net Core Console Application that sends _forecasts_ to a queue at a specific interval. The _Consumer_ is a .NET Core Console application that waits and consumes messages as messages arrive on the queue.

The following 2 sections, _Producer_ and _Consumer_, highlight the code required to interact with _RabbitMQ_

### Consumer

#### Step 1 - Create Connection

```csharp
var connectionFactory = new ConnectionFactory
{
    HostName = "localhost",
    UserName = "admin",
    Password = "password"
};

using var connection = connectionFactory.CreateConnection();
```

#### Step 2 - Create Channel

```csharp
using var channel = connection.CreateModel();
```

#### Step 3 - Declare Queue

```csharp
var queueName = channel.QueueDeclare().QueueName;
```

#### Step 4 - Declare Exchange

```csharp
const string ExchangeName = "example3_forecasts_exchange";

channel.ExchangeDeclare(exchange: ExchangeName, type: ExchangeType.Fanout);
```

#### Step 5 - Create Binding

```csharp
channel.QueueBind(
    queue: queueName,
    exchange: ExchangeName,
    routingKey: string.Empty);
```

#### Step 6 - Create Consumer

```csharp
var consumer = new EventingBasicConsumer(channel);

consumer.Received += (sender, eventArgs) =>
{
    var body = eventArgs.Body.ToArray();
    var forecast = Forecast.FromBytes(body);

    DisplayInfo<Forecast>
        .For(forecast)
        .SetExchange(eventArgs.Exchange)
        .SetQueue(queueName)
        .SetRoutingKey(eventArgs.RoutingKey)
        .SetVirtualHost(connectionFactory.VirtualHost)
        .Display(Color.Yellow);

    channel.BasicAck(eventArgs.DeliveryTag, multiple: false);
};
```

#### Step 7 - Consume Messages

```csharp
channel.BasicConsume(
    queue: queueName,
    autoAck: false,
    consumer: consumer);
```

#### Step 8 - Send Acknowledgements (ACKS)

```csharp
channel.BasicAck(eventArgs.DeliveryTag, multiple: false);
```

#### Full Listing

```csharp
namespace Rabbit.Example3.Consumer
{
    internal sealed class Program
    {
        private static void Main(string[] args)
        {
            Console.WriteLine($"EXAMPLE 3 : PUB/SUB : CONSUMER");

            var connectionFactory = new ConnectionFactory
            {
                HostName = "localhost",
                UserName = "admin",
                Password = "password"
            };

            var connection = connectionFactory.CreateConnection();

            var channel = connection.CreateModel();

            var queueName = channel.QueueDeclare().QueueName;

            const string ExchangeName = "example3_forecasts_exchange";

            channel.ExchangeDeclare(exchange: ExchangeName, type: ExchangeType.Fanout);

            channel.QueueBind(
                queue: queueName,
                exchange: ExchangeName,
                routingKey: string.Empty);

            var consumer = new EventingBasicConsumer(channel);

            consumer.Received += (sender, eventArgs) =>
            {
                var body = eventArgs.Body.ToArray();
                var forecast = Forecast.FromBytes(body);

                DisplayInfo<Forecast>
                    .For(forecast)
                    .SetExchange(eventArgs.Exchange)
                    .SetQueue(queueName)
                    .SetRoutingKey(eventArgs.RoutingKey)
                    .SetVirtualHost(connectionFactory.VirtualHost)
                    .Display(Color.Yellow);

                channel.BasicAck(eventArgs.DeliveryTag, multiple: false);
            };

            channel.BasicConsume(
                queue: queueName,
                autoAck: false,
                consumer: consumer);

            Console.ReadLine();
        }
    }
}
```

### Producer

#### Step 1 - Create Connection

```csharp
var connectionFactory = new ConnectionFactory
{
    HostName = "localhost",
    UserName = "admin",
    Password = "password"
};

using var connection = connectionFactory.CreateConnection();
```

#### Step 2 - Create Channel

```csharp
using var channel = connection.CreateModel();
```

#### Step 3 - Declare Exchange

```csharp
channel.ExchangeDeclare(exchange: ExchangeName, type: ExchangeType.Fanout);
```

#### Step 4 - Create and Publish Message

```csharp
var forecast = Thermometer.Fake().Report();

channel.BasicPublish(
    exchange: ExchangeName,
    routingKey: QueueName,
    body: Encoding.UTF8.GetBytes(forecast.ToJson())
);
```

#### Full Listing

```csharp
namespace Rabbit.Example3.Producer
{
    internal sealed class Program
    {
        internal static async Task Main()
        {
            Console.WriteLine($"EXAMPLE 3 : PUB/SUB : PRODUCER)");

            var connectionFactory = new ConnectionFactory
            {
                HostName = "localhost",
                UserName = "admin",
                Password = "password"
            };

            using var connection = connectionFactory.CreateConnection();

            using var channel = connection.CreateModel();

            const string QueueName = "";
            const string ExchangeName = "example3_forecasts_exchange";

            channel.ExchangeDeclare(exchange: ExchangeName, type: ExchangeType.Fanout);

            while (true)
            {
                var forecast = Thermometer.Fake().Report();

                channel.BasicPublish(
                    exchange: ExchangeName,
                    routingKey: QueueName,
                    body: Encoding.UTF8.GetBytes(forecast.ToJson())
                );

                DisplayInfo<Forecast>
                    .For(forecast)
                    .SetExchange(ExchangeName)
                    .SetQueue(QueueName)
                    .SetRoutingKey(QueueName)
                    .SetVirtualHost(connectionFactory.VirtualHost)
                    .Display(Color.Yellow);

                await Task.Delay(millisecondsDelay: 3000);
            }
        }
    }
}
```

---

## Running the Example

### Source Code Repository

All the code required to run this example can be found on [Github](https://github.com/drminnaar/dotnet-rabbitmq)

```bash

git clone https://github.com/drminnaar/dotnet-rabbitmq.git

```

### Manage RabbitMQ Server

For the example, RabbitMQ is hosted within a _Docker_ container.

The example code repository includes a _'docker-compose'_ file that describes the RabbitMQ stack with a reasonable set of defaults. Use _docker-compose_ to start, stop and display information about the RabbitMQ stack as follows:

```bash
# Verify that 'docker-compose' is installed
docker-compose --version

# Start RabbitMQ stack in the background
docker-compose up --detach

# Verify that RabbitMQ container is running
docker-compose ps

# Display RabbitMQ logs
docker-compose logs

# Display and follow RabbitMQ logs
docker-compose logs --tail="all" --follow

# Tear down RabbitMQ stack
# Remove named volumes declared in the `volumes`
# section of the Compose file and anonymous volumes
# attached to container
docker-compose down --volumes
```

### Start Producer

```bash

# open new terminal and run the following command
dotnet run -p ./Example3/Rabbit.Example3.Producer/

```

### Start Worker 1

```bash

# open new terminal and run the following command
dotnet run -p ./Example3/Rabbit.Example3.Consumer/

```

### Start Worker 2

```bash

# open new terminal and run the following command
dotnet run -p ./Example3/Rabbit.Example3.Consumer/

```

### Display

![example-pubsub-1](https://user-images.githubusercontent.com/33935506/115130034-12e0b700-a040-11eb-90aa-b6b2126fadbc.png)