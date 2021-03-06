---
layout: page
title: Read, Write and Scan
order: 3
redirect_from: /store/read-write-scan.html

---

This page walks through the [Store API][scala-doc-store] and the [example servlet][source-servlet]
built with it by using `curl` to issue requests to a running server.  This walk through connects the
HTTP requests to the API methods.

## Setup

If you haven't already, [build or download the server.jar][get-server].

Next, initialize the database file.

<pre class="highlight">
java -jar finagle-server-{{site.vers}}.jar init -host 0xF47F4AA7602F3857 -cell 0x3B69376FF6CE2141 store.3kv
<div class="go">Dec 07, 2014 7:30:55 AM com.treode.disk.package$log$ initializedDrives
INFO: Initialized drives store.3kv</div>
</pre>

We called the database file `store.3kv`. We will explain the flags `-host` and `-cell` in the
[walkthrough on managing peers][manage-peers].

Now start the server.

<pre class="highlight">
java -jar finagle-server-{{site.vers}}.jar serve -solo store.3kv
<div class="go">INF [20141207-07:34:33.394] twitter: Finagle version 6.22.0 (rev=452c248f9e5bb71d13c48b90bb2e300f6e4fbf61) built at 20141013-200514
INF [20141207-07:34:35.307] treode: Opened drives store.3kv
INF [20141207-07:34:35.308] treode: Accepting peer connections to Host:F47F4AA7602F3857 for Cell:3B69376FF6CE2141 on 0.0.0.0/0.0.0.0:6278
INF [20141207-07:34:35.579] tracing: Tracer: com.twitter.finagle.zipkin.thrift.SamplingTracer</div>
</pre>

We'll discuss the flag `-solo` in the [walkthrough on managing peers][manage-peers]. And we'll say
more about `init` and `serve` in the [walkthrough on managing disks][manage-disks]. For now, we will
use this server to look at read, write and scan.

## Reading

### The `read` method

The `read` method allows you to read one *or more* rows.  Each row is identified by a 64 bit table
number and a key of bytes.  The method returns timestamped values; the order of the returned values
matches the order of the read operations.

```scala
class TxClock (val time: Long) extends AnyVal
class TableId (val id: Long) extends AnyVal
case class ReadOp (table: TableId, key: Bytes)
case class Value (time: TxClock, value: Option [Bytes])

def read (rt: TxClock, ops: ReadOp*): Async [Seq [Value]
```

The `rt` argument specifies when to read the data as-of.  TreodeDB returns the most recent value of
the row at that time and it includes the time the row was written.  After a call to read, you are
guaranteed that any subsequent writes will get a timestamp strictly after `rt`.  These connect
nicely to HTTP ETags, as we will see shortly.

The `read` method is asynchronous, as are all methods in the Store API.  Nothing actually happens
until you supply a callback through the chained `run` method.

```scala
read (TxClock.now, ReadOp (0x1, Bytes ("a key"))) run {
    case Success (vs) => // do something with the values
    case Failure (e)  => // do something with the exception
}
```

For this example to work with Finagle easily, we defined a
[method to convert][source-toTwitterFuture] the asynchronous call into Twitter's Future.

### Using `read` in the `GET` handler

You can read multiple rows at a time.  Though our example only reads one row at a time, you can use
the more general functionality as necessary for your requirements.  Our servlet uses
[Jackson][jackson] to convert between byte values in the database and JSON objects in HTTP requests
and responses; you can convert HTTP entities into bytes any way you need.

Our servlet handles GET requests for `/table/{table-id}?key={key-string}`, and it takes special
action depending on two headers.  If `If-Modified-Since` is present, then the servlet sends the full
response only when the value is newer; without the header it always sends the full response.  If
`Last-Modification-Before` is present, then the servlet uses it for the as-of time, otherwise it
defaults to `TxClock.now`.

To form the HTTP response, the servlet converts the byte value to a JSON object, and it uses the
timestamp on the value for the `ETag`.  If our servlet retrieved multiple rows and composed some
compound object from them, then it would use the *maximum* of the timestamps for the `ETag`.

We've introduced several convenience methods to get the request parameters; here's the meat of
[processing the GET request][source-read]:

```scala
def read (req: Request, tab: TableId, key: String): Async [Response] = {
  val rt = req.lastModificationBefore
  val ct = req.ifModifiedSince
  val ops = Seq (ReadOp (tab, Bytes (key)))
  store.read (rt, ops:_*) .map { vs =>
    val v = vs.head
    v.value match {
      case Some (value) if ct < v.time =>
        respond.json (req, v.time, value)
      case Some (_) =>
        respond (req, Status.NotModified)
      case None =>
        respond (req, Status.NotFound)
    }}}
```

### Try out `GET`

Now we know what the servlet does to handle a GET request:

