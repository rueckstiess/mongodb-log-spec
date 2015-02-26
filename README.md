# MongoDB Log Parsing Spec
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
| `component`          | `cmp`         | `REPL`                                  | One of the above mentioned, strip whitespace, e.g. "REPL" but not "REPL&nbsp;&nbsp;&nbsp;&nbsp;" (quotes are for illustration only, do not include)<br><br>Optional                   |


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

For all operation messages, we can extract the type of operation, the namespace and the duration. Possible values for the type of operation are: `query`, `getmore`, `insert`, `update`, `remove`, `command`

| name                 | short name   | example                                  | description                       |
|----------------------|--------------|------------------------------------------|-----------------------------------|
| `operation`          | `op`         | `update`                                 |                                   |
| `namespace`          | `ns`         | `test.mycoll`                            | namespace in dot-notation         |
| `duration`           | `dur`         | `6513`                                  | numeric value in ms               |


#### Operation-Specific Members

##### Query

The message part of a query contains `query`, `sort`, `planSummary` and `cursorid` information. The `query` information can take 3 forms due to some compatibility issues: 

``` 
query: { a: "foo" }
query: { $query: { a: "foo" } }
query: { query: { a: "foo" } }
```


All of these alternatives need to be reduced to the value `{ "a": "foo" }`. 

Note that if the user uses the third form to query for data, and a top-level document key is actually called "query", then it's possible to see this form in the log line: 

```
query: { query: { query: { a: "foo" } } }
```

Here, only the left two occurrences of "query:" should be removed, and the value is: `{ "query" : { "a" : "foo" } }`


Other keys may be present inside the outer query object, e.g. when the result is sorted, again with multiple alternatives: 

```
query: { query: { a: "foo" }, orderby: { _id: 1.0 } }
query: { $query: { a: "foo" }, $orderby: { _id: 1.0 } }
```

Values for planSummary can also take on multiple different forms. Here some examples: 

```
planSummary: IXSCAN { user_id: 1 }
planSummary: IXSCAN { name.first: 1.0 }, IXSCAN { name.last: 1.0 }
planSummary: COLLSCAN
planSummary: COLLSCAN, COLLSCAN
planSummary: COUNT { foo: 1.0, bar: 1.0 }
planSummary: IDHACK
planSummary: DISTINCT { foo: 1 }
planSummary: EOF
...
```

The entire string, including potentially multiple plans and/or index specs, should be extracted.

Queries returning more documents than fit in a batch, maintain a cursor on the server. These queries have a `cursorid` that is a numeric value.

| name                 | short name   | example                                  | description                       |
|----------------------|--------------|------------------------------------------|-----------------------------------|
| `query`              | `q`          | `{ "key": { "nested": "value" } }`       | JSON object that represents the query, including all subfields. The `{ query : … }` or `{ $query :  … }` wrapper (if present) should not be included. <br><br>Optional. |                   
| `namespace`          | `ns`         | `test.mycoll`                            | namespace in dot-notation         |
| `duration`           | `dur`         | `6513`                                  | numeric value in ms               |


##### Getmore

getmore operations only contain `cursorid` in their operation-specific members.

##### Insert

Inserts can have a `query` member (upserts), see above, but otherwise have no extra information to parse. If the insert was not an upsert, `query` should be empty.

##### Update

Updates have a `query` member and additionally an update member. 


| name                 | short name   | example                                  | description                       |
|----------------------|--------------|------------------------------------------|-----------------------------------|
| `update`             | `u`          | `{ "$set" { "foo" : "bar" } }`           | JSON object that represents the update, including a potential operator at the start (here: `$set`).<br><br>Optional. |


##### Remove

Remove operations have a `query` member.

##### Command

In MongoDB 2.6 and above, commands take the following form:

```
command <database>.$cmd command: <name> <command document> ...
```

In 2.4 and below, commands do not list the `<name>` separately (the name is still present in the command document).

```
command <database>.$cmd command: <command document> ...
```

| name                 | short name   | example                                  | description                       |
|----------------------|--------------|------------------------------------------|-----------------------------------|
| `command`            | `c`          | `replSetGetStatus`                       | Name of the command. For 2.4 and below, the name has to be extracted from the `command_doc` as first key.<br><br>Optional.   |
| `command_doc`        | `cd`         | `{ "count": 1, "query": { "foo": "bar" } }`    | JSON object that represents the command.<br><br>Optional. |


##### Counters

Following the operation-specific information, MongoDB logs certain counters as key/value pairs. All counters that follow the format <name>:<value> should be extracted. One exception is `numYields`. Before version 2.6, it had an extra space after the colon. Both versions must be supported.

All counters are optional. 

