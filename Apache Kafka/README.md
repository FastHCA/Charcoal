<a id="toc"></a>
  > Table of Content
  > + [Quick Start](#quick-start)
  >   + [System Requirement](#system-requirement)
  >   + [Run Kafka Server on Docker](#run-kafka-server-on-docker)
  >   + [Writing Kafka Producer](#writing-kafka-producer-net-core-22)
  >   + [Writing Kafka Consumer](#writing-kafka-consumer-net-core-22)

Quick Start
-----------

### System Requirement
  At least RAM 6 GB

  [Top](#toc)

### Run Kafka Server on Docker

  1. Prepare the docker-compose work folder for Kafka.
  ```bash
  $ mkdir -p /opt/kafka
  ```

  2. Download [docker-compose.yml](docker/docker-compose.yml) file and [.env](docker/.env) file and put them into kafka work folder `/opt/kafka`.

  3. Configure the .env file.
  ```ini
  HOST=<your_host_ip>
  ```

  4. Start and run Kafka.
  ```bash
  $ cd /opt/kafka
  $ docker-compose up -d
  ```

  5. Done.

  > About 30 seconds later. You can open the **Kafka Control Center** `http://<your_host_ip>:9021/` in your browser.

  [Top](#toc)

### Writing Kafka Producer (.Net Core 2.2+)

  1. Create a project with .Net Core 2.2+.
  ```bash
  $ mkdir <your_project_folder>
  $ cd <your_project_folder>
  $ dotnet new console
  ```

  2. Add Kafka client library
  ```bash
  $ dotnet add package -v 1.2.2 Confluent.Kafka
  ```

  3. Change the `BootstrapServers` config and put the following code into `Program.cs`.
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

  4. Run it.
  ```bash
  $ dotnet run
  ```

  > You will see the output like following
  > > ```
  > > Delivered message to my-topic [[0]] @0
  > > Delivered message to my-topic [[0]] @1
  > > Delivered message to my-topic [[0]] @2
  > > ...
  > > Delivered message to my-topic [[0]] @97
  > > Delivered message to my-topic [[0]] @98
  > > Delivered message to my-topic [[0]] @99
  > > ```

  [Top](#toc)

### Writing Kafka Consumer (.Net Core 2.2+)

  1. Create a project with .Net Core 2.2+.
  ```bash
  $ mkdir <your_project_folder>
  $ cd <your_project_folder>
  $ dotnet new console
  ```

  2. Add Kafka client library
  ```bash
  $ dotnet add package -v 1.2.2 Confluent.Kafka
  ```

  3. Change the `BootstrapServers` config and put the following code into `Program.cs`.
  ```csharp
  using System;
  using System.Threading;
  using Confluent.Kafka;

  class Program
  {
      public static void Main(string[] args)
      {
          var conf = new ConsumerConfig
          { 
              GroupId = "test-consumer-group",
              BootstrapServers = "<your_host_ip>:9091",
              // Note: The AutoOffsetReset property determines the start offset in the event
              // there are not yet any committed offsets for the consumer group for the
              // topic/partitions of interest. By default, offsets are committed
              // automatically, so in this example, consumption will only start from the
              // earliest message in the topic 'my-topic' the first time you run the program.
              AutoOffsetReset = AutoOffsetReset.Earliest
          };

          using (var c = new ConsumerBuilder<Ignore, string>(conf).Build())
          {
              c.Subscribe("my-topic");

              CancellationTokenSource cts = new CancellationTokenSource();
              Console.CancelKeyPress += (_, e) => {
                  e.Cancel = true; // prevent the process from terminating.
                  cts.Cancel();
              };

              try
              {
                  while (true)
                  {
                      try
                      {
                          var cr = c.Consume(cts.Token);
                          Console.WriteLine($"Consumed message '{cr.Value}' at: '{cr.TopicPartitionOffset}'.");
                      }
                      catch (ConsumeException e)
                      {
                          Console.WriteLine($"Error occured: {e.Error.Reason}");
                      }
                  }
              }
              catch (OperationCanceledException)
              {
                  // Ensure the consumer leaves the group cleanly and final offsets are committed.
                  c.Close();
              }
          }
      }
  }
  ```

  4. Run it.
  ```bash
  $ dotnet run
  ```

  > You will see the output like following
  > > ```
  > > Consumed message '0' at: 'my-topic [[0]] @0'.
  > > Consumed message '1' at: 'my-topic [[0]] @1'.
  > > Consumed message '2' at: 'my-topic [[0]] @2'.
  > > ...
  > > Consumed message '97' at: 'my-topic [[0]] @97'.
  > > Consumed message '98' at: 'my-topic [[0]] @98'.
  > > Consumed message '99' at: 'my-topic [[0]] @99'.
  > > ```
  > You can press `Ctrl+C` to exit.

  [Top](#toc)