<pre class="highlight">
curl -i -w'\n' http://localhost:7070/table/0x1?key=apple
<div class="go">HTTP/1.1 404 Not Found
Content-Type: text/plain
Content-Length: 0</div>
</pre>

However, there's no data in our database.  Although we haven't covered `write` yet, let's put some
data in there.  In fact, let's PUT a row and then PUT it a second time to overwrite it:

<pre class="highlight">
curl -i -w'\n' -XPUT \
    -H'content-type: application/json' \
    -d'{"v":"antelope"}' \
    http://localhost:7070/table/0x1?key=apple
<div class="go">HTTP/1.1 200 OK
ETag: 0x4FC8E28D00681
Content-Type: text/plain
Content-Length: 0
</div>
curl -i -w'\n' -XPUT \
    -H'content-type: application/json' \
    -d'{"v":"anteater"}' \
    http://localhost:7070/table/0x1?key=apple
<div class="go">HTTP/1.1 200 OK
ETag: 0x4FC8E29D1E621
Content-Type: text/plain
Content-Length: 0</div>
</pre>

Take note of the ETags in *your* run; they will be different.  You'll need the second ETag shortly.
Let's send off another GET request:

<pre class="highlight">
curl -i -w'\n' http://localhost:7070/table/0x1?key=apple
<div class="go">HTTP/1.1 200 OK
ETag: 0x4FC8E29D1E621
Content-Type: application/json
Content-Length: 16

{"v":"anteater"}</div>
</pre>

Without any special direction, we get the most recent value.  We can add a header to get the older
value.  Use the ETag from *your* second request above for `Last-Modification-Before`, but subtract
one from its value:

<pre class="highlight">
curl -i -w'\n' \
    -H'last-modification-before: 0x4FC8E29D1E620' \
    http://localhost:7070/table/0x1?key=apple
<div class="go">HTTP/1.1 200 OK
ETag: 0x4FC8E28D00681
Content-Type: application/json
Content-Length: 16

{"v":"antelope"}</div>
</pre>

With the addition of the header, we get the older value and its ETag.  Neat, eh?

## Writing

### The `write` method

The `write` method allows you to update one or more rows *atomically*.  You supply a condition time,
and if any of the rows has been changed since that time, then the write will do nothing.  If none of
the rows has been updated, then the write will succeed and return the new timestamp of the modified
rows.

```scala
case class TxId (id: Bytes, time: Instant)

sealed abstract class WriteOp
case class Create (table: TableId, key: Bytes, value: Bytes) extends WriteOp
case class Hold (table: TableId, key: Bytes) extends WriteOp
case class Update (table: TableId, key: Bytes, value: Bytes) extends WriteOp
case class Delete (table: TableId, key: Bytes) extends WriteOp

def write (xid: TxId, ct: TxClock, ops: WriteOp*): Async [TxClock]
```

The `ct` argument is the condition time.  The latest timestamp on every row must be on or before
`ct`.  The `xid` argument is a transaction ID; we will discuss it more in the next section.

The `write` method supports four kinds of updates.  `Create` and `Update` are similar, except that
`Create` requires that the row is currently deleted, whereas `Update` will apply the change whether
the row currently exists or not.  `Delete` should be clear enough.  `Hold` does not change the value
of the row; thus it may participate in the write condition without effecting a change.

When all has gone well, the `write` method returns the new timestamp on the rows.  When things go
wrong, the method yields some exceptions.  If rows failed to satisfy the condition time, that is if
they have been updated since that time, then the method yields `StaleException`.  If the write tried
to create a row that already exists, then it yields `CollisionException`; this can only happen if
the write includes a `Create` operation.  Finally if too many machines in the storage cell are down
or multiple writes deadlocked, then it yields `TimeoutException`.

Notice that we say *yields* rather than *throws* the exception.  The `write` method is asynchronous,
so the exception is passed to a callback.

```scala
val op1 = Update (0x1, Bytes ("a key"), Bytes ("a value"))
write (TxId.random (host), TxClock.now, op1) run {
    case Success (wt)                    => // 200 Ok, ETag: wt
    case Failure (e: StaleException)     => // 412 Precondition failed
    case Failure (e: CollisionException) => // 409 Conflict
    case Failure (e: TimeoutException)   => // maybe retry
    case Failure (e)                     => // 500 Internal server error
}
```

### The `status` method

Sometimes your client will loose its connection to the server.  It happens.  What if the connection
should break while the client is awaiting a response to a write request?  The `status` method allows
a client to reconnect to *any* server the the cell and learn the outcome of an earlier write.

```scala
sealed abstract class TxStatus
case object Aborted extends TxStatus
case class Committed (wt: TxClock) extends TxStatus

def status (xid: TxId): Async [TxStatus]
```