| name                 | short name   | example                                  | description                       |
|----------------------|--------------|------------------------------------------|-----------------------------------|
| `ntoreturn`          | `lim`        | `20`                                     | Optional                          | 
| `ntoskip`            | `skp`        | `5`                                      | Optional                          | 
| `nreturned`          | `n`          | `101`                                    | Optional                          | 
| `nscanned`           | `nsc`        | `1503433`                                | Optional                          | 
| `nscannedObjects`    | `nso`        | `1000`                                   | Optional                          | 
| `numYields`          | `ny`         | `1`                                      | versions before 2.6 had a space between the colon and value, e.g. `numYields: 7`. Both versions need to be correctly parsed.                         | 
| `keyUpdates`         | `ku`         | `1`                                      | Optional                          | 
| `writeConflicts`     | `wc`         | `0`                                      | Optional                          |
| `ninserted`          | `ni`         | `100`                                    | Optional                          | 
| `nMatched`           | `nma`        | `55`                                     | Optional                          | 
| `nModified`          | `nmo`        | `42`                                     | Optional                          | 
| `ndeleted`           | `nd`         | `16`                                     | Optional                          | 

##### Locks

Locks are logged after the counters, following this format: 

```
locks(micros) <type>:<value>
```

Here `<type>` is either `w` (for write locks) or `r` (for read locks). The value is a long number in microseconds and describes the time a certain lock was held by the operation. 

Notice that the lock information has been removed since 2.8.0-rc0 and it's currently being discussed if the information can be added back in, see [SERVER-16799](https://jira.mongodb.org/browse/SERVER-16799). One alternative is to add the time it took to acquire the lock instead. This section may change pending that decision.

| name                 | short name   | example                                  | description                       |
|----------------------|--------------|------------------------------------------|-----------------------------------|
| `wlock`              | `w`          | `15544933003`                            | numeric value in microseconds. <br><br>Optional | 
| `rlock`              | `r`          | `1002`                                   | numeric value in microseconds. <br><br>Optional | 


## Source ID

Log lines belong to log files, which belong to servers, which possibly belong to clusters or replica sets, which belong to users/customers. This spec only deals with the inner-most level of containment, called source_id. This value needs to be an identifier linking all log lines to their source (e.g. a log file). The spec does not specify a particular format for the value, as long as it is unambiguous for each source (i.e. the log file name is not an option).

| name                 | short name   | example                                  | description                       |
|----------------------|--------------|------------------------------------------|-----------------------------------|
| `source_id`          | `sid`        | `{ "$oid": "54eeb00f80f13e27653feafa" }`   | unambiguous identifier linking all log events from the same source together. |


## Special Cases

### Flushing mmaps

Information about flushing mmaps can be quite significant for issue diagnosis. The message part of the specific log line for these events follows this pattern:

```
flushing mmaps took <duration>ms  for <number> files
```

For these messages, the duration should be extracted and stored under the `duration` member.


### Time to load chunks

Information about loading chunks also has additional information embedded. The message pattern is: 

```
ChunkManager: time to load chunks for <namespace>: <duration>ms sequenceNumber: <number> version: <version> based on: <base version>
```

Both `namespace` and `duration` need to be extracted.


## Derived Values

Some useful information isn't logged explicitly but can be derived from the parsed members. 

### Query Shape

For many use cases it is important to know the query shape rather than the query itself. 

The query shape is defined as the JSON document that matches the query, except with all leaf values (including "data arrays" and "data documents") replaced by 1. Operators are not leaf values. Objects and arrays providing the syntax for operators are not leaf values. Additionally, all names in any object are sorted lexicographically in ascending order.

**Examples** 

| Query                                                 | Shape (JSON)                                       |
|-------------------------------------------------------|----------------------------------------------------|
| `{ a: "foo" }`                                        | `{ "a": 1 }`                                       |
| `{ a: {$in: [1, 2, "empty"]}}`                        | `{ "a": { "$in": 1 } }`                            |
| `{ b: 10, a: { $ne: 5 }}`                             | `{ "a": { "$ne": 1 }, "b": 1 }`                    |
| `{ a: "foo" }`                                        | `{ "a": 1 }`                                       |
| `{ a: null, $or: [ { b: "foo" }, { c: "bar" } ] }`    | `{ "$or": [ { "b": 1 }, { "c": 1 } ], "a": 1 }`    |
| `{ a: { b: 1, c: 1 } }`                               | `{ "a": 1 }`                                       |
| `{ a: [1, { foo: "bar" }, 3] }`                       | `{ "a": 1 }`                                       |


Each object that contains a query should also contain a query_shape.


