
Quick Start
-----------

### Kafka Server on Docker

  1. Prepare the docker-compose work folder for your kafka
  ```bash
  $ mkdir -p /opt/kafka
  ```

  2. Download [docker-compose.yml](docker/docker-compose.yml) file and [.env](docker/.env) file and put them into `/opt/kafka`

  3. Configure the .env file
  ```ini
  HOST=<your_host_ip>
  ```

  4. Start and Run Kafka
  ```bash
  $ cd /opt/kafka
  $ docker-compose up -d
  ```

  1. About 30 seconds later. You can open the Kafka Control Center `http://<your_host_ip>:9021/` by your browser.

### Kafka Producer

  1. Create a project with .Net Core 2.2+.
  ```bash
  $ dotnet new console
  ```

  2. Add Kafka client library
  ```bash
  $ dotnet add package -v 1.2.2 Confluent.Kafka
  ```

  3. Put the following in your program.
  ```csharp
    using System;
    using Confluent.Kafka;

    class Program
    {
      public static void Main(string[] args)
      {
        var conf = new ProducerConfig { BootstrapServers = "<your_host_ip>:9091" };

        Action<DeliveryReport<Null, string>> handler = r => 
          Console.WriteLine(!r.Error.IsError
              ? $"Delivered message to {r.TopicPartitionOffset}"
              : $"Delivery Error: {r.Error.Reason}");

        using (var p = new ProducerBuilder<Null, string>(conf).Build())
        {
          for (int i=0; i<100; ++i)
          {
              p.Produce("my-topic", new Message<Null, string> { Value = i.ToString() }, handler);
          }

          // wait for up to 10 seconds for any inflight messages to be delivered.
          p.Flush(TimeSpan.FromSeconds(10));
        }
      }
    }
  ```


