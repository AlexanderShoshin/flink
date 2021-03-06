---
title: "Failure & Recovery Model"
nav-id: fault_tolerance
nav-parent_id: setup
nav-pos: 4
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

Flink's fault tolerance mechanism recovers programs in the presence of failures and
continues to execute them. Such failures include machine hardware failures, network failures,
transient program failures, etc.

* This will be replaced by the TOC
{:toc}


Streaming Fault Tolerance
-------------------------

Flink has a checkpointing mechanism that recovers streaming jobs after failures. The checkpointing mechanism requires a *persistent* (or *durable*) source that
can be asked for prior records again (Apache Kafka is a good example of such a source).

The checkpointing mechanism stores the progress in the data sources and data sinks, the state of windows, as well as the user-defined state (see [Working with State]({{ site.baseurl }}/dev/state.html)) consistently to provide *exactly once* processing semantics. Where the checkpoints are stored (e.g., JobManager memory, file system, database) depends on the configured [state backend]({{ site.baseurl }}/dev/state_backends.html).

The [docs on streaming fault tolerance]({{ site.baseurl }}/internals/stream_checkpointing.html) describe in detail the technique behind Flink's streaming fault tolerance mechanism.

To enable checkpointing, call `enableCheckpointing(n)` on the `StreamExecutionEnvironment`, where *n* is the checkpoint interval in milliseconds.

Other parameters for checkpointing include:

- *exactly-once vs. at-least-once*: You can optionally pass a mode to the `enableCheckpointing(n)` method to choose between the two guarantee levels.
  Exactly-once is preferrable for most applications. At-least-once may be relevant for certain super-low-latency (consistently few milliseconds) applications.

- *number of concurrent checkpoints*: By default, the system will not trigger another checkpoint while one is still in progress. This ensures that the topology does not spend too much time on checkpoints and not make progress with processing the streams. It is possible to allow for multiple overlapping checkpoints, which is interesting for pipelines that have a certain processing delay (for example because the functions call external services that need some time to respond) but that still want to do very frequent checkpoints (100s of milliseconds) to re-process very little upon failures.

- *checkpoint timeout*: The time after which a checkpoint-in-progress is aborted, if it did not complete by then.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// start a checkpoint every 1000 ms
env.enableCheckpointing(1000);

// advanced options:

// set mode to exactly-once (this is the default)
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);

// checkpoints have to complete within one minute, or are discarded
env.getCheckpointConfig().setCheckpointTimeout(60000);

// allow only one checkpoint to be in progress at the same time
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()

// start a checkpoint every 1000 ms
env.enableCheckpointing(1000)

// advanced options:

// set mode to exactly-once (this is the default)
env.getCheckpointConfig.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE)

// checkpoints have to complete within one minute, or are discarded
env.getCheckpointConfig.setCheckpointTimeout(60000)

// allow only one checkpoint to be in progress at the same time
env.getCheckpointConfig.setMaxConcurrentCheckpoints(1)
{% endhighlight %}
</div>
</div>

{% top %}

### Externalized Checkpoints

You can configure periodic checkpoints to be persisted externally. Externalized checkpoints write their meta data out to persistent storage and are *not* automatically cleaned up when the job fails. This way, you will have a checkpoint around to resume from if your job fails.

```java
CheckpoingConfig config = env.getCheckpointConfig();
config.enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
```

The `ExternalizedCheckpointCleanup` mode configures what happens with externalized checkpoints when you cancel the job:

- **`ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION`**: Retain the externalized checkpoint when the job is cancelled. Note that you have to manually clean up the checkpoint state after cancellation in this case.

- **`ExternalizedCheckpointCleanup.DELETE_ON_CANCELLATION`**: Delete the externalized checkpoint when the job is cancelled. The checkpoint state will only be available if the job fails.

The **target directory** for the checkpoint is determined from the default checkpoint directory configuration. This is configured via the configuration key `state.checkpoints.dir`, which should point to the desired target directory:

```
state.checkpoints.dir: hdfs:///checkpoints/
```

This directory will then contain the checkpoint meta data required to restore the checkpoint. The actual checkpoint files will still be available in their configured directory. You currently can only set this via the configuration files.

