

{% summary %}About failure detection, reducing failure detection time, and relevant parameters.{% endsummary %}

# Overview

Failure detection is the time it takes for the space and the client to detect that failure has occurred. Failure detection consists of two main phases:

1. The backup space detects that the primary space is down, and takes over as primary.
1. The client detects that the machine running the primary space is down. In case it is running against a clustered space, it routs its requests to the new primary space (the backup space that has just taken over as primary).

{% info %}
If the client is running against a single primary space, a disconnection exception is thrown, and the client cannot proceed.
{%endinfo%}

One of two main failure scenarios might occur:

- Process failure or machine crash
- Network cable disconnection

It takes GigaSpaces a few seconds to recover from process failure or a machine crash. In case of network cable disconnection, the client first has to detect that it has been disconnected from the machine running the space. Therefore, recovery time in this case is longer. For details on how network failure is detected and handled, see the [Communication Protocol]({%currentjavaurl%}/communication-protocol.html#watchdog) section.

# Reducing Failure Detection Time

Configuring failure detection time can help you handle extreme failure scenarios more effectively. For example, in extreme cases of network disconnection, you might want the failover process to take 2-3 seconds.

Here is a good combination for the space settings you may use to reduce the failover time - these should be used with a fast network:

{% highlight java %}
cluster-config.groups.group.fail-over-policy.active-election.yield-time=300
cluster-config.groups.group.fail-over-policy.active-election.fault-detector.invocation-delay=300
cluster-config.groups.group.fail-over-policy.active-election.fault-detector.retry-count=2
{% endhighlight %}

The following should be specified as system properties:

{% highlight java %}
-Dcom.gs.transport_protocol.lrmi.connect_timeout=3
-Dcom.gs.transport_protocol.lrmi.request_timeout=3
-Dcom.gs.jini.config.maxLeaseDuration=2000
{% endhighlight %}

By default, the maximum time it takes for a backup space to switch into a primary mode takes ~6-8 seconds.
If you would like to reduce the failover time , you should use the following formula:

{% highlight java %}
100 [ms] + (yield-time * 7) + invocation-delay + (retry-count * retry-timeout) = failover time
{% endhighlight %}

- The 100 ms above refers a constant that is related to the network latency. You should reduce this number for a fast network.

Change the default settings **only if you have a special need** to reduce the failover duration. (~1 second). In this case:

- the `yield-time` minimum value should be 200 ms.
- Reducing the `invocation-delay` and `retry-timeout` values, and increasing the `retry-count` values might increase the chatting overhead between the spaces and the lookup service.

{% note %}
For additional tuning options please contact the [GigaSpaces Support Team](http://www.gigaspaces.com/supportcenter).
{% endnote %}

# Failure Detection Parameters

{% toczone minLevel=2|maxLevel=2|type=flat|separator=pipe|location=top %}

## Space Side Parameters

### Active Election Parameters

The following parameters in the cluster schema [active election](#Active Election and Avoiding Split-Brain Scenarios) block regard failure detection and recovery:

{: .table .table-bordered}
|Parameter|Parameter Description|Default Value| Unit |
|:--------|:--------------------|:------------|:-----|
| `cluster-config.groups.group.fail-over-policy.active-election.yield-time` |This parameter allows you to configure the time it takes to yield to other participants between every election phase. | `1000` | millisec |
| `cluster-config.groups.group.fail-over-policy.active-election.fault-detector.invocation-delay` |This parameter limits the amount of time the backup space waits between each ping to the primary space. | `1000`| millisec |
| `cluster-config.groups.group.fail-over-policy.active-election.fault-detector.retry-count` |Related to the `cluster-config.groups.group.fail-over-policy.active-election.fault-detector.invocation-delay` parameter, defines the number of times the backup checks if the primary space has failed | `3`|   |
| `cluster-config.groups.group.fail-over-policy.active-election.fault-detector.retry-timeout` |Related to the `retry-count` parameter, defines the time between retries the backup checks if the primary space has failed | `100`| millisec |
| `cluster-config.groups.group.fail-over-policy.active-election.connection-retries` | Defines the number of times the space instance will try to establish connection with the lookup service. The wait time in between retries is defined by the `yield-time` parameter | `60` | |

## Client Side Parameters

### Proxy Connectivity

The [Proxy Connectivity]({%currentjavaurl%}/proxy-connectivity.html) defines the settings in which the system checks the liveness of space members.

### Watchdog Parameters

See the `com.gs.transport_protocol.lrmi.idle_connection_timeout` and the `com.gs.transport_protocol.lrmi.request_timeout` watchdog properties at the [Communication Protocol]({%currentjavaurl%}/communication-protocol.html#watchdog) section.

## Service Grid Parameters

The Service Grid uses two complementary mechanisms for service detections -- the Lookup Service and fault-detection handlers.

- `GSMFaultDetectionHandler` -- used by GSMs to monitor each other.
- `GSCFaultDetectionHandler` -- used by the GigaSpaces Management Center to monitor GSCs.
- `PUFaultDetectionHandler` -- Used by GSMs to monitor Processing Units deployed on GSCs.

The fault-detection handlers check periodically if a service is alive, and in case of failure, how many times to retry and how often.

The GSM and GSC fault-detection handler settings are controlled via the relevant properties. The `PUFaultDetectionHandler` is configurable using the [SLA - member alive indicator]({%currentjavaurl%}/configuring-the-processing-unit-sla.html#livenessDetection).

For logging information, it is advised to monitor service failure by setting the logging level to `Level.FINE`.

{% highlight java %}
# ServiceGrid FaultDetectionHandler logging

com.gigaspaces.grid.gsc.GSCFaultDetectionHandler.level = INFO
com.gigaspaces.grid.gsm.GSMFaultDetectionHandler.level = INFO
org.openspaces.pu.container.servicegrid.PUFaultDetectionHandler.level = INFO
{% endhighlight %}

## Jini Lookup Service Parameters

The `LeaseRenewalManager` in the `advanced-space.config` file is also related to failure detection and recovery:

{: .table .table-bordered}
|Parameter|Parameter Description|Default Value|
|:--------|:--------------------|:------------|
| `maxLeaseDuration` | The time the system waits between every lease renewal, for example: if the parameter value is `8000`, the system renews the space lease every 8000 `\[ms\]`.{% wbr %}{% infosign %} As this value is reduced, renewal requests are performed more frequently while the service is up, and lease expiration occurs sooner when the service goes down. | `8000` |
| `roundTripTime` | This parameter instructs the renewal process to begin a certain amount of time (by default, 100 `\[ms\]`) before the actual renewal time, thus making sure that the renewal process is successful.{% wbr %}{% exclamation %} Significantly low values might result in failure to renew a lease. Durations of managed leases should exceed the `roundTripTime`. | `4000` |

{% endtoczone %}

## Lookup Service Unicast discovery parameters

When a Jini Lookup Service fails and is brought back online, a client (such as a GSC, space or a client with a space proxy) needs to re-discover it. It uses Jini unicast discovery retrying to connect to the failed remote lookup service. The default unicast retry protocol provides a graduating approach, increasing the amount of time to wait before the next discovery attempts are made - upon each invocation, eventually reaching a maximum time interval over which discovery is re-tried. In this way, the network is not flooded with unicast discovery requests referencing a lookup service that may not be available for quite some time (if ever).

The downside is that it may delay the discovery of services if these are not brought up quickly. A discovery can be delayed us much as 15 minutes. If you have two GSMs and one fails, but it will be brought back up only in the next hour, then it's discovery will take ~15 minutes after it has loaded.

These settings can be configured - see [How to Configure Unicast Discovery](./how-to-configure-unicast-discovery.html).