The transaction ID must be universally unique.  You can construct one on the server side or on the
client side.  If it's constructed on server side, then the client does not know the id and has
little recourse if its network connection is broken.  On the other hand, if the client constructs
the id, then it and can use it after reconnecting to get the status of its write.

The transaction ID includes a `time` field.  This is not necessary for correctness of the atomic
write, so the transaction can complete even when it's slightly skewed from server's clock.  The
server only uses the `time` to purge old statuses.

### Using `write` in the `PUT` handler

Our servlet also handles PUT requests for `/table/{table-id}?key={key-string}`.  If the client
chooses a transaction id, it may place it in the `Transaction-ID` header, otherwise the server will
make one.  The client can include the condition time, that is the `ETag` from an earlier read, in
the `If-Unmodified-Since` header; without that the server will use `TxClock.now` as the condition
time.

Our servlet will respond either with `200 Ok` and an ETag when the write succeeds, or with
`412 Precondition Failed` when the write fails because the `If-Unmodified-Since` check failed.

The [code for handling][source-write] `PUT`:

```scala
def put (req: Request, table: TableId, key: String): Async [Response] = {
  val tx = req.transactionId (host)
  val ct = req.ifUnmodifiedSince
  val value = req.readJson [JsonNode]
  val ops = Seq (WriteOp.Update (table, Bytes (key), value.toBytes))
  store.write (tx, ct, ops:_*)
  .map [Response] { vt =>
    val rsp = req.response
    rsp.status = Status.Ok
    rsp.headerMap.add ("ETag", vt.toString)
    rsp
  }
  .recover {
    case _: StaleException =>
      respond (req, Status.PreconditionFailed)
  }}
```

### Try out `PUT`

We have already tried PUT, back when we added data to GET.  Now we know what happened behind the
scenes.  However, we haven't yet tried to put data with an precondition. Using the ETag from *your*
second PUT request above, but again subtracting one:

<pre class="highlight">
curl -i -w'\n' -XPUT \
    -H'content-type: application/json' \
    -H'if-unmodified-since: 0x4FC8E29D1E620' \
    -d'{"v":"zebra"}' \
    http://localhost:7070/table/0x1?key=apple
<div class="go">HTTP/1.1 412 Precondition Failed
Content-Type: text/plain
Content-Length: 0</div>
</pre>

The failure tells the client that its update is based on stale data.  The client can GET the new
data, and then the client can reconcile the update and new data as best fits its needs.

## Scanning

### The `scan` method

The `scan` method iterates rows of a table.  It can iterate only the most recent value, or the
historical values over a window of time.  It can iterate all the rows, or only those rows whose key
hashes into a slice.  These options allow you to page through a table and to scan large tables in
parallel.

```scala
sealed abstract class Bound [A]
case class Inclusive [A] (bound: A) extends Bound [A]
case class Exclusive [A] (bound: A) extends Bound [A]

sealed abstract class Window
case class Recent (later: Bound [TxClock], earlier: Bound [TxClock]) extends Window
case class Between (later: Bound [TxClock], earlier: Bound [TxClock]) extends Window
case class Through (later: Bound [TxClock], earlier: TxClock) extends Window

case class Slice (slice: Int, nslices: Int)

case class Batch (entries: Int, bytes: Int)

case class Cell (key: Bytes, time: TxClock, value: Option [Bytes])

def scan (table: TableId, start: Bound [Key], window: Window, slice: Slice, batch: Batch): BatchIterator [Cell]
```

The `start` argument begins the scan at a given key, and the batch iterator provides a
`whilst` method for you to end the scan as you desire.

The `window` argument controls which portion of the update history will be included. The time window
called `Latest` is the most like scanning a table without history.  It selects the most recent value
for a row before `later` as long as that value was set after `earlier`.  The `Between` time window
selects values with timestamps between `later` and `earlier`; it will return multiple rows for a key
if that key was updated multiple times during the period.  The time window called `Through` combines
the two.  It chooses values with timestamps between `later` and `earlier` as well as the most recent
value as of `earlier`.

The `slice` argument allows you to specify how large of a hash space you want to slice the keys
into&mdash;any power of two works&mdash;and which of those slices you want in this scan. If you
had a very small table or your scan wasn't computationally expensive, you might scan using just
one worker, that is one slice. On the other hand, if your table was large or the scan loops was
intensive, you might setup multiple workers to each scan a portion of the table in parallel. For
example, you could have 16 scan workers, each processing 1/16th of the table. Which slice each one
gets depends on how the keys hash.

Finally, the `batch` argument provides a hint for TreodeDB. If you will stop after scanning only a
few rows, you can supply small numbers for the batch size. On the other hand, if you will scan most
of the table, you can supply large numbers for the batch size. Tailoring this properly to
expectations will help performance.

