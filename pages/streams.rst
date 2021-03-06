*******
Streams
*******

What are streams?
*****************

The Graylog streams are a mechanism to route messages into categories in realtime while they are processed. You define rules that
instruct Graylog which message to route into which streams. Imagine sending these three messages to Graylog::

  message: INSERT failed (out of disk space)
  level: 3 (error)
  source: database-host-1

  message: Added user 'foo'.
  level: 6 (informational)
  source: database-host-2

  message: smtp ERR: remote closed the connection
  level: 3 (error)
  source: application-x

One of the many things that you could do with streams is creating a stream called *Database errors* that is catching every error
message from one of your database hosts.

Create a new stream with these rules: (stream rules are AND connected)

* Field ``level`` must be greater than ``4``
* Field ``source`` must match regular expression ``^database-host-\d+``

This will route every new message with a ``level`` higher than *WARN* and a ``source`` that matches the database host regular
expression into the stream.

A message will be routed into every stream that has all its rules matching. This means that a message can be part of many streams
and not just one.

The stream is now appearing in the streams list and a click on its title will show you all database errors.

The next parts of this document cover how to be alerted in case of too many errors, some specific error types that should never
happen or how to forward the errors to another system or endpoint.

What's the difference to saved searches?
========================================

The biggest difference is that streams are processed in realtime. This allows realtime alerting and forwarding to other systems.
Imagine forwarding your database errors to another system or writing them to a file by regularly reading them from the message
storage. Realtime streams do this much better.

Another difference is that searches for complex stream rule sets are always comparably cheap to perform because a message is
*tagged* with stream IDs when processed. A search for Graylog internally always looks like this, no matter how many stream
rules you have configured::

  streams:[STREAM_ID]

Building a query with all rules would cause significantly higher load on the message storage.

How do I create a stream?
=========================

#. Navigate to the streams section from the top navigation bar
#. Click "Create stream"
#. Save the stream after entering a name and a description. For example *All error messages* and
   *Catching all error messages from all sources*
#. The stream is now saved but **not yet activated**. Add stream rules in the next dialogue. Try it against some messages by
   entering a message ID on the same page. Save the rules when the right messages are matched or not matched.
#. The stream is marked as *paused* in the list of streams. Activate the stream by hitting *Resume this stream* in the *Action*
   dropdown.

Alerts
******

You can define conditions that trigger alerts. For example whenever the stream *All production exceptions* has more than 50
messages per minute or when the field *milliseconds* had a too high standard deviation in the last five minutes.

Hit *Manage alerts* in the stream *Action* dropdown to see already configured alerts, alerts that were fired in the past or
to configure new alert conditions.

Graylog ships with default *alert callbacks* and can be extended with
`plugins <https://www.graylog.org/resources/documentation/general/plugins>`_

What is the difference between alert callbacks and alert receivers?
===================================================================

There are two type of actions to be triggered when an alert is fired: Alert callbacks or an email to a list of alert
receivers.

Alert callbacks are single actions that are just called once. For example: The *Email Alert Callback* is triggering
an email to exactly one receiver and the *HTTP Alert Callback* is calling a HTTP endpoint once.

The alert receivers in difference will all receive an email about the same alert.

Outputs
*******

The stream output system allows you to forward every message that is routed into a stream to other destinations.

Outputs are managed globally (like message inputs) and not for single streams. You can create new outputs and activate them
for as many streams as you like. This way you can configure a forwarding destination once and select multiple streams to use it.

Graylog ships with default outputs and can be extended with
`plugins <http://www.graylog.org/resources/documentation/general/plugins>`_.

Use cases
*********

These are a few example use cases for streams:

* Forward a subset of messages to other data analysis or BI systems to reduce their license costs.
* Monitor exception or error rates in your whole environment and broken down per subsystem.
* Get a list of all failed SSH logins and use the *quickvalues* to analyze which user names where affected.
* Catch all HTTP POST requests to ``/login`` that were answered with a HTTP 302 and route them into a stream called
  *Successful user logins*. Now get a chart of when users logged in and use the *quickvalues* to get a list of users that performed
  the most logins in the search time frame.