| name                 | short name   | example                                  | description                       |
|----------------------|--------------|------------------------------------------|-----------------------------------|
| `query_shape`        | `qs`         | `{"bar": 1, "foo": {"$gt": 1}}`   | Every object that has a `query` should also have a `query_shape`.<br><br>Optional. |


### Connections vs. Context

The log message for a new connection does not use the context of the new connection, but `[initandlisten]` instead: 

**Example**
```
2015-01-05T16:43:51.031+1100 I NETWORK  [initandlisten] connection accepted from 127.0.0.1:55335 #53 (2 connections now open)
```

The equivalent "end connection" message on the other hand uses the connection context (here `conn53`): 

```
2015-01-05T17:02:47.780+1100 I NETWORK  [conn53] end connection 127.0.0.1:55335 (5 connections now open)
```

It is often helpful to find all log lines of a given connection, including the "connection accepted" message. Therefore, another member `connection` is derived, that is set to the connection in all cases. The connection number in the "connection accepted" message can be found behind the `#` symbol (in example above: `#53` for `conn53`).

The `connection` member should only be included if the `context` is `initandlisten`. 


| name                 | short name   | example                                  | description                       |
|----------------------|--------------|------------------------------------------|-----------------------------------|
| `connection`        | `con`         | `conn53`                                 | Optional.                         |




## Support Document 

Any software implementing this spec, be it as parser or processor of log lines, should state what exact version of the spec it supports, and if it deviates in any way from the spec. This statement should be in the documentation and/or queryable programmatically. This section describes the "meta-schema" that any implementation should follow. 

This spec will likely change over time and with future versions of MongoDB. Therefore it includes a version identifier. The schema also allows for certain choices to be made, for example, if short or long key names are used. These options are listed in the `options` section below. 

Additionally, many already existing implementations may not follow this schema yet, and backwards-breaking changes are hard. Therefore, a `delta` section allows to define some common deviations from the spec, for example not supporting certain fields, other alternative names for fields, or additional information parsed. These definitions should help to make transitions to the exact spec easier. However, specifying a `delta` is meant as a transitional feature and should not be used to keep changes from the schema permanently without intention to eventually converge. 


Example support document: 

```
{
  "id": "MongoDB Log Parsing Spec",
  "location": "https://github.com/rueckstiess/mongodb-log-spec",
  "version": "0.3.0",
  "comment": "shape information not extracted due to lack of standard.",
  "options": {
    "name_format": "short"
  },
  "delta": { 
    "unsupported": ["query_shape", "sort_shape"],
    "substitutions": {
      "timestamp": "datetime",
      "context": "thread",
      "dur": "d"
    }
  }
}
```

### Root Level Support Members

The `version` field should follow [semver 2.0](http://semver.org/spec/v2.0.0.html) compatible rules, and it should be the very first field parsed, as all other fields are subject to change in future versions.

| name                | example                                  | description                       |
|---------------------|------------------------------------------|-----------------------------------|
| `id`                | `MongoDB Log Parsing Spec`                   | Always use this value.            |
| `version`           | `0.3.5`                                  | Semantic Versioning compatibility rules apply |
| `options`           | `{ "name_format": "short" }`             | See Options below                 |
| `delta`             | `{ "unsupported": ["connection"] }`      | See Deviations below              |
| `comment`           | `Can use both "short" and "long" name format.` | Free text comment, that can be used to verbally explain certain choices, limitations or other useful information to the users of the implementation.<br><br>Optional |


### Options

All support schema options are "optional" as long as the spec defines a default value for each option.

Currently, `name_format` is the only available option, with possible values `long` and `short`. The default is `short`.


| name                | example                                  | description                       |
|---------------------|------------------------------------------|-----------------------------------|
| `name_format`       | `long`                                   | Default is `short`.<br><br>Optional |


### Deviations

The `delta` member contains information about how the actual implementation differs from the spec. This is to make transition to the spec easier.

If a member is supported but the format differs to the one stated in the spec, this member should be listed as unsupported.

The `delta` member can have the following sub-fields: 

| name                | example                                  | description                       |
|---------------------|------------------------------------------|-----------------------------------|
| `unsupported`       | `["query_shape", "sort_shape"]`          | JSON array of names not supported by this implementation. Naming should follow the `name_format` option. If not present, it is assumed that all names are supported. |
| `substitutions`     | `{ "ts": "t" }`                          | JSON object, mapping names from this spec to alternative names that the implementation supports instead. Naming should follow the `name_format` option. If not present, it is assumed that all fields are supported in exactly the way the spec dictates. |
| `additions`           | `{ "db": "separate database name", "coll": "separate collection name" }` | JSON object, mapping additional names not included in the spec to strings describing the member's purpose. |



