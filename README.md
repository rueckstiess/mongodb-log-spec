# MongoDB Log Parsing
#### Version 0.3.0

This document is a specification **draft** for parsing and processing MongoDB server logs. The goal is to have a unified schema that various tools can implement and understand.

This document assumes the latest MongoDB version (currently 2.8.0-rc11) unless stated otherwise. 

## General Information about this document

There are two types of applications or tools that should implement this spec: Log parsers (they produce output conforming to this spec) and log processors, including visualization tools (they expect input conforming to this spec). This spec guarantees that any processor can understand the data of any parser. It defines which values need to be extracted, how they need to be formatted, and under which name they can be retrieved.

The intermediate data format of this spec (output for parsers, input for processors) is [MongoDB's extended JSON](http://docs.mongodb.org/manual/reference/mongodb-extended-json/) format in **strict mode** (referred to as "eJSON" below). This format complies with actual [JSON](http://json.org/) but adds a number of conventions to represent types like Dates, ObjectIds, etc. It can also easily be imported into a MongoDB instance with [mongoimport](http://docs.mongodb.org/manual/reference/program/mongoimport/#bin.mongoimport).

### Taxonomy and Syntax

This document uses the same taxonomy and terms used by the [JSON](http://json.org/) standard. Individual log lines are mapped to a JSON object containing name/value pairs. These pairs are also called members.

The spec uses monospaced font for names and values, e.g. `component` and `JOURNAL`. Placeholders and variables also use monospaced font and are wrapped in angle brackets, e.g. `<counters>`. Everything in proportional font is considered an explanation, annotation or comment.

### Required and Optional Members

Any member of the schema description below is considered required unless stated as optional. If a value for an optional member is not available, the member should not be included in the result object at all. Required members should always have a defined value and be included in the result object.


## Information from individual MongoDB log lines

In version 3.0 and above, every log line follows this pattern: 

```
<timestamp> <severity> <component> [<context>] <message>
```

Previous versions did not have a `severity` or `component`, they instead followed this pattern:

```
<timestamp> [<context>] <message>
```

### Timestamps

The timestamp is at the beginning of every log line. There are 4 timestamp formats that need to be supported for full backwards compatibility. 

| Timestamp Format Name | Example                        | MongoDB Version |
|-----------------------|--------------------------------|-----------------|
| ctime-no-ms           | `Wed Dec 31 19:00:00`          | < 2.4           |
| ctime                 | `Wed Dec 31 19:00:00.000`      | 2.4             |
| iso8601-utc           | `1970-01-01T00:00:00.000Z`     | >= 2.6          |
| iso8601-local         | `1969-12-31T19:00:00.000+0500` | >= 2.6          |

In MongoDB 2.6 and above the timestamp format for logging [can be configured](http://docs.mongodb.org/master/reference/configuration-options/#systemLog.timeStampFormat) to any of the last 3 formats (`ctime-no-ms` is not available). The default is `iso8601-local`.

`ctime` and `ctime-no-ms` do not have year information. For those cases, the current year should be assumed when parsing individual lines. When parsing a whole file, and a rollover from December to January is detected, it should be assumed the log file ended this year, and adjustments to previous messages should be made.

Note that `ctime` and `ctime-no-ms` have whitespace in the timestamp. Extra care has to be taken when tokenizing lines on whitespace separators.

| name                 | short name   | example                                  | description                       |
|----------------------|--------------|------------------------------------------|-----------------------------------|
| `timestamp`          | `ts`         | `{"$date": "1969-12-31T19:00:00.000Z"}`  | eJSON timestamp converted to UTC  |
| `timestamp_format`   | `tsf`        | `ctime-no-ms`                            | original format name as above     |
                              

### Severity

Beginning with version 3.0, log lines include a severity in each line, see [documentation](http://docs.mongodb.org/master/reference/log-messages/). Possible values are these: 

- `D` (debug)
- `I` (informational)
- `W` (warning)
- `E` (error)
- `F` (fatal)

The `severity` is a single letter following the timestamp (separated with a space). 

| name                 | short name   | example                                  | description                       |
|----------------------|--------------|------------------------------------------|-----------------------------------|
| `severity`          | `sev`         | `W`                                      | Optional                          |


### Component

Beginning with version 3.0, log lines include a component in each line, see [documentation](http://docs.mongodb.org/master/reference/log-messages/#components) and [implementation](https://github.com/mongodb/mongo/blob/master/src/mongo/logger/log_component.cpp#L138-L151). Possible values are these 14 strings: 

`ACCESS`, `COMMAND`, `CONTROL`, `GEO`, `INDEX`, `NETWORK`, `QUERY`, `REPL`, `SHARDING`, `STORAGE`, `JOURNAL`, `WRITE`, `TOTAL`, `-`

The component in the log line is an exactly 8 character long string (shorter words have padding at the end) following the severity, separated by a space.

| name                 | short name   | example                                  | description                       |
|----------------------|--------------|------------------------------------------|-----------------------------------|
| `component`          | `cmp`         | `REPL`                                  | One of the above mentioned, strip whitespace, e.g. "`REPL`" but not "`REPL    `" (quotes are for illustration only, do not include)<br><br>Optional                   |


### Context

The `context` follows the `component`. It is surrounded by square brackets. Most log lines print the current thread/connection as context, for example `[conn12345]`. Other values are possible however, for example `[initandlisten]`.

| name                 | short name   | example                                  | description                       |
|----------------------|--------------|------------------------------------------|-----------------------------------|
| `context`            | `ctx`        | `conn53`                                 | remove square brackets.           |


### Message

Additionally, the message part of a log line should be kept, regardless if the message is parsed further or not.

| name                 | short name   | example                                  | description                       |
|----------------------|--------------|------------------------------------------|-----------------------------------|
| `message`            | `msg`        | `end connection 192.168.248.69:45613 (65 connections now open)`  |           |


### Operations

A subset of log messages are considered "operations", classified as one of `query`, `getmore`, `insert`, `update`, `remove`, `command`. 

The `message` part of an operation follows a common pattern that can be parsed further. The format for these operations is:

```
<operation> <namespace> <op-specific> <counters> <locks> <duration>ms
```

All members listed in this section should only be included in the result object for operations, and should be absent for non-operation log lines.

#### General Operation Members

For all operation messages, we can extract the type of operation, the namespace and the duration. Possible values are:

`query`, `getmore`, `insert`, `update`, `remove`, `command`

| name                 | short name   | example                                  | description                       |
|----------------------|--------------|------------------------------------------|-----------------------------------|
| `operation`          | `op`         | `update`                                 |                                   |
| `namespace`          | `ns`         | `test.mycoll`                            | namespace in dot-notation         |
| `duration`           | `dur`         | `6513`                                  | numeric value in ms               |