How are streams processed internally?
*************************************

The most important thing to know about Graylog stream matching is that there is no duplication of stored messages. Every message that comes
in is matched against all rules of a stream. The internal ID of every stream that has *all* rules matching is appended to the ``streams``
array of the processed message.

All analysis methods and searches that are bound to streams can now easily narrow their operation by searching with a
``streams:[STREAM_ID]`` limit. This is done automatically by Graylog and does not have to be provided by the user.

.. image:: /images/internal_stream_processing.png

Stream Processing Runtime Limits
********************************

An important step during the processing of a message is the stream classification. Every message is matched against the user-configured
stream rules. If every rule of a stream matches, the message is added to this stream. Applying stream rules is done during the indexing
of a message only, so the amount of time spent for the classification of a message is crucial for the overall performance and message
throughput the system can handle.

There are certain scenarios when a stream rule takes very long to match. When this happens for a number of messages, message processing
can stall, messages waiting for processing accumulate in memory and the whole system could become non-responsive. Messages are lost and
manual intervention would be necessary. This is the worst case scenario.

To prevent this, the runtime of stream rule matching is limited. When it is taking longer than the configured runtime limit, the process
of matching this exact message against the rules of this specific stream is aborted. Message processing in general and for this specific
message continues though. As the runtime limit needs to be configured pretty high (usually a magnitude higher as a regular stream rule
match takes), any excess of it is considered a fault and is recorded for this stream. If the number of recorded faults for a single stream
is higher than a configured threshold, the stream rule set of this stream is considered faulty and the stream is disabled. This is done
to protect the overall stability and performance of message processing. Obviously, this is a tradeoff and based on the assumption, that
the total loss of one or more messages is worse than a loss of stream classification for these.

There are scenarios where this might not be applicable or even detrimental. If there is a high fluctuation of the message load including
situations where the message load is much higher than the system can handle, overall stream matching can take longer than the configured
timeout. If this happens repeatedly, all streams get disabled. This is a clear indicator that your system is overutilized and not able
to handle the peak message load.

How to configure the timeout values if the defaults do not match
================================================================

There are two configuration variables in the configuration file of the server, which influence the behavior of this functionality.

* ``stream_processing_timeout`` defines the maximum amount of time the rules of a stream are able to spend. When this is exceeded, stream
  rule matching for this stream is aborted and a fault is recorded. This setting is defined in milliseconds, the default is ``2000`` (2 seconds).
* ``stream_processing_max_faults`` is the maximum number of times a single stream can exceed this runtime limit. When it happens more often,
  the stream is disabled until it is manually reenabled. The default for this setting is ``3``.

What could cause it?
====================

If a single stream has been disabled and all others are doing well, the chances are high that one or more stream rules are performing bad under
certain circumstances. In most cases, this is related to stream rules which are utilizing regular expressions. For most other stream rules types
the general runtime is constant, while it varies very much for regular expressions, influenced by the regular expression itself and the input
matched against it. In some special cases, the difference between a match and a non-match of a regular expression can be in the order of 100
or even 1000. This is caused by a phenomenon called *catastrophic backtracking*. There are good write-ups about it on the web which will help
you understanding it.

Summary: How do I solve it?
===========================

#. Check the rules of the stream that is disabled for rules that could take very long (especially regular expressions).
#. Modify or delete those stream rules.
#. Re-enable the stream.

Programmatic access via the REST API
************************************

Many organisations already run monitoring infrastructure that are able to alert operations staff when incidents are detected.
These systems are often capable of either polling for information on a regular schedule or being pushed new alerts - this article describes how to
use the Graylog Stream Alert API to poll for currently active alerts in order to further process them in third party products.

Checking for currently active alert/triggered conditions
========================================================

Graylog stream alerts can currently be configured to send emails when one or more of the associated alert conditions evaluate to true. While
sending email solves many immediate problems when it comes to alerting, it can be helpful to gain programmatic access to the currently active alerts.