The scan returns an asynchronous [BatchIterator][scala-doc-iterator], which provides many helpful
methods like `filter`, `map`, and `whilst`.    That last method is similar to the `while` keyword,
but works in an asynchronous way.  The asynchronous iterator also has a `foreach` method, but you
may find `toSeq` and `toMap` more helpful. Although there is much flexibility available here, we
expect many uses of the asynchronous iterator will look something like:

```scala
val start: Bound [Key] = ...
val end: Bound [Key] = ...
scan (0x1, start, Latest.now)
.toSeqWhile (end >* _.key)
.run {
    case Success ((vs, next) => // do something with the values
    case Failure (e)         => // do something with the exception
}
```

### Using `scan` in the `GET` handler

The `scan` method offers great flexibility, however our servlet will limit the options.  A GET
request for `/table/{table-id}?slice=0&nslices=1` will use the `Latest` window. Much like with read,
scanning will use the two headers `If-Modified-Since` and `Last-Modification-Before`.  The first
will correspond to `earlier` and the second to `later`, so the servlet will return the most recent
update as of `Last-Modification-Before` (inclusive, default now) if that update was made after
`If-Modified-Since` (inclusive, default UNIX epoch).  The parameters `slice` and `nslices` are
optional, and the servlet will scan the full table if they are missing.

Our [servlet method][source-scan] to read the headers, scan the table and convert result to JSON:

```scala
def scan (req: Request, table: TableId): Async [Response] = {
  val rt = req.lastModificationBefore
  val ct = req.ifModifiedSince
  val window = Window.Latest (rt, true, ct, false)
  val slice = req.slice
  val iter = store
      .scan (table, Bound.firstKey, window, slice)
      .filter (_.value.isDefined)
  supply (respond.json (req, iter))
}
```

### Try out `GET` for `scan`

This is going to do something very close to what you hopefully expect.

<pre class="highlight">
curl -i -w'\n' http://localhost:7070/table/0x1
<div class="go">HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 66

[{"key":"apple","time":1403587424020001,"value":{"v":"anteater"}}]</div>
</pre>

And you can try out slices too; remember that the number of slices must be a power of two.

<pre class="highlight">
curl -i -w'\n' http://localhost:7070/table/0x1?slice=0\&amp;nslices=2
<div class="go">HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 2

[]
</div>
curl -i -w'\n' http://localhost:7070/table/0x1?slice=1\&amp;nslices=2
<div class="go">HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 66

[{"key":"apple","time":1403587424020001,"value":{"v":"anteater"}}]</div>
</pre>

## Testing

After writing a complex servlet, you'll want to test it. We've written example tests that use
[RestAssured][rest-assured] and [Treode's testing stub][scala-doc-stub].  RestAssured provides a
specification trait that makes it easy to submit requests and check responses.  And Treode provides
an in-memory stub that's very easy to setup.  You can consult [this fully worked][source-spec] test.

## Next

Now you're an expert on reading, writing and scanning with TreodeDB.  Move on to [add and remove disks][manage-disks].

[finatra-testing]: //finatra.info/docs/index.html#testing "Finatra Testing"

[get-server]: /getting-started "Get the example server"

[jackson]: //wiki.fasterxml.com/JacksonHome "Jackson JSON Processor"

[manage-disks]: /managing-disks "Managing Disks"

[manage-peers]: /managing-peers "Managing Peers"

[rest-assured]: https://github.com/jayway/rest-assured "RestAssured on GitHub"

[source-servlet]: {{site.blob}}/examples/finagle/src/main/scala/example/Resource.scala "Example servlet using the Store API"

[source-spec]: {{site.blob}}/examples/finagle/src/test/scala/example/ResourceSpec.scala "Example test using StubStore"

[source-read]: {{site.blob}}/examples/finagle/src/main/scala/example/Resource.scala#L34-47 "Example to handle GET for read"

[source-scan]: {{site.blob}}/examples/finagle/src/main/scala/example/Resource.scala#L49-58 "Example to handle GET for scan"

[source-write]: {{site.blob}}/examples/finagle/src/main/scala/example/Resource.scala#L70-85 "Example to handle PUT"

[source-toTwitterFuture]: {{site.blob}}/twitter-server/src/com/treode/twitter/util/package.scala#L29-36 "Method to convert Treode's Async to Twitter's Future"

[scala-doc-iterator]: //oss.treode.com/docs/scala/store/{{site.vers}}#com.treode.async.BatchIterator "BatchIterator API"

[scala-doc-store]: //oss.treode.com/docs/scala/store/{{site.vers}}#com.treode.store.Store "Store API"

[scala-doc-stub]: //oss.treode.com/docs/scala/store/{{site.vers}}#com.treode.store.stubs.StubStore "Stub Store API"
