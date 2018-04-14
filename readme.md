# 🐀 Ratlog - Application Logging for Rats, Humans and Machines

**Disclaimer**: *The Ratlog specification is still in draft state and might be subject to breaking changes. We try our best to publish a stable version as soon as possible. [Leave feedback](https://github.com/ratlog/ratlog.github.io/issues) and help us get there faster!*

----------------------------


Ratlog is a specification for an application logging format.

The format is language independent and libraries for producing and parsing logs are available in different programming languages.


----------------------------

- [Examples](#examples)
- [Goals](#goals)
- [Specification](#specification)
- [Library and Tooling Development](#library-and-tooling-development)
- [Questions](#questions)

----------------------------



## Examples

The most simple Ratlog line is an empty line. Even if no output is produced, the system should know that something happened:

```

```

All log lines should have a **message** describing the event:

```
System started
```

Logs can be scoped in context or categorized by adding **tags**:

```
[warn] Disk space running low
```

**Tags** can be chained and their order can carry meaning:

```
[fs|warn|critical] Disk space running low
```

Additional information can be added to logs using **fields**:

```
File not found | path: /tmp/notfound.txt
```

Putting everything together, a log line consist of **tags**, **message** and **fields**:

```
[http|request|error] File not found | code: 404 | method: GET | route: /admin
```

----------------------------

Logs should be easy to parse by programs and CLI tools such as `grep` and `cut`:

- Filter by tag:

```
cat ./logs.rat | grep '\[|\|warn'
```

- Filter by field:

```
cat ./logs.rat | grep '| code: 404'
```

- Count tag:

```
cat ./logs.rat | grep -c '\[|\|warn'
```

- Ignore fields:

```
cat ./logs.rat | cut -d'|' -f1
```

- Ignore tags:

```
cat ./logs.rat | cut -d] -f2-
```

----------------------------



## Goals

- Create a log format independent from any programming language or library.
- Logs should be readable by humans, without external tooling. JSON is not readable.
- Logs should be parsable by machines.
- Logging should never fail. There are no invalid logs.
- Provide semantics useful for the majority of application logging. More specific semantics can be built on top of Ratlog using tags and fields.
- Every log has a message, even if it is empty.
- Log levels are arbitrary and restricting. Use tags as flexible alternative instead.
- A log event is always a single line of text.
- Applications don't need to worry about logging timestamps.
- Applications don't need to know their own name.
- Play well with supervisor tools such as docker and systemd. Don't duplicate information they already collect such as service names and timestamps.

----------------------------



## Specification

A Ratlog line consists of the segements **tags**, **message** and **fields**, in this order.

### Encoding

- Text should be **UTF-8** encoded. If that is not possible, the reason and encoding must be clearly stated.


### Lines

- A log is always a single line ending with a unix line separator `\n`.
- Line breaks in the log data must be escaped as `\\n`.


### Spacing

- Leave no space before and after `[` surrounding tags.
- Leave no space before `]` surrounding tags.
- Leave a space after `]` surrounding tags.
- Leave no space before `:` in fields.
- Leave a space after `:` in fields.
- Leave no space around `|` between tags.
- Leave a space around `|` infront of fields.
- All other spaces are treated as data.


### Tags

- The tags segement must start at the beginning of the line.
- The tags segement starts with `[` and ends with `]`.
- If there is no matching closing token, there is no tags segement and
  the opening `[` and all following characters are treated as part of the *message* segment.
- Tags are separated by `|` without spaces.
- Inside the tags segment `]` can be escaped as `\]` and `|` as `\|`.
- An empty tag value is still a valid tag.
- The tags segment can be ommited. If no tags exist no `[]` is printed.
  `[]` represents a single empty tag.
- Tag values are not unique. They can be repeated.
- The sorting of tags *may* be user-defined and *may* be relevant.
- In terms of data structures tags, should be thought of as a *list* rather than a *set*.


### Message

- The message segement always exists, even if it is completely empty. It is never omitted.
- In the message segement `[` can be escaped as `\[` and `|` as `\|`.


### Fields

- The fields segement may be ommited if no fields exist.
- Each field starts with `|`.
- Field key and value are separate by `:` followed by a space.
- Inside a field `|` can be escaped as `\|` and `:` as `\:`.
- A completely empty field key or value is still valid.
- If a field's value is empty, the separator `: ` may be ommited.
- Field keys must be unique.
- If any field is invalid, the fields segment is ignored and all potential field characters, even those if valid fields, are treated as part of the *message* segement instead.
- The sorting of fields is not relevant and may be ignored.
- In terms of data structures fields should be thought of as a hashmap/dictionary rather than a list.

----------------------------



## Library and Tooling Development

We always welcome new developers to contribute new tooling to the Ratlog ecosystem.
If you would like to create or extend a logging library in a programming language of your choice,
you can use the *[JSON test suite](./ratloc.testsuite.json)* to verify your logger or parser is producing or consuming logs correctly:

- The test suite contains an array of test cases.
- Each test case is an object with `"log"` and `"data"` fields.
- `"log"` is a string containing a log line.
- `"data"` is an object with `"message"`, `"fields"`, `"tags"` and `"initialTags"` fields. All but `"message are optional"`.
- `"message"` is a string containing the logged message.
- `"fields"` is an object with arbitrary fields mapping strings to strings.
- `"tags"` and `"initialTags"` are arrays of arbitrary strings. `"tags"` should be with each logging and `"initialTags"` should be already set when instanciating the logger. This way `"initialTags"` always show up before the `"tags"` in the produced log lne.

When writing a logging library, use `"data"` as input and verify that the produced line matches `"log"`.
Since all parameters here are well defined and all values are represented as strings, make sure your logging library is also handling other input passed to it, which - depending of the strictness of the type system of the language you are working with - might have all sorts of shapes. Keep in mind [GX](TODO) - logging should never crash a program.

When writing a parser `"log"` should be parsed and the fields specified in `"data"` should be retrieved from it.

----------------------------



## Questions

### Are there not enough logging libraries already?

There are enough libraries. They all have different goals and functionality. Some are simple, some are quiet sofisticated.
But hardly any two of them produce matching formats - or formatting is not one of their concerns at all and they leave it up to the user.

The closest to a common format on which a few libraries agree on is JSON.
But JSON doesn't know about logging. Each developer has to come up with their own JSON format.
And JSON is not made for humans. You need additional tooling to query and read logs in a sane way.


### Why doesn't Ratlog come with timestamps?

Timestamps are a fundamental part of evented data such as application logs.

However in the context applications are embedded in a supervisor that is collecting the logs most likely already adds timestamp anyways.
Therefore there is no need to duplicate the information.

If you have a [Docker](https://docker.com/) setup and you run `docker logs -t myapp`, you get logs displayed with timestamps:

```
2018-03-29T11:10:29.116Z [file-import|warning] file not found | path: /tmp/notfound.txt | code: 404
```

Similarly when using Systemd `journalctl -u myapp` includes timestamps in its output:

```
Apr 13 22:15:34 myhost myapp[1234]: [file-import|warning] file not found | path: /tmp/notfound.txt | code: 404
```

Running a standalone application `ts` can add timestamps easily:

```
$ ./myapp | ts
Apr 14 12:03:38 [file-import|warning] file not found | path: /tmp/notfound.txt | code: 404
```

And in case you really need the application to log timestamps directly, nothing is stopping you from adding a *field* for it.


### What about [Common Log Format](https://en.wikipedia.org/wiki/Common_Log_Format)?

There are some standardized logging formats for specific use cases such as [Common Log Format](https://en.wikipedia.org/wiki/Common_Log_Format)  for server logs, which is very useful if you are building an HTTP server.

If you have a more specific output format to use for your domain, you can provide more meaningful context which is great.
Ratlog tries to be a foundation for generic application logging.

If you build a web server, use Common Log Format.


### What about [logfmt](https://brandur.org/logfmt)?

[logfmt](https://brandur.org/logfmt) describes logs as key-value pairs.
It is simpler and (arguably) easier to read than JSON.
It is a great start for adding some structure to log output.
The key-value semantics of logfmt are similar to *fields* in Ratlog,
but Ratlog additionally gives you the semantics of *message* and *tags*, which I would argue are generally useful for logging.
If you rather only have *field* semantics, logfmt is a great and even simpler alternative to Ratlog.


### How do I add Ratlog to my library/module/package?

Libraries shouldn't produce logs.
They should return machine readable information about their state to the application code making use of the library.
The application should be responsible for deciding on what and how to log the state.


### Should I use Ratlog for my CLI tool?

Certain CLI tools produce generic output similar to application logs.
In those scenarios Ratlog might be a great way of displaying consistent, readable, parsable output.

Many tools, however, produce very specific output.
For example, it wouldn't be helpful if `git` would try to display its information as Ratlog.

Mostly the output tools write to *stdout* is too specific and not log-like.
But tools sometimes like to provide additional information about their state via *stderr*.
That might be a good usecase of Ratlog.


### How do I collect metrics from Ratlog logs?

Logs are for humans. Metrics, events and other output are better collected separately because there are tools more suited for them for them (such as [Prometheus](https://prometheus.io/) for metrics). Metrics should be handled different to text based logs. Metrics can be aggregated and have very different performance characteristics and usage goals.


### How do I log errors?

Errors should not be logged. Other tools are better suited at handling them (such as [Sentry](https://github.com/getsentry/sentry)).
Tools specialized on errors can provide many useful features such as grouping, silencing or linking stacktraces to source code.
Those features are not in the scope of logging.


### How do I query logs across many services?

Ratlog is just text.
You can collect logs from many services in a central place like you are already used to with tools such as
[fluentd](https://www.fluentd.org/),
[Elastic](https://www.elastic.co/elk-stack),
[CloudWatch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html) or
[OK Log](https://github.com/oklog/oklog).


### Why are there no nested fields or lists inside fields?

Ratlog tries to be as simple and generic as possible.
If you have need to store more complex data, custom formats can be built on top of Ratlog.
It is also possible to store data in a format such as JSON inside a field.
However, we recommend keeping fields simple.
That makes the logs easier to read an query.
You also don't want to log more than absolutely necessary and especially don't write sensitive user data inside logs.


### Why use `[]` and `|` as separators?

They conflict little with natural language or other popular formats such as JSON or XML/HTML,
while at the same time being common enough that you should be able to locate them on your keyboard.

----------------------------



## Licsense

[MIT](./license)
