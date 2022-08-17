Apache Kafka is well known for its high performance. It is able to process a high rate of messages while maintaining low latency. [Apache Pulsar](https://kesque.com/what-is-apache-pulsar/) is a fast-growing alternative to Kafka. 

There are reports that suggest Pulsar has better performance characteristics than Kafka, but the raw results are not easy to find. Plus, those reports are based on older versions of Pulsar and Kafka, which are both fast-moving projects. 

So, in a series of posts, we are going to run the latest stable Kafka (2.3.0) and Pulsar (2.4.0) versions through a number of performance tests and publish those results.

In this first post, we are going to focus on latency. In future posts, we will look at throughput. Before we dive into the test results, let’s go over the test methodology.

## Performance testing of messaging systems

Both Kafka and Pulsar provide performance testing tools as part of their packages. While it would be possible to modify the performance tools from either to work with both, we are going to use a third-party benchmark framework from the [OpenMessaging Project](http://openmessaging.cloud/), a Linux Foundation Collaborative Project. 

The OpenMessaging Project has a goal of providing vendor-neutral and language-independent standards for messaging and streaming technologies. The project includes a performance testing framework that supports various messaging technologies. All the code used in these tests is in the OpenMessaging benchmark GitHub [repository](https://github.com/openmessaging/openmessaging-benchmark). The tests are designed to run in public cloud providers. In our case, we are going to run all the tests in Amazon Web Services (AWS) using standard EC2 instances.

We are publishing the full set of outputs from each test run as a series of GitHub gists. So, you are welcome to analyze the data and come up with your own insights. Of course, you could also run the tests yourself and generate new data. You should get similar results since we find the tests are reliable run after run even with different sets of EC2 instances. During our testing, we stood up and tore down the environment multiple times.

Although we used the OpenMessaging benchmark tools to run the tests which include a set of workloads, we are going to add some workloads that are inspired by the blog post on the LinkedIn Engineering site titled “[Benchmarking Apache Kafka: 2 Million Writes Per Second (On Three Cheap Machines)](https://engineering.linkedin.com/kafka/benchmarking-apache-kafka-2-million-writes-second-three-cheap-machines)” because we thought it would be interesting for comparison purposes. 

However, this is quite an old blog post. Today, hardware is better and what we are using is not necessarily cheap, though far from top-of-the-line. So — spoiler alert — both Kafka and Pulsar can handle 2 million writes per second in the blog title without breaking a sweat, which we will see in a future post.

## OpenMessaging benchmark

The OpenMessaging benchmark is a framework that is open and extensible. To add a messaging technology to test, you just need to add a Terraform configuration, an Ansible playbook, and an implementation of a Java library that controls the producers and consumers in the test harness. 

The Terraform configuration currently provided for Kafka and Pulsar are the same, starting the same set of EC2 instances in AWS. Obviously, for comparison purposes, it is important that the test is run on the same hardware configuration. 

So, the existing benchmark code makes it easy to compare Kafka and Pulsar. As we mentioned, all the code to run these tests is in the OpenMessaging GitHub [benchmark repository](https://github.com/openmessaging/openmessaging-benchmark).

The tests all begin with a warm-up period before starting the actual measuring phase of the test. The latency tests publish at a constant rate, recording the latency and throughput at regular intervals.

If you are planning to run these tests yourself, be warned. Running the tests in AWS is not cheap. A significant number of well-powered EC2 instances are needed. 

This makes sense when doing benchmarks of software. You want to make sure the hardware isn’t the bottleneck, so you oversubscribe the hardware. But this oversubscription comes at a cost (over $5 an hour, or $3,800 a month) that will make a mark on your AWS bill if you’re not careful. You don’t want to leave the environment running if you aren’t using it (for example, overnight). To avoid that fate, make sure you delete all the resources after running the tests.

## Performance considerations

Before we dive into the test results, there are a few important concepts we need to cover to put the results in context. 

First, we need to go over what the test is measuring: latency. Next, we need to go over the durability of the messages, especially when it comes to storing messages to disk. And last, we need to understand the different models of message replication used in Kafka and Pulsar.

There are many similarities between Kafka and Pulsar, but there are some significant differences that can impact performance. To fairly evaluate both systems, you need to understand these differences.

### A note on latency tests

All latency measurements necessarily include the network latency between the application and the messaging system. Assuming all tests are performed in the same network configuration and that network provides consistent latency, then the network latency is a constant that affects all tests equally. When comparing latency measurements, then, it is important the network is held constant when making comparisons.

We point this out because the latency in these tests differs from those published in the LinkedIn Engineering blog post. Those tests were run on (presumably) a dedicated 1 GB Ethernet network. Our tests are run in the black-box network of a public cloud provider on instances that provide “Up to 10 Gigabit” network performance. 

So, the latency numbers in this test cannot be directly compared with those in that blog post. However, we should be able to compare the latency results between the two messaging systems in our tests, since we are keeping the network configuration constant.

There are two types of latency we are measuring: end-to-end latency and publishing latency.

### End-to-end latency

Let’s start by discussing end-to-end latency since it is relatively straightforward. End-to-end latency is simply the time from when the message is sent by the producer to when it’s received by the consumer. 

In both the Pulsar and Kafka implementations of the benchmark, the publish timestamp is generated by the API when sending the message. When the message is received by the consumer, a second timestamp is taken. The difference between these two times is the end-to-end latency.

Where end-to-end latency gets complicated is with the clocks used to take those timestamp measurements. When measuring end-to-end latency, the clocks used for the timestamps must be synchronized. If they are not synchronized, the difference between the clocks will impact your measurements. You end up measuring how much the clocks are different as well as the messaging system latency. Since clocks drift over time, the problem becomes worse in long-running tests.

Ideally, the producer and consumer are on the same server and therefore the timestamps are taken using the same clock, so there is no difference. Unfortunately, the benchmark tests are designed to separate the producer and consumer on to different servers to distribute the load.

The next best option is to synchronize the clocks between the servers as accurately as possible. Luckily, AWS has a free [time sync service](https://aws.amazon.com/blogs/aws/keeping-time-with-amazon-time-sync-service/) that, combined with [chrony](https://chrony.tuxfamily.org/), appears to keep the clocks between EC2 instances in very close sync (i.e., within a handful of microseconds of the reference clock). For these tests, we installed chrony on all the client servers and configured them to use the AWS time source.

### Publishing latency

Publishing latency is the amount of time that passes from when the message is sent until the time an acknowledgment is received from the messaging system. The acknowledgment indicates that the messaging system has persisted the message and will guarantee its delivery. 

Essentially, the acknowledgment indicates the responsibility for handling the message has been successfully passed from the producing application to the messaging system. Low, consistent publishing latency is good for applications. 

When the application is ready to hand off the delivery of a message, the messaging system quickly accepts the message, allowing the application to continue working on application-level concerns, such as business logic. This handoff of responsibility for the message is a key feature of any messaging system.

In the benchmark tests, the messages are sent asynchronously, so the producer doesn’t actually block waiting for the acknowledgment of a message. The call to send the message returns immediately and a callback handles the acknowledgment when it arrives. It may seem that publishing latency is not that important when doing asynchronous sends, but it is. Both the Kafka (max.in.flight.requests.per.connection) and Pulsar (maxPendingMessages) producers have a buffer for holding unacknowledged messages. 

If this buffer fills up, calls to the send method will start to block (or fail, depending on configuration). So, if the messaging system does not quickly acknowledge messages, it can lead to the producing application waiting for the messaging system.

In the benchmark tests, the publishing latency is measured from the time when calling the send method until the acknowledgment callback is triggered. These two timestamps are done in the producer, so clock synchronization is not a concern.

### Durability and flushing messages to disk

Durability in the context of a messaging system means that if parts of the system fail, messages are not lost. To ensure this, messages need to be stored somewhere that will survive a crash of the server running the messaging system software. 

Both Kafka and Pulsar ultimately write messages to disk as a way to provide durability. However, telling the operating system to write a message to the file system is not sufficient. All POSIX systems cache reads and writes to the file system in memory for improved performance. Writing to the file system only means that the data has been put into the write cache, but it is not necessarily stored safely on the physical disk. 

Since this cache resides in memory if the server crashes (for example, power loss and kernel panic), data that has been written to the cache but not yet written or flushed to disk will be lost. The operation that forces cached data to be written to the physical disk is called [fsync](http://man7.org/linux/man-pages/man2/fdatasync.2.html). To guarantee that a message has been stored on disk, the file cache has to be flushed to disk after writing each message by triggering an fsync operation.

By default, Kafka does not explicitly flush each message to disk. It leaves the decision when to flush to the operating system. So in the event of a server crash, some undefined amount of message data can be lost. 

This is the default setting for performance reasons. Writing to the physical disk is slower than writing to the memory cache, so this flushing slows down message processing performance. It is possible to configure Kafka to flush messages regularly (or even for every message), but the Kafka documentation recommends against this for efficiency reasons.

Pulsar, on the other hand, flushes each message to the disk by default. The message is not acknowledged to the producer until it is stored on the physical disk. This provides stronger durability guarantees if the server crashes. It also, as we shall see, is able to provide these durability guarantees while maintaining high performance. 

It is able to do this because it uses [Apache Bookkeeper](https://bookkeeper.apache.org/) to store messages, which is a distributed log storage system that has been optimized for this purpose. It is, however, possible to disable this fsync behavior in Pulsar.

Since the flushing to disk can have an impact on performance, we are going to run the performance tests twice for both Kafka and Pulsar — once with flushing each message to disk enabled and once with it disabled. This will allow for a better apples-to-apples comparison between the two systems.

### Message replication

Both Kafka and Pulsar provide additional message durability by making replicas of each message. That way, even if one of the copies of the message is lost for some reason, there will be other copies available for recovery. Message replication has an effect on performance and is implemented differently on Kafka and Pulsar. We want to make sure we are getting similar replication behavior between Kafka and Pulsar in our tests.

Kafka uses a leader-follower replication model. One of the Kafka brokers is elected the leader for a partition. All messages are initially written to the leader, and the followers read and replicate the messages from the leader. 

Kafka monitors if each follower is caught up, or “in sync” with the leader. With Kafka, you can control the total number of copies (replication-factor) that are made of a message and the minimum number of replicas that need to be in-sync (min.insync.replicas) before a message is considered successfully stored and acknowledged to the producer. 

A typical configuration would have Kafka make three copies of a message and only acknowledge once at least two (a majority) have been confirmed successfully written. This is the configuration (replication-factor=3, in.sync.replicas=2, acks=all) we will use for all our Kafka tests.

Pulsar uses a quorum-vote replication model. Multiple copies of the message (write quorum) are written in parallel. Once some number of copies have been confirmed stored, then the message is acknowledged (ack quorum). 

Unlike the leader-follower model which generally writes copies to the same set of leaders and followers for a particular topic partition, Pulsar can spread (or stripe) the copies over a set of storage nodes (ensemble) which can improve the read and write performance. In our tests, we only have three storage nodes and want to make three copies like with the Kafka configuration, so our settings will not take advantage of striping. Our configuration for Pulsar (ensemble=3, write quorum=3, ack quorum=2) gives similar replication behavior as Kafka: three copies of the messages, acknowledged after two are confirmed.

Now that we have covered off some of these important concepts, let’s move on to the details of the tests.

## Setting up the benchmark

To set up the benchmark tests, we followed the steps documented on the [OpenMessaging site](http://openmessaging.cloud/docs/benchmarks/). After applying the Terraform configuration, you get the following set of EC2 instances:

| Count | EC2 Instance | Pulsar Purpose                      | Kafka Purpose |
| ----- | ------------ | ----------------------------------- | ------------- |
| 3     | i3.4xlarge   | Pulsar Broker and Bookkeeper Bookie | Kafka Broker  |
| 3     | t2.small     | Zookeeper                           | Zookeeper     |
| 4     | c5.2xlarge   | Test Client                         | Test Client   |

The i3.4xlarge instances used for Pulsar/Bookkeeper and the Kafka broker include two NVMe SSDs for high disk performance. These are pretty powerful virtual machines, sporting 16 vCPUs, 122 GiB of memory along with their high-performance disks. Having two SSDs is ideal for the Pulsar setup since it writes two streams of data which can be parallelized on the disks. Kafka also takes advantage of these two SSDs by distributing partitions over both drives.

The Ansible playbook for both Pulsar and Kafka tunes the performance for low latency using the [tuned-adm](https://linux.die.net/man/1/tuned-adm) command (latency-performance profile).

## Workloads

Although the benchmark comes with a set of workloads that we could run right out of the box, we are going to modify them a little bit so that they line up more closely with the benchmark results for Kafka in the LinkedIn Engineering blog. 

Defining new workloads is easy. You just need to create a YAML file with the updated parameters to use for the test.

If you look at tests in the LinkedIn blog, you will see that they are all run with 100-byte messages. The reason given “for focusing on small records in these tests is that it is the harder case for a messaging system (generally) ” 

That makes sense since there is a fixed amount of work to be done for each message regardless of its size, so small messages measure how efficient the system is in processing a message. More efficiency generally leads to higher performance. It also reduces the possibility that the test is being impacted by throughput limits of the network or disk. How well a messaging system performs when handling large messages can be an interesting benchmark. But for now, we are going to focus on small messages.

The other change from the stock benchmark workloads is that we are adding a six-partition test. Six partitions are used extensively in the LinkedIn tests, so we wanted to include that workload in our set.

You may notice that the LinkedIn blog includes producer-only and consumer-only workloads. All our workloads are going to include both producers and consumers for two reasons. 

First, as it stands, the benchmark tests do not support standalone producer-only or consumer-only workloads. Second, in the real world, messaging systems will always be serving producers and consumers at the same time. So, both producing and consuming messages during the test gives a more realistic test scenario.

All that being said, here is the set of workloads we used for these tests:

| Topics | Partitions | Message Size | Consumer Groups/ Subscriptions | Producers | Consumers | Producer Rate (msg/s) | Consumer Rate (msg/s) | Duration (min) |
| ------ | ---------- | ------------ | ------------------------------ | --------- | --------- | --------------------- | --------------------- | -------------- |
| 1      | 1          | 100 Byte     | 1                              | 1         | 1         | 50 000                | 50 000                | 15             |
| 1      | 6          | 100 Byte     | 1                              | 1         | 1         | 50 000                | 50 000                | 15             |
| 1      | 16         | 100 Byte     | 1                              | 1         | 1         | 50 000                | 50 000                | 15             |

A Kafka consumer group and a Pulsar subscription are similar concepts. They both allow one or more consumers to receive all the messages on a topic. If a topic has multiple consumer groups/subscriptions associated with it, the messaging system is providing multiple copies of each message in the topic, or “fanning out” the message. 

For each message published into the topic, one message is sent to each consumer group/subscription. If all messages are sent to a single topic that has a single consumer group/subscription, then the producer rate and consumer rate are equal. If, for example, there are two consumer groups/subscriptions, then the consumer rate is double the producer rate. 

For these tests, we are keeping things simple. There is only one consumer group/subscription, so the producer rate and consumer rate are equal.

## Apache Pulsar results

The following sections present the latency results for the Apache Pulsar tests. We first present the results with per-message flushing enabled since this is how Pulsar works out of the box, followed by the results with per-message flushing disabled. 

For each workload, we include two graphs: One for the 99th percentile publishing latency over the duration of the test, and one for the average end-to-end latency. The graphs are followed by a table that summarizes the latency distribution. The latency measurements reported in the tables are aggregated for the duration of the test. The percentile calculations for end-to-end latency have less precision than the publishing latency since the end-to-end calculations are using the timestamp that is automatically placed in the message header and that timestamp only has ms precision. The publishing latency is calculated using nanosecond precision.

All tests used a 100-byte message. The produce and consume rates were a constant 50,000 msgs/s during the 15-minute duration of each test. Only two client servers were used. The version of Apache Pulsar used for the tests was 2.4.0.

### Latency with Flush

#### TEST 1: 1 TOPIC, 1 PARTITION

You can get the command output and raw results [here](https://gist.github.com/cdbartholomew/6980295b60799832981db1be3eacfb11). Click on “view raw” to see a larger version of a graph.

![1topic-1partition-100b-PublishLatency99thPct.png](https://lh6.googleusercontent.com/S_-VwkaIMQo487pW79p32UgRXgGFU5_57MttAQH57Vc77xhgf5FD6E9lV7IW6pPfiVGRBt4G-Tpz7rA020n0pryTdfRyQE_ibUT2rgLip-1UBgiRPBNiHX3yFTsT7TUCOgvgp7Nb)

[view raw](https://gist.github.com/cdbartholomew/6980295b60799832981db1be3eacfb11/raw/7360d20bea9be704b62d1908dc1145bc9ed8d7ee/1topic-1partition-100b-PublishLatency99thPct.png)[1topic-1partition-100b-PublishLatency99thPct.png](https://gist.github.com/cdbartholomew/6980295b60799832981db1be3eacfb11#file-1topic-1partition-100b-publishlatency99thpct-png) hosted with by [GitHub](https://github.com/)

![1topic-1partition-100b-End-to-endLatencyAvg.png](https://lh4.googleusercontent.com/MWkzLRpnFQS5byZkavDrGnDVAlxjw9rwZ-630aHC15Xyr3lbRIbRTgrYauIyxp1f3qIVc1rbI2V2_swVJHJZZPhxY4R8qO374xj99iMqG_RdLc8dqFCscKeRuckBjbcJrIK1sW-t)

[view raw](https://gist.github.com/cdbartholomew/6980295b60799832981db1be3eacfb11/raw/7360d20bea9be704b62d1908dc1145bc9ed8d7ee/1topic-1partition-100b-End-to-endLatencyAvg.png)[1topic-1partition-100b-End-to-endLatencyAvg.png](https://gist.github.com/cdbartholomew/6980295b60799832981db1be3eacfb11#file-1topic-1partition-100b-end-to-endlatencyavg-png) hosted with by [GitHub](https://github.com/)

 

| Latency Type              | Average | 50th  | 75th  | 95th  | 99th  | 99.9th | 99.99th | Maximum |
| ------------------------- | ------- | ----- | ----- | ----- | ----- | ------ | ------- | ------- |
| Publishing (milliseconds) | 2.969   | 2.916 | 3.481 | 3.911 | 4.003 | 4.095  | 52.958  | 266.671 |
| End-to-end (milliseconds) | 9.052   | 9.0   | 11.0  | 13.0  | 14.0  | 128.0  | 213.0   | 267.0   |

#### TEST 2: 1 TOPIC, 6 PARTITIONS

You can get the command output and raw results [here](https://gist.github.com/cdbartholomew/fd1a33596cfb89e1f7b561acea7cb670). Click on “view raw” to see a larger version of a graph.

![1topic-6partition-100b-PublishLatency99thPct.png](https://lh3.googleusercontent.com/KKC8DZUfxwPaPFPzyVVpS9KPWdpeLHE-9XG66Q5Ni6jRuCkB0du7XVaTtBB8oVYG-t_ramwHTfIdPU5bU6JaNP4DKyOTxh8YrOZxmYHQVuzPlLMR3HBhHCYRCqmUJaHE3ufrxt1Z)

[view raw](https://gist.github.com/cdbartholomew/fd1a33596cfb89e1f7b561acea7cb670/raw/170dc999a46568cbda832d6e4fb198cd48e4f0e7/1topic-6partition-100b-PublishLatency99thPct.png)[1topic-6partition-100b-PublishLatency99thPct.png](https://gist.github.com/cdbartholomew/fd1a33596cfb89e1f7b561acea7cb670#file-1topic-6partition-100b-publishlatency99thpct-png) hosted with by [GitHub](https://github.com/)

![1topic-6partition-100b-End-to-endLatencyAvg.png](https://lh6.googleusercontent.com/ciCXM8utc8ycAvLycHikhfBCnj1F4_KkY70nZ4Bq1gydf2R25cFoWIeJzLcGvi1IWk5ICJbsq5wZ6b7vMiNKItYCnijwzzgPZO9UN3AQuxLEVW1ips84KzlWQTeG6_a-erEvc-yo)

[view raw](https://gist.github.com/cdbartholomew/fd1a33596cfb89e1f7b561acea7cb670/raw/170dc999a46568cbda832d6e4fb198cd48e4f0e7/1topic-6partition-100b-End-to-endLatencyAvg.png)[1topic-6partition-100b-End-to-endLatencyAvg.png](https://gist.github.com/cdbartholomew/fd1a33596cfb89e1f7b561acea7cb670#file-1topic-6partition-100b-end-to-endlatencyavg-png) hosted with by [GitHub](https://github.com/)

| Latency Type              | Average | 50th  | 75th  | 95th  | 99th  | 99.9th | 99.99th | Maximum |
| ------------------------- | ------- | ----- | ----- | ----- | ----- | ------ | ------- | ------- |
| Publishing (milliseconds) | 2.800   | 2.775 | 3.368 | 3.848 | 3.978 | 4.153  | 11.224  | 252.395 |
| End-to-end (milliseconds) | 8.060   | 8.0   | 10.0  | 13.0  | 14.0  | 110.0  | 199.0   | 253.0   |

#### TEST 3: 1 TOPIC, 16 PARTITIONS

You can get the command output and raw results [here](https://gist.github.com/cdbartholomew/cfb5e08d98370824b8cdd3790be043e6). Click on “view raw” to see a larger version of a graph.

![1topic-16partition-100b-PublishLatency99thPct.png](https://lh5.googleusercontent.com/2tdsM6Y5CPPFJhyTmHQlWEn2FlSesDVWgDlGK38dyI-gnlBjFWvoF_ujh-9Z51M1fQzfalreaXSF_TnlJwZrG4WwKSlGgNvX9t-89yF_K8y2qNtrbBGlCO7NHv72PBDffiORrJDE)

[view raw](https://gist.github.com/cdbartholomew/cfb5e08d98370824b8cdd3790be043e6/raw/55bf4301dad6eea797058e52ccd35be0ed070f84/1topic-16partition-100b-PublishLatency99thPct.png)[1topic-16partition-100b-PublishLatency99thPct.png](https://gist.github.com/cdbartholomew/cfb5e08d98370824b8cdd3790be043e6#file-1topic-16partition-100b-publishlatency99thpct-png) hosted with by [GitHub](https://github.com/)

![1topic-16partition-100b-End-to-endLatencyAvg.png](https://lh4.googleusercontent.com/lpjqmUd7xazAkmRv3E2CqqMiEz5tKj4GnXfGkigVne9YF3vOJHjdkTg1TnD5fPejQLnyw__-n-A65aTuEv-29UH7sQQtJ5oeMS8Bvc76pOPXmpq9tNBM_43mORDkf-YETO14OpuO)

[view raw](https://gist.github.com/cdbartholomew/cfb5e08d98370824b8cdd3790be043e6/raw/55bf4301dad6eea797058e52ccd35be0ed070f84/1topic-16partition-100b-End-to-endLatencyAvg.png)[1topic-16partition-100b-End-to-endLatencyAvg.png](https://gist.github.com/cdbartholomew/cfb5e08d98370824b8cdd3790be043e6#file-1topic-16partition-100b-end-to-endlatencyavg-png) hosted with by [GitHub](https://github.com/)

| Latency Type              | Average | 50th  | 75th  | 95th  | 99th  | 99.9th | 99.99th | Maximum |
| ------------------------- | ------- | ----- | ----- | ----- | ----- | ------ | ------- | ------- |
| Publishing (milliseconds) | 2.723   | 2.721 | 3.332 | 3.843 | 3.977 | 4.135  | 9.712   | 254.427 |
| End-to-end (milliseconds) | 3.170   | 3.0   | 4.0   | 4.0   | 12.0  | 89.0   | 178.0   | 255.0   |

#### DISCUSSION

Since partitions are a unit of parallelism in both Pulsar and Kafka, we would expect to see latency reduction as we increase the number of partitions, and this is exactly the result we get. 

 

Across the board, as the number of partitions increases, both the publishing and the end-to-end latency decreases. There are some outliers in each test, but the maximum latency never exceeds 267 ms. The publishing latency is more tightly bounded than the end-to-end latency. The 99.99th percentile in publishing latency never exceeds 11.6 ms in any of the tests. The effect of extra partitions on latency is most apparent in the end-to-end latency results with 16 partitions. The 16-partition tests record an average latency (3 ms) that is one-third of the 1-partition tests (9 ms).

Pulsar provides consistent publishing latency over time. All the tests run for 15 minutes. As shown in the graphs, the average publishing latency shows very little variation over the duration of the test. The end-to-end latency shows some variability over time, with the average latency increasing by about 2 ms at a regular period of about 90 seconds. 

Interestingly, this 2-ms periodic bump appears to be constant regardless of the end-to-end latency. For example, the average end-to-end latency is 9 ms for one partition and only 3 ms for 16 partitions, but the spike is 2 ms (9 to 11, 3 to 5) in both cases.

### Latency without Flush

The following tests are identical to the previous set, except that per-message flushing to disk is disabled by setting journalSyncData=false in the bookkeeper.conf file and restarting the software (Pulsar broker and Bookkeeper).

#### TEST 4: 1 TOPIC, 1 PARTITION

You can get the command output and raw results [here](https://gist.github.com/cdbartholomew/cc983d991f7d391ecbc0d359414d59ae). Click on “view raw” to see a larger version of a graph.

![1topic-1partition-100b-noflush-PublishLatency99thPct.png](https://lh4.googleusercontent.com/1c442jsiLSEcQmw8ZWpMm2mnBM6Ed8kauML7PJ3Tcvw3c2QPGrM30KEGYUD76Dw8jRu2k1y0jdXJz3XVM50k38pgji4wYLmmTg5yY8Kx7DdFM9YUUPTMwOfBpahyLxszwMeQtJ7n)

[view raw](https://gist.github.com/cdbartholomew/cc983d991f7d391ecbc0d359414d59ae/raw/eb82cfc97455b0c2606d434c2e047efa5804911e/1topic-1partition-100b-noflush-PublishLatency99thPct.png)[1topic-1partition-100b-noflush-PublishLatency99thPct.png](https://gist.github.com/cdbartholomew/cc983d991f7d391ecbc0d359414d59ae#file-1topic-1partition-100b-noflush-publishlatency99thpct-png) hosted with by [GitHub](https://github.com/)

![1topic-1partition-100b-noflush-End-to-endLatencyAvg.png](https://lh3.googleusercontent.com/cckw2gkBAE0P-MHedmk86N2T8dZ3JICwRIt9myDh8qCrbme8nUZvhKx874PBLfjZwrWY4hBNcebDaJi1KCaf_ZMKVVN93EbOvkXO1lLGByJcaskIRGQdT4RDAZRXN9rJ7e-Z-aw1)

[view raw](https://gist.github.com/cdbartholomew/cc983d991f7d391ecbc0d359414d59ae/raw/eb82cfc97455b0c2606d434c2e047efa5804911e/1topic-1partition-100b-noflush-End-to-endLatencyAvg.png)[1topic-1partition-100b-noflush-End-to-endLatencyAvg.png](https://gist.github.com/cdbartholomew/cc983d991f7d391ecbc0d359414d59ae#file-1topic-1partition-100b-noflush-end-to-endlatencyavg-png) hosted with by [GitHub](https://github.com/)

| Latency Type              | Average | 50th  | 75th  | 95th  | 99th  | 99.9th | 99.99th | Maximum |
| ------------------------- | ------- | ----- | ----- | ----- | ----- | ------ | ------- | ------- |
| Publishing (milliseconds) | 2.726   | 2.694 | 3.245 | 3.668 | 3.783 | 3.928  | 4.508   | 253.055 |
| End-to-end (milliseconds) | 8.819   | 9.0   | 11.0  | 13.0  | 14.0  | 108.0  | 205.0   | 253.0   |

#### TEST 5: 1 TOPIC, 6 PARTITIONS

You can get the command output and raw results [here](https://gist.github.com/cdbartholomew/0349153b0a6cb45559e77560dec61bca). Click on “view raw” to see a larger version of a graph.

![1topic-6partition-100b-noflush-PublishLatency99thPct.png](https://lh4.googleusercontent.com/Tf31_hOAmRWOanxoqxmrYbx_L3xj7uh6Zr9hPfP2Gyv7hTPIubm2YZX1X2MGglxP49T1QO028yhCQH_iHdV9erWOxRtfjEXmEvu6mMryA765AxN4nZTZIsl1TUlFTJk4XQZolQPe)

[view raw](https://gist.github.com/cdbartholomew/0349153b0a6cb45559e77560dec61bca/raw/108eae7dc66c2da52df41fd146c4aa8781475413/1topic-6partition-100b-noflush-PublishLatency99thPct.png)[1topic-6partition-100b-noflush-PublishLatency99thPct.png](https://gist.github.com/cdbartholomew/0349153b0a6cb45559e77560dec61bca#file-1topic-6partition-100b-noflush-publishlatency99thpct-png) hosted with by [GitHub](https://github.com/)

![1topic-6partition-100b-noflush-End-to-endLatencyAvg.png](https://lh4.googleusercontent.com/daih6ry1e8UuSqkGeevnuqMFig68_-lPgwtHDjE_vXQy8yN_IEDD9muUHAsd4GD9hXqNTQGywhzJRdYm4WyfrZeGOEt5hEIFh3NBIzceFICysL6fZjilIXgMi8lWBEHAEP8_z4f4)

[view raw](https://gist.github.com/cdbartholomew/0349153b0a6cb45559e77560dec61bca/raw/108eae7dc66c2da52df41fd146c4aa8781475413/1topic-6partition-100b-noflush-End-to-endLatencyAvg.png)[1topic-6partition-100b-noflush-End-to-endLatencyAvg.png](https://gist.github.com/cdbartholomew/0349153b0a6cb45559e77560dec61bca#file-1topic-6partition-100b-noflush-end-to-endlatencyavg-png) hosted with by [GitHub](https://github.com/)

| Latency Type              | Average | 50th  | 75th  | 95th  | 99th  | 99.9th | 99.99th | Maximum |
| ------------------------- | ------- | ----- | ----- | ----- | ----- | ------ | ------- | ------- |
| Publishing (milliseconds) | 2.670   | 2.634 | 3.211 | 3.679 | 3.809 | 3.952  | 5.212   | 239.408 |
| End-to-end (milliseconds) | 7.930   | 8.0   | 10.0  | 13.0  | 13.0  | 116.0  | 215.0   | 244.0   |

#### TEST 6: 1 TOPIC, 16 PARTITIONS

You can get the command output and raw results [here](https://gist.github.com/cdbartholomew/b51ee90f1d9b99e3e9e8c3eec2a64933). Click on “view raw” to see a larger version of a graph.

![1topic-16partition-100b-noflush-PublishLatency99thPct.png](https://lh5.googleusercontent.com/V48rI8OAlxSf1TIoy2zrwETH_ZBSOVi6B4yn96NOeQ4oUsAcqrzUwbH-I89pvKEbTEGp04l5QwCHRTMg3CQDECi5zS6qdVJjqz8yGK88nPmxMJ-6SvVY6W-PDe-vmyx7AgMdE4gF)

[view raw](https://gist.github.com/cdbartholomew/b51ee90f1d9b99e3e9e8c3eec2a64933/raw/697d30b69db6de765c9d1194ddb24a9b5d3747c9/1topic-16partition-100b-noflush-PublishLatency99thPct.png)[1topic-16partition-100b-noflush-PublishLatency99thPct.png](https://gist.github.com/cdbartholomew/b51ee90f1d9b99e3e9e8c3eec2a64933#file-1topic-16partition-100b-noflush-publishlatency99thpct-png) hosted with by [GitHub](https://github.com/)

![1topic-16partition-100b-noflush-End-to-endLatencyAvg.png](https://lh4.googleusercontent.com/YzwN_5Ld1P5zld9-3xbJ9IcOAKq1zy0-TrejtTZTRHgeEt9EXncMTxKYiAOl-EoqEXl9yZFvPdqA-2FmdH2ouLe9cR1_2NHG_XfqlDY-l2QYchzpdjvQLMMzTt30JXISlqZfcHQo)

[view raw](https://gist.github.com/cdbartholomew/b51ee90f1d9b99e3e9e8c3eec2a64933/raw/697d30b69db6de765c9d1194ddb24a9b5d3747c9/1topic-16partition-100b-noflush-End-to-endLatencyAvg.png)[1topic-16partition-100b-noflush-End-to-endLatencyAvg.png](https://gist.github.com/cdbartholomew/b51ee90f1d9b99e3e9e8c3eec2a64933#file-1topic-16partition-100b-noflush-end-to-endlatencyavg-png) hosted with by [GitHub](https://github.com/)

| Latency Type              | Average | 50th  | 75th | 95th  | 99th  | 99.9th | 99.99th | Maximum |
| ------------------------- | ------- | ----- | ---- | ----- | ----- | ------ | ------- | ------- |
| Publishing (milliseconds) | 2.677   | 2.655 | 3.23 | 3.691 | 3.825 | 3.994  | 20.883  | 265.625 |
| End-to-end (milliseconds) | 3.165   | 3.0   | 3.0  | 4.0   | 12.0  | 104.0  | 190.0   | 265.0   |

#### DISCUSSION

As expected, the no-flush results give lower latency, but not very much. For example, the 99th percentile publishing latency with one partition when flushing to disk is 4.129 ms but drops to only 3.928 ms when not flushing. In fact, in the 16-partition test, there is little difference between the flush and no-flush cases. The periodic 2-ms spike in end-to-end latency still exists in these tests with the same time interval.

Given the durability tradeoff that comes with disabling flushing to disk, it hardly seems worth disabling it from a latency standpoint when using Apache Pulsar.

## Apache Kafka results

Since the default behavior for Kafka is to not flush each message to disk, we are going to start with those results, followed by the results when flushing each message to disk. As with the Pulsar tests, all tests used a 100-byte message and had a message rate of 50,000 msgs/s. Only two client servers were used. The latency measurements reported in the tables are aggregated for the duration of the test.

The version of Apache Kafka for these tests was 2.11-2.3.0.

### Latency with No Flush

#### TEST 7: 1 TOPIC, 1 PARTITION

You can get the command output and raw results [here](https://gist.github.com/cdbartholomew/5592b22fcd4ebdedfd6b6ceca9fab1ae). Click on “view raw” to see a larger version of a graph.

![1topic-1partition-100b-PublishLatency99thPct.png](https://lh6.googleusercontent.com/aR56Pi4AbhVI0LMCVw7jKxEe_OLW9sfC3S980q2CaBdgZuokotAmrj0Wzz5ImLQACXot_9d2TIsBBXs7rUfRUuO1tZxTaQ91rkaBa0Thply0jYuTqgmkc0oXTA1NZvvxZulKNbLj)

[view raw](https://gist.github.com/cdbartholomew/5592b22fcd4ebdedfd6b6ceca9fab1ae/raw/471b1e5a6c3c1e867ec40ab53569d86134fe18a1/1topic-1partition-100b-PublishLatency99thPct.png)[1topic-1partition-100b-PublishLatency99thPct.png](https://gist.github.com/cdbartholomew/5592b22fcd4ebdedfd6b6ceca9fab1ae#file-1topic-1partition-100b-publishlatency99thpct-png) hosted with by [GitHub](https://github.com/)

![1topic-1partition-100b-End-to-endLatencyAvg.png](https://lh5.googleusercontent.com/cOosN90BmBOjh0mUaNRSGzz270qVs2jJKoVrToi0yxI6zQBqgTANHgR_yDbP4r5Uv53p4Squmjoii7uLc_YgxH4bDtQ_Vv2f42kUERNYtg9adeJi6DmV9udBrGmHnKSKMRSrI4WG)

[view raw](https://gist.github.com/cdbartholomew/5592b22fcd4ebdedfd6b6ceca9fab1ae/raw/471b1e5a6c3c1e867ec40ab53569d86134fe18a1/1topic-1partition-100b-End-to-endLatencyAvg.png)[1topic-1partition-100b-End-to-endLatencyAvg.png](https://gist.github.com/cdbartholomew/5592b22fcd4ebdedfd6b6ceca9fab1ae#file-1topic-1partition-100b-end-to-endlatencyavg-png) hosted with by [GitHub](https://github.com/)

| Latency Type              | Average | 50th  | 75th  | 95th  | 99th | 99.9th  | 99.99th | Maximum |
| ------------------------- | ------- | ----- | ----- | ----- | ---- | ------- | ------- | ------- |
| Publishing (milliseconds) | 2.191   | 1.733 | 2.157 | 2.732 | 3.15 | 149.616 | 201.701 | 225.463 |
| End-to-end (milliseconds) | 2.865   | 2.0   | 2.0   | 3.0   | 7.0  | 189.0   | 277.0   | 341.0   |

#### TEST 8: 1 TOPIC, 6 PARTITIONS

You can get the command output and raw results [here](https://gist.github.com/cdbartholomew/edda1382cc7eb743b0e7d9650187eb4d). Click on “view raw” to see a larger version of a graph.

![1topic-6partition-100b-PublishLatency99thPct.png](https://lh5.googleusercontent.com/Nc_qHePeHOUWhPMlfvavpNJ-R8XYJ2mff883iYsTw2Fppc9Yu_w6NrD4xO46Jype-DcVmD8lOc7IpKLXxssW-DU2hqIp6x7Le3zedkvkDMN-97AgaMutvjYSuv8EoVhpeHLNbvai)

[view raw](https://gist.github.com/cdbartholomew/edda1382cc7eb743b0e7d9650187eb4d/raw/d51c9d9147bae3cdb6abf29e6a3a3d2261165739/1topic-6partition-100b-PublishLatency99thPct.png)[1topic-6partition-100b-PublishLatency99thPct.png](https://gist.github.com/cdbartholomew/edda1382cc7eb743b0e7d9650187eb4d#file-1topic-6partition-100b-publishlatency99thpct-png) hosted with by [GitHub](https://github.com/)

![1topic-6partition-100b-End-to-endLatencyAvg.png](https://lh4.googleusercontent.com/d1h5WzQS_rWr6Kulo5e6sFHCp3UqkxIFFjbKYWIZ-XuS__dGGh_YuZpafcH3QCvQL7m3d1SWyts519q9EghVpcdFBxNMPPhZGdWNFECfx3JRSpLXcFNRiQVFVJd8mJzVfWN1zVoB)

[view raw](https://gist.github.com/cdbartholomew/edda1382cc7eb743b0e7d9650187eb4d/raw/d51c9d9147bae3cdb6abf29e6a3a3d2261165739/1topic-6partition-100b-End-to-endLatencyAvg.png)[1topic-6partition-100b-End-to-endLatencyAvg.png](https://gist.github.com/cdbartholomew/edda1382cc7eb743b0e7d9650187eb4d#file-1topic-6partition-100b-end-to-endlatencyavg-png) hosted with by [GitHub](https://github.com/)

| Latency Type              | Average | 50th  | 75th  | 95th  | 99th  | 99.9th  | 99.99th | Maximum |
| ------------------------- | ------- | ----- | ----- | ----- | ----- | ------- | ------- | ------- |
| Publishing (milliseconds) | 4.475   | 3.657 | 5.814 | 7.474 | 8.349 | 139.389 | 205.085 | 213.136 |
| End-to-end (milliseconds) | 6.508   | 5.0   | 7.0   | 9.0   | 19.0  | 189.0   | 220.0   | 277.0   |

#### TEST 9: 1 TOPIC, 16 PARTITIONS

You can get the command output and raw results [here](https://gist.github.com/cdbartholomew/8e9c8ef9bfe6943a1f7578efa6cfc603). Click on “view raw” to see a larger version of a graph.

![1topic-16partition-100b-PublishLatency99thPct.png](https://lh6.googleusercontent.com/e65NJpp0qirq1PyKwFWpdPZILmOwVxgNXZMMh_9JqWw9_ADG91vVCX23WYLFlPLp-kxC83pHP_2gB101ju1I2Gi6QyBJpgCv1tW3kTbTb-cLgQp-ae69KF8ua49vib9wxa9iNCxf)

[view raw](https://gist.github.com/cdbartholomew/8e9c8ef9bfe6943a1f7578efa6cfc603/raw/d1173492a68c2e70333a9626e162969c1354fc15/1topic-16partition-100b-PublishLatency99thPct.png)[1topic-16partition-100b-PublishLatency99thPct.png](https://gist.github.com/cdbartholomew/8e9c8ef9bfe6943a1f7578efa6cfc603#file-1topic-16partition-100b-publishlatency99thpct-png) hosted with by [GitHub](https://github.com/)

![1topic-16partition-100b-End-to-endLatencyAvg.png](https://lh5.googleusercontent.com/gYdxhlH8U6i79XI3JYh53J9qoaXpTf6fadu9j7U0rCbol1XP-P1EQ82_X4em7v-04HvitfyVmC5cNFjS8UdjPpuJ8HvQxmJ1_kcKQSH3_pq0hQU1K_IFkWCB9DPr96w7MgSUQiBC)

[view raw](https://gist.github.com/cdbartholomew/8e9c8ef9bfe6943a1f7578efa6cfc603/raw/d1173492a68c2e70333a9626e162969c1354fc15/1topic-16partition-100b-End-to-endLatencyAvg.png)[1topic-16partition-100b-End-to-endLatencyAvg.png](https://gist.github.com/cdbartholomew/8e9c8ef9bfe6943a1f7578efa6cfc603#file-1topic-16partition-100b-end-to-endlatencyavg-png) hosted with by [GitHub](https://github.com/)

| Latency Type              | Average | 50th  | 75th  | 95th  | 99th   | 99.9th  | 99.99th | Maximum |
| ------------------------- | ------- | ----- | ----- | ----- | ------ | ------- | ------- | ------- |
| Publishing (milliseconds) | 8.479   | 8.152 | 8.781 | 9.635 | 10.656 | 169.557 | 211.642 | 234.369 |
| End-to-end (milliseconds) | 11.031  | 10.0  | 11.0  | 12.0  | 28.0   | 209.0   | 259.0   | 319.0   |

#### DISCUSSION

Looking first at the publishing latency with one partition, we see that on average Kafka without per-message flushing (2.19 ms) has lower latency than Pulsar whether Pulsar is flushing each message to disk or not ( 2.969 ms flush, 2.72 ms no flush). 

However, in the distribution of latency, we see a major difference between Pulsar and Kafka. While Pulsar has tight latency distribution all the way up to the 99.9th percentile (2.916 to 4.095 ms from 50th to 99.9th), Kafka latency spikes up to 149.616 ms at the 99.9th percentile. 

This is quite a difference. At the 99.99th percentile with one partition, Pulsar latency is 52.958 ms and Kafka is nearly four times higher at 201.701 ms. And here, we are comparing default modes, so Pulsar is flushing each message to disk, while Kafka is not. If you disable flushing to disk for Pulsar, the 99.99th latency drops to just 4.508 ms.

The reason for the large number of publishing latency outliers in Kafka seems pretty obvious when looking at the graph of 99th percentile publishing latency over time. Kafka shows periodic spikes where publishing latency jumps from single digits to over 100 ms. The effect is diminished as the number of partitions is increased., Bbut it is still present. 

Compare this to Pulsar, where the 99th percentile publishing latency is essentially a straight line for the entire duration of the test.

Another interesting difference between Pulsar and Kafka is that increasing the number of partitions lowers the publishing latency for Pulsar, but has the opposite effect for Kafka. Although the average publishing latency for Kafka is lower than Pulsar for the one1-partition test, Pulsar is lower for the six6- and 16-partition tests. For the 16-partition test, Pulsar gives under 3 ms of average publishing latency, while Kafka has nearly 8.5 ms.

Looking at the average end-to-end latency, Kafka beats Pulsar at the lowest partition count., bBut just likeas with the publishing latency, end-to-end latency increases with partition count, so that at 16-partitions, Kafka has an average end-to-end latency of 11 ms, while Pulsar is approaching 3 ms. 

With end-to-end latency over time, we saw a periodic 2 ms spike with Pulsar. With Kafka, we similarly see spikes, but they are more frequent and typically higher, often spiking over 5 ms.

### Latency with Flush

These tests are identical to the previous set, except that per-message flushing (fsync) is enabled. This was configured (flush.messages=1, flush.ms=0) for all topics used in the test.

#### TEST 10: 1 TOPIC, 1 PARTITION

You can get the command output and raw results [here](https://gist.github.com/cdbartholomew/d878d7c780a0b5a9d38e979eefa68513). Click on “view raw” to see a larger version of a graph.

![1topic-1partition-100b-flush-PublishLatency99thPct.png](https://lh3.googleusercontent.com/IqOk-3BW8eMrEhKnCw8_P6T1zVmNus8z5AXoMGmEIIvzRMxSzEnIuK3OB_SZOqVoAutj2w-T3vedRJFbv0oG9e3tqzieF2qq9u51vZFd0qW1ix-rZ9GUIaVADHV2YEFTmIUGJlcK)

[view raw](https://gist.github.com/cdbartholomew/d878d7c780a0b5a9d38e979eefa68513/raw/14e128ebaf6a5e59bab1001d7236a8a2ff18f4dd/1topic-1partition-100b-flush-PublishLatency99thPct.png)[1topic-1partition-100b-flush-PublishLatency99thPct.png](https://gist.github.com/cdbartholomew/d878d7c780a0b5a9d38e979eefa68513#file-1topic-1partition-100b-flush-publishlatency99thpct-png) hosted with by [GitHub](https://github.com/)

![1topic-1partition-100b-flush-End-to-endLatencyAvg.png](https://lh5.googleusercontent.com/OI6gG1aa1ZaEnXF6YHiYwTaORDVVu9Oge1So_GFtXJZEbSEjPeDzfczSrkrFmGBIjxfpigR77oy5Z0K-2HqUtYBhJtUq4bUDVUzd2Qn4BcfPW608qjg-Ewnf4zK82jNa67nVoKpQ)

[view raw](https://gist.github.com/cdbartholomew/d878d7c780a0b5a9d38e979eefa68513/raw/14e128ebaf6a5e59bab1001d7236a8a2ff18f4dd/1topic-1partition-100b-flush-End-to-endLatencyAvg.png)[1topic-1partition-100b-flush-End-to-endLatencyAvg.png](https://gist.github.com/cdbartholomew/d878d7c780a0b5a9d38e979eefa68513#file-1topic-1partition-100b-flush-end-to-endlatencyavg-png) hosted with by [GitHub](https://github.com/)

| Latency Type              | Average | 50th  | 75th  | 95th  | 99th  | 99.9th | 99.99th | Maximum |
| ------------------------- | ------- | ----- | ----- | ----- | ----- | ------ | ------- | ------- |
| Publishing (milliseconds) | 6.652   | 6.747 | 7.277 | 8.032 | 8.641 | 22.416 | 194.108 | 219.741 |
| End-to-end (milliseconds) | 7.129   | 7.0   | 7.0   | 8.0   | 9.0   | 170.0  | 210.0   | 243.0   |

#### TEST 11: 1 TOPIC, 6 PARTITIONS

You can get the command output and raw results [here](https://gist.github.com/cdbartholomew/223aec311fd4a1d5afbe9864cdd141f4). Click on “view raw” to see a larger version of a graph.

![1topic-6partition-100b-flush-PublishLatency99thPct.png](https://lh4.googleusercontent.com/aGV2kEOpUMO4B3WmVO8NFreWVH76Biyl4u7w764nyOLWmv2tdrsWYVWlnfdavqtu42bLXqKmXyhodkUgXadlLNaxA6_lcF7jc-On7gvSiIt5RdfwBJzUXgMAfNJVAXgzlpywo1Sj)

[view raw](https://gist.github.com/cdbartholomew/223aec311fd4a1d5afbe9864cdd141f4/raw/093969780692df4d7be93264230b0c912a45ae33/1topic-6partition-100b-flush-PublishLatency99thPct.png)[1topic-6partition-100b-flush-PublishLatency99thPct.png](https://gist.github.com/cdbartholomew/223aec311fd4a1d5afbe9864cdd141f4#file-1topic-6partition-100b-flush-publishlatency99thpct-png) hosted with by [GitHub](https://github.com/)

![1topic-6partition-100b-flush-End-to-endLatencyAvg.png](https://lh6.googleusercontent.com/hWhwVPIyUSompFVLwh89CWcHVkVZcb8jQ05NnB07d61h9KpQKbS4nySBbIA33U5bTYgpuWATGnlhwEGy3bYxHzGA8AJV4yQalgFP-oz4aGOm3WTggn1ow6E6ei3H1RINf3TGRQI3)

[view raw](https://gist.github.com/cdbartholomew/223aec311fd4a1d5afbe9864cdd141f4/raw/093969780692df4d7be93264230b0c912a45ae33/1topic-6partition-100b-flush-End-to-endLatencyAvg.png)[1topic-6partition-100b-flush-End-to-endLatencyAvg.png](https://gist.github.com/cdbartholomew/223aec311fd4a1d5afbe9864cdd141f4#file-1topic-6partition-100b-flush-end-to-endlatencyavg-png) hosted with by [GitHub](https://github.com/)

| Latency Type              | Average | 50th   | 75th  | 95th   | 99th   | 99.9th  | 99.99th | Maximum |
| ------------------------- | ------- | ------ | ----- | ------ | ------ | ------- | ------- | ------- |
| Publishing (milliseconds) | 11.125  | 10.823 | 11.33 | 12.081 | 13.517 | 132.025 | 212.062 | 225.853 |
| End-to-end (milliseconds) | 13.857  | 13.0   | 13.0  | 15.0   | 32.0   | 208.0   | 239.0   | 287.0   |

#### TEST 12: 1 TOPIC, 16 PARTITIONS

You can get the command output and raw results [here](https://gist.github.com/cdbartholomew/07993bf0e6b39098bf14012524ead381). Click on “view raw” to see a larger version of a graph.

![1topic-16partition-100b-flush-PublishLatency99thPct.png](https://lh4.googleusercontent.com/11iKm_10C86QkF2MTvSOogv__wYBhSPtelGqkgH-iiZhrmj1AXaTTNMRKb2CswJyPqouwvgKV40sgEldbc6YlIk5B8NXi9OxGGquDpsaN_bLw5vo0ar_byD70qr6n5CTOY829NUG)

[view raw](https://gist.github.com/cdbartholomew/07993bf0e6b39098bf14012524ead381/raw/0d80805a3e5eb2deacc0687156e9beb6b57751b9/1topic-16partition-100b-flush-PublishLatency99thPct.png)[1topic-16partition-100b-flush-PublishLatency99thPct.png](https://gist.github.com/cdbartholomew/07993bf0e6b39098bf14012524ead381#file-1topic-16partition-100b-flush-publishlatency99thpct-png) hosted with by [GitHub](https://github.com/)

![1topic-16partition-100b-flush-End-to-endLatencyAvg.png](https://lh3.googleusercontent.com/3iCw19R2swxxU7p_xZp3tfGnSQ526Wgu0wjvaSyKQFDeD5csFO1-7sBE8_Mj1wCjbDfKlXM0E5BWrCWgX7OyV2B2S8um4UmF1-RPvampRjP2Um7_ITP_tJr9wYYU9H5_0BkysOgB)

[view raw](https://gist.github.com/cdbartholomew/07993bf0e6b39098bf14012524ead381/raw/0d80805a3e5eb2deacc0687156e9beb6b57751b9/1topic-16partition-100b-flush-End-to-endLatencyAvg.png)[1topic-16partition-100b-flush-End-to-endLatencyAvg.png](https://gist.github.com/cdbartholomew/07993bf0e6b39098bf14012524ead381#file-1topic-16partition-100b-flush-end-to-endlatencyavg-png) hosted with by [GitHub](https://github.com/)

| Latency Type              | Average | 50th   | 75th   | 95th   | 99th   | 99.9th  | 99.99th | Maximum |
| ------------------------- | ------- | ------ | ------ | ------ | ------ | ------- | ------- | ------- |
| Publishing (milliseconds) | 18.454  | 17.935 | 19.815 | 22.075 | 23.404 | 123.801 | 222.137 | 290.615 |
| End-to-end (milliseconds) | 21.119  | 20.0   | 22.0   | 25.0   | 33.0   | 199.0   | 259.0   | 334.0   |

#### DISCUSSION

This set of tests are an apples-to-apples comparison to Pulsar’s default behavior of flushing each message to disk. In this comparison, Pulsar is clearly better. In the one-partition test where Kafka had an advantage over Pulsar when it wasn’t flushing to disk, when both systems are flushing to disk, Pulsar’s average latency is 2.969 ms while Kafka’s is more than double at 6.652 ms. Because adding partitions still increases latency for Kafka in these tests, the disparity grows even greater at 16-partitions where Pulsar is giving 2.72 ms of latency and Kafka is clocking in at 18.454 ms, which is six times higher.

The large publishing latency spikes still remain when Kafka is configured to flush each message to disk. This, however, happens less often.

For average end-to-latency, not surprisingly flushing to disk increases the latency for Kafka across the board. Kafka actually has an advantage in the one-partition case (7.129 vs 9.052 ms), but Pulsar is clearly better in the six- and 16-partitions cases. Looking at the end-to-end latency over time with Kafka, there are still periodic spikes as high as 5 ms.

## Wrapping up

Based on this set of results, we can draw the following conclusions:

- Pulsar gives more predictable latency over time. The graphs of latency over time are smoother with Pulsar than Kafka. This comparison chart (six-partition, average end-to-end latency, no flushing) shows one case where Kafka latency is actually lower than Pulsar. But the Pulsar plot is less variable:

 

![1topic-6partition-100b-noflush-End-to-endLatencyAvg.png](https://lh4.googleusercontent.com/2qWmpqm3lo777iS1s5GYM8SUZpHqtMo7JZND6WJ_T-QUVDpBMivIkz98aI7vS5DJkhigQvj5rcaoNLFYDq7Pghm-HC0qSsYaqpvcy2_jE91BddXbOb3ONwSHwred926sEEvO4R5U)

[view raw](https://gist.github.com/cdbartholomew/98b861c83e43020d15e2bf1c4e10ae95/raw/ab331568852d4c36a9643bb69f7919af8cfba4f0/1topic-6partition-100b-noflush-End-to-endLatencyAvg.png)[1topic-6partition-100b-noflush-End-to-endLatencyAvg.png](https://gist.github.com/cdbartholomew/98b861c83e43020d15e2bf1c4e10ae95#file-1topic-6partition-100b-noflush-end-to-endlatencyavg-png) hosted with by [GitHub](https://github.com/)

- Pulsar has more tightly bounded latency. Most of the Kafka tests show elevated latency at the 99.9th percentile. In the few cases where Pulsar shows elevated latency, it occurs at the 99.99th percentile. This comparison chart (six-partition, 99th publishing latency, flushing), clearly shows just how bounded Pulsar latency is compared to Kafka:

![1topic-6partition-100b-flush-PublishLatency99thPct.png](https://lh3.googleusercontent.com/rIktOrmownFeI5UTHWzvCL2nSIDICIJXBauvmxxM988x4RcfmbhD05cf6CVC2whErKOR82yjsELxZuvMhBS6IdD3tUFgZg5ZNEKtzeFskIsNxIWWAJZbA8ef9iFpPR1N71_-6_pJ)

[view raw](https://gist.github.com/cdbartholomew/98b861c83e43020d15e2bf1c4e10ae95/raw/ab331568852d4c36a9643bb69f7919af8cfba4f0/1topic-6partition-100b-flush-PublishLatency99thPct.png)[1topic-6partition-100b-flush-PublishLatency99thPct.png](https://gist.github.com/cdbartholomew/98b861c83e43020d15e2bf1c4e10ae95#file-1topic-6partition-100b-flush-publishlatency99thpct-png) hosted with by [GitHub](https://github.com/)

- Increasing the partition count for a Pulsar topic lowers latency when using a single producer and consumer. Increasing the partition count for a Kafka topic increases latency for a single producer and consumer.
- Given a need for the highest message durability, Pulsar provides lower latency than Kafka.
- Disabling flushing of messages to disk with Pulsar provides small latency gains and is not warranted given the durability tradeoff.

For latency-sensitive workloads, Pulsar is the overall winner. It is able to provide consistent, low latency as well as strong durability guarantees. 

Of course, not all workloads are latency-sensitive. Some may be willing to trade off latency for higher throughput. In a future post, we will be doing a similar comparison of the throughput performance between Apache Kafka and Apache Pulsar.

(Editor's note: [DataStax acquired Kesque in January 2021](https://www.datastax.com/press-release/datastax-delivers-scale-out-enterprise-event-streaming-modern-data-apps).)

**Want to try out Apache Pulsar?** [Sign up now](https://astra.datastax.com/register) for [Astra Streaming](https://www.datastax.com/products/astra-streaming), our fully managed Apache Pulsar service. We’ll give you access to its full capabilities entirely free through beta. See for yourself how easy it is to build modern data applications and let us know what you’d like to see to make your experience even better. 