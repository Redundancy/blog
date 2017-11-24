+++
date = "2017-07-12T14:46:39Z"
title = "Logs, Metrics and Events"
description = "Don't just deal with logs"

tags = ["logging"]
categories = ["engineering"]
+++
Logging is an area of engineering that isn't really discussed enough.

At some point early during our careers, someone shows us how to debug
stuff with print statements, console.log or one of the other variations and we are
set. At a later point, someone shows us centralized logging with all our logs going
to ELK or Splunk, and now we're able to do the same sort of thing with loads of
servers at once.

Our power is now defined by cost - how expensive it is to log something, how much
we log, how much we can throw at processing or searching them, how much we can afford to store.
All our problems have solutions, as long as you have enough power and a regex.

## The problem with logs

Once people get in the habit of using regexes to extract information from our logs
we will usually start to see a problem follow soon after - there's no guarantees with
logs. As engineers, our logs are just whatever we think is useful to see what happened at a later point.
We make no guarantees about them staying the same.

{{< highlight python>}}
log.info("Sending email failed")
{{< /highlight >}}

The line above is a perfectly reasonable line of log, at least as far as it goes.

You can almost imagine someone creating an alert that will trigger if it fails to send emails more than
a few times based off this log message, probably with an expectation that the email server might be down.
However, while working on a bug, the developer realizes that they're not checking for errors from the send function.
The obvious thing is to add that to the log, so that they can see a bit more about what went wrong, perhaps
because at one time they had an issue where the email server was up but it was failing to send.

{{< highlight python>}}
log.info("Sending email returned error code", err_code)
{{< /highlight >}}

However, if we were looking for "Sending email failed", we'll no longer get it, but there's nothing in the code
that indicates that there was something important that depended on it.

If we get a bit too enthusiastic with our logging, we might end up in the position where this logging
could have a performance impact, and perhaps we increase the log level to only send on warning logs.
Unfortunately, because we decided to make our log an info level log, we no longer get it,
and won't get any associated alerts either.

## Events
Here's a silly rule - if there's another system that consumes it or depends on it,
it's a message or an event, regardless of where it goes to.

Messages and Events are like API or RPC calls, the parameters matter, they should
be well defined in what they mean or do and they are functionality, not a debugging
tool. Sending events should not look like logging, they should look like function calls.
The removal or change of an event is understood to have potential effects on other systems.

Events don't have a level like logs do - since functionality may depend on them,
they MUST get through to the consumers at some SLA, and you may have to guarantee that
they are received.

This also suggests that events shouldn't go through the logging system, since we
can't always guarantee the same SLA for all logs.

### Errors / Exceptions
Errors should be treated like events, and routed to a system that can provide more
insight into them than is easily available in logs.

While things like Python exception tracebacks are nice, they are nothing like as
useful as something like Sentry, reporting exceptions, callstacks, local variables,
categorizing them and counting them.

Logs are easily ignored, but can become clogged with errors that are expected,
creating a level of noise that obscured new issues being raised and slowing debugging.
Many products that I've looked at over the years have built up a steady state of noise,
and it almost always results in tense sessions where you have to see if the volume of errors
looks like it did a week or a month ago, or if it's an indication of a change.

Don't use exceptions, or their equivalent as a way of reporting things like bad user input,
REST responses or other external failures.

## Metrics
Metrics are something else that I often see mixed in with, or derived from logs.

Metrics deserve to be treated seperately, because they are exceedingly useful for
tracking the behaviour of your application and alerting, and as such are a form
of contract to your monitoring. Metrics are also significantly cheaper to handle
than extracting information from logs after the fact too.

Like Events, metrics are something you don't want to get turned off just because
you disabled some logging. Measure your response times by your endpoint, count
your response codes, your db query times and your cache hit rates and do it always.

## General Advice
### Stdout and Stderr
Stdout is for output - as in program output that's input to something else.
Stderr is for information (not just errors)

Python will get this right by default - logging.StreamHandler will output to
stderr as will the standard logger in Go.

The standard comes from dealing with CLI applications and others that pipe output
to each other.

### Logs and partitions
Rotating logs is a great practice to prevent machines from filling up, but it's
worth considering further:

* How bad is it if I lose logs?
* Am I at risk of filling the disks if I lose connection?
* Can certain failure cases cause me to fill my logs far faster than usually?
* If I do fill the disk with logs, can I cause failures in more critical functionality?

While it might sound paranoid, it's not entirely unreasonable to ensure that if you are
logging to disk, you create a limited size partition especially for it. This practice
can prevent an issue with your logs from preventing you from staging new versions
of your application, or otherwise affecting how your application functions.

### Readable or Structured logs?
Structured logging involves outputting logs in a format that has structure -
frequently JSON, allowing you to preserve multiple fields of information together
without the need to extract them.

Tools like ELK and Splunk can really come into their own with well structured data,
especially where you standardize field names. This can really help you collect all
logs for a user action, even when they happen asynchronously across many machines.

Structured logging feels like the price of entry of any sufficiently complex,
async and distributed application where time ordering may interleave different contexts.

Beware when you use structured logs where you should be using events.

### BI
Business intelligence (BI) is one of the more common ways to come across events.
You're often reporting something that happened, or that a user did, in a way that
has a significant value to the business.

There are all sorts of things that you might need to consider that might differ:

* Long term compatibility
* Loss of events

Generating events that are easy to transform and correlate is possibly your largest concern,
and be very aware that you rarely go back and compensate for missing information.
Secondly, you need to consider that events from launch may still be relevant years into your
product life-cycle, and design accordingly in how you evolve.

### 12 Factor
https://12factor.net/logs

You may notice that a lot of what I'm saying is at odds with a 12 factor app setup -
you're going to have to make your own mind up about it, but I'm fairly convinced
that the simple reading of 12 factor logging ends up being bad practice.

Firstly, 12FA treats says you should send everything to stdout, breaking with
existing standards.

By putting everything into a single stream, you create problems for yourself in
extracting and enforcing different SLAs and create a need for something
like Fluentd.

For my money, save yourself the work of rebuilding information that you already had,
and the time and cost of dealing with frequently performance-hungry bandaids.

*Do* decouple yourself from the consumers.