Each stream which has alerts configured also has a list of active alerts, which can potentially be empty if there were no alerts so far.
Using the stream's ID, one can check the current state of the alert conditions associated with the stream using the authenticated API call::

  GET /streams/<streamid>/alerts/check

It returns a description of the configured conditions as well as a count of how many triggered the alert. This data can be used to for example
send SNMP traps in other parts of the monitoring system.

Sample JSON return value::

  {
    "total_triggered": 0,
    "results": [
      {
        "condition": {
          "id": "984d04d5-1791-4500-a17e-cd9621cc2ea7",
          "in_grace": false,
          "created_at": "2014-06-11T12:42:50.312Z",
          "parameters": {
            "field": "one_minute_rate",
            "grace": 1,
            "time": 1,
            "backlog": 0,
            "threshold_type": "lower",
            "type": "mean",
            "threshold": 1
          },
          "creator_user_id": "admin",
          "type": "field_value"
        },
        "triggered": false
      }
    ],
    "calculated_at": "2014-06-12T13:44:20.704Z"
  }

Note that the result is cached for 30 seconds.

List of already triggered stream alerts
=======================================

Checking the current state of a stream's alerts can be useful to trigger alarms in other monitoring systems, but if one wants to send more detailed
messages to operations, it can be very helpful to get more information about the current state of the stream, for example the list of all triggered
alerts since a certain timestamp.

This information is available per stream using the call::

  GET /streams/<streamid>/alerts?since=1402460923

The since parameter is a unix timestamp value. Its return value could be::

  {
    "total": 1,
    "alerts": [
      {
        "id": "539878473004e72240a5c829",
        "condition_id": "984d04d5-1791-4500-a17e-cd9621cc2ea7",
        "condition_parameters": {
          "field": "one_minute_rate",
          "grace": 1,
          "time": 1,
          "backlog": 0,
          "threshold_type": "lower",
          "type": "mean",
          "threshold": 1
        },
        "description": "Field one_minute_rate had a mean of 0.0 in the last 1 minutes with trigger condition lower than 1.0. (Current grace time: 1 minutes)",
        "triggered_at": "2014-06-11T15:39:51.780Z",
        "stream_id": "53984d8630042acb39c79f84"
      }
    ]
  }

Using this information more detailed messages can be produced, since the response contains more detailed information about the nature of the
alert, as well as the number of alerts triggered since the timestamp provided.

Note that currently a maximum of 300 alerts will be returned.

FAQs
****

Using regular expressions for stream matching
=============================================

Stream rules support matching field values using regular expressions.
Graylog uses the `Java Pattern class <http://docs.oracle.com/javase/7/docs/api/java/util/regex/Pattern.html>`_ to execute regular expressions.

For the individual elements of regular expression syntax, please refer to Oracle's documentation, however the syntax largely follows the familiar
regular expression languages in widespread use today and will be familiar to most.

However, one key question that is often raised is matching a string in case insensitive manner. Java regular expressions are case sensitive by
default. Certain flags, such as the one to ignore case sensitivity can either be set in the code, or as an inline flag in the regular expression.

To for example route every message that matches the browser name in the following user agent string::

    Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.107 Safari/537.36

the regular expression ``.*applewebkit.*`` will not match because it is case sensitive.
In order to match the expression using any combination of upper- and lowercase characters use the ``(?i)`` flag as such::

    (?i).*applewebkit.*

Most of the other flags supported by Java are rarely used in the context of matching stream rules or extractors, but if you need them their use
is documented on the same Javadoc page by Oracle.

Can I add messages to a stream after they were processed and stored?
====================================================================

No. Currently there is no way to re-process or re-match messages into streams.

Only new messages are routed into the current set of streams.

Can I write own outputs or alert callbacks methods?
===================================================

Yes. Please refer to the `plugins <http://www.graylog.org/resources/documentation/general/plugins>`_ documentation page.
