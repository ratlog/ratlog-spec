# 🐀 Ratlog - Application Logging for Humans and Machines

**Disclaimer**: *The Ratlog specification is still in draft state and might be subject to breaking changes. We try our best to publish a stable version as soon as possible.*


Ratlog is an attempt to create a well defined specification of an application logging format.

The format is language independent and libraries for producing and parsing logs are available in different programming languages.


## Examples

The most simple Ratlog line is an empty line. Even if no output is produced, the system should know that something happened:

```

```

All log lines should have a *message* describing the event. A log line can consist of simply a *message*:

```
System started
```

Logs can be scoped in context or categorized by adding *tags*:

```
[warn] Disk space running low
```

*Tags* can be chained and their order can carry meaning:

```
[fs|warn|critical] Disk space running low
```

Additional information can be added to logs using *fields*:

```
File not found | path: /tmp/notfound.txt
```

Putting everything together, logs consist of *tags*, *message* and *fields*:

```
[http|request|error] File not found | code: 404 | method: GET | route: /admin
```


## Motivation

### Are there not enough logging libraries already?

There are enough libraries. They all have different goals and functionality. Some are simple, some are quiet sofisticated.
But hardly any two of them produce matching formats - or formatting is not one of their concerns at all and they leave it up to the user.

The closest to a common format on which a few libraries agree on is JSON.
But JSON doesn't know about logging. Each developer has to come up with their own JSON format.
And JSON is not made for humans. You need additional tooling to query and read logs in a sane way.



The supervisor of a service has a name for a service.
Services themselves don't need to know their names.

The supervisor already knows how to timestamp logs.
Docker, Systemd+journalctl, syslog, service | ts

Logs are for humans.
Events, metrics and other output are collected separately because there are better tools for them (Prometheus, ...).

Errors should not be logged. Other tools are better suited at handling them (Sentry, ...)

Log levels are arbitrary and restricting. Use tags instead.

All logging fields should be optional. Logging should never crash your application.

Logs should try to leave a message for humans

Logs should be easy to read for humans.

No external cli to read logs

JSON is hard to read.


Logging should be simple: message+fields+tags

Logging is for applications. Applications consist of components.
Components are stateful units of a system.
A component doesn't need to know about it's scope but it should return information about its state
consisting of messages plus, optionally, fields and tags.


Modules/libraries/packages shouldn't produce logs.
Tools have their custom output format but can write logs to stderr.
Applications/Services write logs to stdout.
  stderr is for errors (if they are not handled externally) and errors are not logs, they are language specific and should contain stacktraces.

Logs are always a single line.

Play well with docker, docker compose and journalctl.



## Goals

- Log format independent from any programming language or library
- Readable by humans, without external tooling
- Parsable by machines
- Structured in message, tags and fields
- Logs should never fail. There are no invalid logs.
- Every log has a message.
- Applications don't need to worry about logging timestamps.
- Applications don't need to know their own name.




Logs should be easy to parse by programs and cli tools (such as grep and cut):

Find warnings:

    tail logs | grep '| warning'

Find by field:

    tail logs | grep '| code: 404'

Filter by scope:

    tail logs | grep '\[file'

Count lines of scope:

    tail logs | grep -c '\[file-import\]'

ignore fields:

    tail logs | cut -d'|' -f1


```
connect | 2018-03-29T11:10:29.116Z [file-import|warning] file not found | path: /tmp/notfound.txt | code: 404
connect | 2018-03-29T11:10:29.116Z [file-import|warning] file not found
connect | 2018-03-29T11:10:29.116Z [file-import|warning]
connect | 2018-03-29T11:10:29.116Z [file-import] | path: /tmp/notfound.txt | code: 404
connect | 2018-03-29T11:10:29.116Z [file-import | warning ] | path: /tmp/notfound.txt | code: 404
connect | 2018-03-29T11:10:29.116Z | path: /tmp/notfound.txt | code: 404
connect | file not found
connect | 2018-03-29T11:10:29.116Z file not found
connect | [file-import] file not found | path: /tmp/notfound.txt | code: 404
connect | file not found | path: /tmp/notfound.txt | code: 404
[file-import] file not found | path: /tmp/notfound.txt | code: 404
file not found | path: /tmp/notfound.txt | code: 404
| path: /tmp/notfound.txt | code: 404
file not found | path: /tmp/notfound.txt
file not found
[ ] file not found | path: /tmp/notfound.txt | code: 404
[file-import]  | path: /tmp/notfound.txt | code: 404
[ ]  | path: /tmp/notfound.txt | code: 404
 | path: /tmp/notfound.txt

|________________________________| |_______________| |______________| |___________________________________|
                  |                         |                |                           |
      this part is from docker        dashed-tags        message         additional fields (no nesting)
```



## Spec Description

Text should be utf-8 encoded. If that is not possible, it must b e clearly stated.

a log is always a single line ending with a unix line separator \n
line breaks in the log must be escaped as \\n.

### spacing

- leave no space before and after [ surrounding the tags
- leave no space before ] surrounding the tags
- leave a space after ] surrounding the tags
- leave no space before : in fields
- leave a space after : in fields
- leave no space around | between tags
- leave a space around | between fields
- all other spaces are treated as part of values

### tags

  tags are surrounded by [].
  in a tag ] must be escaped as \] and | as \|.
  tags are separated by | without spaces.
  if no tags exist no [] is printed.
  tags can be repeated.
  the sorting of tags may be user defined and may be relevant.
  in terms of data structures tags should be thought of as a list rather than a set.

### message

  in the message [ must be escaped as \[ and | must be escaped as \|.

### fields

  each field starts with a |.
  field key and value are separate by :.
  inside a field | must be escaped as \| and : as \:.
  the sorting of fields may be user defined and may be relevant, but it is not recommended to do so.
  field keys may be duplicated, but it is not recommended.
  In terms of data structures fields should be thought of as a Hashmap/Dictionary rather than a list.


## Examples


## Logger and Parser Development

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




Parsing:

``` js
const [service, timestamp, scope, message, ...fieldString] = log.split(/\s*[^\\]\|\s*/)
const fields = fieldString.reduce((fs, s) => {
  if (!s) return fs
  const [k, ...vs] = s.split(':')
  return {...fs, [k]: vs.join(':')}
}, {})
```



docker logs -t
grep
tail -f


### timestamps

not recommended since supervisor collects them already.
but can be put in a field if necessary
or pipe through a tool like `ts`


why | pipe not , or something else?
- conflicts little with natural language and other formats such as JSON


### Questions

#### What about [Common Log Format](https://en.wikipedia.org/wiki/Common_Log_Format)?

There are some standardized logging formats for specific use cases such as the  for server logs, which is very useful if you are building an HTTP server. But for generic services and applications we have other requirements.

#### What about [logfmt](https://brandur.org/logfmt)?


#### How to query logs across many services?

Want to centralize logs? Easily collect with typical systems such as elastic or fluentd or use the simpler https://github.com/oklog/oklog





## Licsense

[MIT](./license)
