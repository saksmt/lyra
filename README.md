# Lyra

*Fault tolerant RabbitMQ client*

## Introduction

Dealing with failure is a fact of life in distributed systems. Lyra is a [RabbitMQ](http://www.rabbitmq.com/) client that embraces failure, allowing for AMQP resources such as connections, channels and consumers to be automatically recovered when server or network failures occur.

#### Features

* Automatic resource recovery
* Automatic invocation retries
* Channel pooling
* Event listeners

## Setup

`mvn install`

## Usage

### Resource Recovery

The key feature of Lyra is its ability to automatically recover resources such as [Connections](http://www.rabbitmq.com/releases/rabbitmq-java-client/current-javadoc/com/rabbitmq/client/Connection.html), [Channels](http://www.rabbitmq.com/releases/rabbitmq-java-client/current-javadoc/com/rabbitmq/client/Channel.html) and [Consumers](http://www.rabbitmq.com/releases/rabbitmq-java-client/current-javadoc/com/rabbitmq/client/Consumer.html) when unexpected failures occur. For example:

```java
LyraOptions options = LyraOptions.forHost("localhost")
	.withRecoveryPolicy(new RetryPolicy()
		.withMaxRetries(100)
		.withInterval(Duration.seconds(1))
		.withMaxDuration(Duration.minutes(5)));

Connection connection = Connections.create(options);
Channel channel = connection.createChannel();
channel.basicConsume("foo-queue", myConsumer);
```

Here we've created a new `Connection` and `Channel`, specifying a recovery policy to use in case any of our resources are *unexpectedly* closed. If the consumer is cancelled, the channel is closed, or the connection is closed, Lyra will automatically attempt to recover the closed resources according to the given recovery policy.

### Invocation Retries

Lyra also supports invocation retries when a *retryable* failure occurs while creating a connection or invoking a method against a [Connection](http://www.rabbitmq.com/releases/rabbitmq-java-client/current-javadoc/com/rabbitmq/client/Connection.html) or [Channel](http://www.rabbitmq.com/releases/rabbitmq-java-client/current-javadoc/com/rabbitmq/client/Channel.html). For example:

```java
LyraOptions options = LyraOptions.forHost("localhost")
	.withRecoveryPolicy(RetryPolicies.retryAlways())
	.withRetryPolicy(new RetryPolicy()
		.withBackoff(Duration.seconds(1), Duration.seconds(30))
		.withMaxDuration(Duration.minutes(10)));
		
Connection connection = Connections.create(options);
Channel channel = connection.createChannel();
channel.basicConsume("foo-queue", myConsumer);
```

Here again we've created a new `Connection` and `Channel`, specifying a recovery policy to use in case any of our resources are *unexpectedly* closed as a result of an invocation failure, and a retry policy that dictates how and when the failed method invocation should be retried. If the `Connections.create()`, `connection.createChannel()`, or `channel.basicConsume()` method invocations fail as the result of a *retryable* error, Lyra will recover any resources that were closed according to the recovery policy and retry the invocation according to the retry policy.

### Channel Pooling

Lyra supports channel pooling to avoid the expense of creating/closing channels. When enabled, channel pooling allows for generic (non-numbered) channels to be recycled. To enable, simply set a channel pool site:

```java
LyraOptions.forHost("localhost")
	.withChannelPoolSize(12);
```

### Event Listeners

Lyra offers listeners for creation and recovery events:

```java
LyraOptions.forHost("localhost")
	.withConnectionListeners(myConnectionListener)
	.withChannelListeners(myChannelListener)
	.withConsumerListeners(myConsumerListener);
```

## Additional Notes

### On Recovery / Retry Policies

Recovery/Retry policies allow you to specify:

* The maximum number of retry attempts to perform
* The maxmimum duration that retries should be performed for
* The standard interval between retry attempts
* The maximum interval between retries to exponentially backoff to

Lyra allows for Recovery/Retry policies to be set globally or for individual resource types, as well as for initial connection attempts.

### On Retryable Failures

Lyra will only retry failed invocations that are deemed *retryable*. These include connection errors that are not related to failed authentication, and channel or connection errors that might be the result of temporary network failures.

### On Threading

When a Connection or Channel are closed as the result of an invocation, the calling thread is blocked according to the recovery policy until the Connection/Channel can be recovered. Connection recovery occurs in a background thread while Channel recovery occurs in the invoking thread. When a Connection is closed involuntarily (not as the result of an invocation), recovery occurs in a background thread.

## License

Copyright 2013 Jonathan Halterman - Released under the [Apache 2.0 license](http://www.apache.org/licenses/LICENSE-2.0.html).