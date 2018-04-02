# 🐀 ratlog - application logging for humans and machines

ratlog is an attempt to create a well defined specification of a common logging format.

The format is language independent and libraries for producing and parsing logs are available in different programming languages.


## Are there not enough logging libraries already?

There are enough libraries. They all have different goals and functionality. Some are simple, some are quiet sofisticated.
But hardly any two of them produce matching formats - or formatting is not one of their concerns at all and they leave it up to the user.

The closest to a common format on which a few libraries agree on is JSON.
But JSON doesn't know about logging. Each developer has to come up with their own JSON format.
And JSON is not made for humans. You need additional tooling to query and read logs.


ratlog is a format independent of any specific logging library.



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
ts
grep
tail -f



js

    log('message', { field: 'value' }, 'tag')
    const fileLog = log.tag('file')


example

    log('app starting')

    const counterLog = log.tag('counter')

    const counter = startCounter({
      max: 2,
      > log: log.tag('counter')
      log: (msg, fields = {}, ...tags) => {
        if (tags.includes('event')) {
          countMetric(fields.count)
        }
        counterLog(msg, fields, tags...)
      }
      done: () => {
        log('app shutting down')
      }
    })

    log('app ready')

    const startCounter = ({ max = 0, interval = 3000, log, done }) => {
      log('starting')

      const count = (x = 1) => {
        if (x > max) {
          log('stopped')
          done()
          return
        }

        log('counting', { count: x }, 'event')

        setTimeout(count, interval, x+1)
      }
      count()

      log('started')
    }



log(string, { [string]: string }, ...string)

first param is message
if second is object it is used for fields
if second is string

Show example how to do application with components:

    app starting
    [counter] started
    [counter|event] counting | count: 1
    [counter|event] counting | count: 2
    [counter] stopped
    app ready
    app shutting down



why | pipe not , or something else?
- conflicts little with natural language and other formats such as JSON


js lib
  link to ratlog.github.io
  beta warning
  inline docs
  generate docs
  readme
  example


There are some standardized logging formats for specific use cases such as the [Common Log Format](https://en.wikipedia.org/wiki/Common_Log_Format) for server logs, which is very useful if you are building an HTTP server. But for generic services and applications we have other requirements.