Follow the [savepoint guide]({{ site.baseurl }}/setup/cli.html#savepoints) when you want to resume from a specific checkpoint.

### Fault Tolerance Guarantees of Data Sources and Sinks

Flink can guarantee exactly-once state updates to user-defined state only when the source participates in the
snapshotting mechanism. The following table lists the state update guarantees of Flink coupled with the bundled connectors.

Please read the documentation of each connector to understand the details of the fault tolerance guarantees.

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 25%">Source</th>
      <th class="text-left" style="width: 25%">Guarantees</th>
      <th class="text-left">Notes</th>
    </tr>
   </thead>
   <tbody>
        <tr>
            <td>Apache Kafka</td>
            <td>exactly once</td>
            <td>Use the appropriate Kafka connector for your version</td>
        </tr>
        <tr>
            <td>AWS Kinesis Streams</td>
            <td>exactly once</td>
            <td></td>
        </tr>
        <tr>
            <td>RabbitMQ</td>
            <td>at most once (v 0.10) / exactly once (v 1.0) </td>
            <td></td>
        </tr>
        <tr>
            <td>Twitter Streaming API</td>
            <td>at most once</td>
            <td></td>
        </tr>
        <tr>
            <td>Collections</td>
            <td>exactly once</td>
            <td></td>
        </tr>
        <tr>
            <td>Files</td>
            <td>exactly once</td>
            <td></td>
        </tr>
        <tr>
            <td>Sockets</td>
            <td>at most once</td>
            <td></td>
        </tr>
  </tbody>
</table>

To guarantee end-to-end exactly-once record delivery (in addition to exactly-once state semantics), the data sink needs
to take part in the checkpointing mechanism. The following table lists the delivery guarantees (assuming exactly-once
state updates) of Flink coupled with bundled sinks:

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 25%">Sink</th>
      <th class="text-left" style="width: 25%">Guarantees</th>
      <th class="text-left">Notes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td>HDFS rolling sink</td>
        <td>exactly once</td>
        <td>Implementation depends on Hadoop version</td>
    </tr>
    <tr>
        <td>Elasticsearch</td>
        <td>at least once</td>
        <td></td>
    </tr>
    <tr>
        <td>Kafka producer</td>
        <td>at least once</td>
        <td></td>
    </tr>
    <tr>
        <td>Cassandra sink</td>
        <td>at least once / exactly once</td>
        <td>exactly once only for idempotent updates</td>
    </tr>
    <tr>
        <td>AWS Kinesis Streams</td>
        <td>at least once</td>
        <td></td>
    </tr>
    <tr>
        <td>File sinks</td>
        <td>at least once</td>
        <td></td>
    </tr>
    <tr>
        <td>Socket sinks</td>
        <td>at least once</td>
        <td></td>
    </tr>
    <tr>
        <td>Standard output</td>
        <td>at least once</td>
        <td></td>
    </tr>
    <tr>
        <td>Redis sink</td>
        <td>at least once</td>
        <td></td>
    </tr>
  </tbody>
</table>

{% top %}

## Restart Strategies

Flink supports different restart strategies which control how the jobs are restarted in case of a failure.
The cluster can be started with a default restart strategy which is always used when no job specific restart strategy has been defined.
In case that the job is submitted with a restart strategy, this strategy overrides the cluster's default setting.

The default restart strategy is set via Flink's configuration file `flink-conf.yaml`.
The configuration parameter *restart-strategy* defines which strategy is taken.
Per default, the no-restart strategy is used.
When checkpointing is activated and no restart strategy has been configured, the job will be restarted infinitely often.
See the following list of available restart strategies to learn what values are supported.

Each restart strategy comes with its own set of parameters which control its behaviour.
These values are also set in the configuration file.
The description of each restart strategy contains more information about the respective configuration values.

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 50%">Restart Strategy</th>
      <th class="text-left">Value for restart-strategy</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td>Fixed delay</td>
        <td>fixed-delay</td>
    </tr>
    <tr>
        <td>Failure rate</td>
        <td>failure-rate</td>
    </tr>
    <tr>
        <td>No restart</td>
        <td>none</td>
    </tr>
  </tbody>
</table>

Apart from defining a default restart strategy, it is possible to define for each Flink job a specific restart strategy.
This restart strategy is set programmatically by calling the `setRestartStrategy` method on the `ExecutionEnvironment`.
Note that this also works for the `StreamExecutionEnvironment`.

The following example shows how we can set a fixed delay restart strategy for our job.
In case of a failure the system tries to restart the job 3 times and waits 10 seconds in-between successive restart attempts.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // number of restart attempts
  Time.of(10, TimeUnit.SECONDS) // delay
));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // number of restart attempts
  Time.of(10, TimeUnit.SECONDS) // delay
))
{% endhighlight %}
</div>
</div>

{% top %}

### Fixed Delay Restart Strategy

The fixed delay restart strategy attempts a given number of times to restart the job.
If the maximum number of attempts is exceeded, the job eventually fails.
In-between two consecutive restart attempts, the restart strategy waits a fixed amount of time.

This strategy is enabled as default by setting the following configuration parameter in `flink-conf.yaml`.

~~~
restart-strategy: fixed-delay
~~~

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Configuration Parameter</th>
      <th class="text-left" style="width: 40%">Description</th>
      <th class="text-left">Default Value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td><it>restart-strategy.fixed-delay.attempts</it></td>
        <td>The number of times that Flink retries the execution before the job is declared as failed</td>
        <td>1</td>
    </tr>
    <tr>
        <td><it>restart-strategy.fixed-delay.delay</it></td>
        <td>Delay between two consecutive restart attempts. Delaying the retry means that after a failed execution, the re-execution does not start immediately, but only after a certain delay. Delaying the retries can be helpful when the program interacts with external systems where for example connections or pending transactions should reach a timeout before re-execution is attempted.</td>
        <td><it>akka.ask.timeout</it></td>
    </tr>
  </tbody>
</table>

~~~
restart-strategy.fixed-delay.attempts: 3
restart-strategy.fixed-delay.delay: 10 s
~~~

The fixed delay restart strategy can also be set programmatically:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // number of restart attempts
  Time.of(10, TimeUnit.SECONDS) // delay
));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // number of restart attempts
  Time.of(10, TimeUnit.SECONDS) // delay
))
{% endhighlight %}
</div>
</div>

{% top %}

### Failure Rate Restart Strategy

The failure rate restart strategy restarts job after failure, but when `failure rate` (failures per time interval) is exceeded, the job eventually fails.
In-between two consecutive restart attempts, the restart strategy waits a fixed amount of time.

This strategy is enabled as default by setting the following configuration parameter in `flink-conf.yaml`.

~~~
restart-strategy: failure-rate
~~~

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Configuration Parameter</th>
      <th class="text-left" style="width: 40%">Description</th>
      <th class="text-left">Default Value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td><it>restart-strategy.failure-rate.max-failures-per-interval</it></td>
        <td>Maximum number of restarts in given time interval before failing a job</td>
        <td>1</td>
    </tr>
    <tr>
        <td><it>restart-strategy.failure-rate.failure-rate-interval</it></td>
        <td>Time interval for measuring failure rate.</td>
        <td>1 minute</td>
    </tr>
    <tr>
        <td><it>restart-strategy.failure-rate.delay</it></td>
        <td>Delay between two consecutive restart attempts</td>
        <td><it>akka.ask.timeout</it></td>
    </tr>
  </tbody>
</table>

~~~
restart-strategy.failure-rate.max-failures-per-interval: 3
restart-strategy.failure-rate.failure-rate-interval: 5 min
restart-strategy.failure-rate.delay: 10 s
~~~

The failure rate restart strategy can also be set programmatically:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.failureRateRestart(
  3, // max failures per interval
  Time.of(5, TimeUnit.MINUTES), //time interval for measuring failure rate
  Time.of(10, TimeUnit.SECONDS) // delay
));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.failureRateRestart(
  3, // max failures per unit
  Time.of(5, TimeUnit.MINUTES), //time interval for measuring failure rate
  Time.of(10, TimeUnit.SECONDS) // delay
))
{% endhighlight %}
</div>
</div>

{% top %}

### No Restart Strategy

The job fails directly and no restart is attempted.

~~~
restart-strategy: none
~~~

The no restart strategy can also be set programmatically:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.noRestart());
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.noRestart())
{% endhighlight %}
</div>
</div>

### Fallback Restart Strategy

The cluster defined restart strategy is used. 
This helpful for streaming programs which enable checkpointing.
Per default, a fixed delay restart strategy is chosen if there is no other restart strategy defined.

{% top %}
