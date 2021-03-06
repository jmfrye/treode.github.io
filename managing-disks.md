---
layout: page
title: Managing Disks
order: 4
redirect_from: /store/managing-disks.html

---

The Treode API breaks creating a new database and opening an existing database into two distinct
steps.

```scala
object Store {

  def init (
      hostId: HostId,
      cellId: CellId,
      superBlockBits: Int,
      segmentBits: Int,
      blockBits: Int,
      diskBytes: Long,
      paths: Path*
  )

  def recover (
      bindAddr: SocketAddress,
      shareAddr: SocketAddress,
      paths: Path*
  ) (implicit
      diskConfig: DiskConfig,
      clusterConfig: ClusterConfig,
      storeConfig: StoreConfig
  ): Async [Controller]
}
```

The `main` method of our example checks the first command line argument, and it performs these
actions as two separate steps.

<pre class="highlight">
<span class="c"># Create a database; performs Store.init and exits.</span>
java -jar finagle-server-{{site.vers}}.jar init -host 0xF47F4AA7602F3857 -cell 0x3B69376FF6CE2141 store.3kv

<span class="c"># Open an existing database and serve it; performs Store.recover.</span>
java -jar finagle-server-{{site.vers}}.jar serve -solo store.3kv
</pre>

These are two steps since often an install script will initialize the database, and an `/etc/rc.d`
script will start the server.

## Creating a new database

You may supply multiple paths to the `init` method.  Those paths may name files or raw devices;
either way you specify a disk geometry.  Although disks come in different sizes and block sizes,
those provided to `init` will be treated with the same geometry.  You can add disks with different
characteristics later.

TreodeDB allocates and frees disk space in segments.  The `segmentBits` argument specifies the
segments size of (2^segmentBits) bytes.  The system will cap its allocation according to `diskBytes`
which gives the total byte size of the disk; it does not need to be a whole multiple of the segment
size.  TreodeDB will write the log and pages aligned per `blockBits`.  Like most storage systems,
the database places superblocks in known locations, and the `superBlockBits` arguments allows the
superblock to span multiple disk blocks.

The `hostId` and `cellId` identify this peer and the storage cell, where a cell is one or more
servers cooperating to provide the same database.  The `hostId` must be unique for every host in the
cell, and the `cellId` must be the same across all hosts in the cell.  If you are running multiple
cells, you should consider making a unique `cellId` for each one.  A peer will reject connections
from another peer if the `cellId` does not match, and this can prevent corruption of data.  Both
arguments are 64 bit values.  We aren't very imaginative, so when we need a new `hostId` or
`cellId`, we invoke this magic incantation:

    head -c 8 /dev/random | hexdump -e "\"0x\" 8/1 \"%02X\" \"\n\""

You must provide a `hostId` and `cellId`, even when starting a server to run stand alone.  Your
business is going to be booming someday, and you'll be glad that you prepared early for running a
large cluster.

So let's create a database:

<pre class="highlight">
java -jar finagle-server-{{site.vers}}.jar init -host 0xF47F4AA7602F3857 -cell 0x3B69376FF6CE2141 store.3kv
<div class="go">Dec 07, 2014 8:48:55 AM com.treode.disk.package$log$ initializedDrives
INFO: Initialized drives store.3kv</div>
</pre>

## Opening an existing database

You may also supply multiple paths to the `recover` method, however it is only necessary to supply
one.  All disks attached to a server are listed in the superblock on all disks, so TreodeDB can find
all disks when given any one of them.  Without this trick, adding and removing disks would require
you to synchronize your startup scripts (or configuration files) with the database's commands to
change the disks.

You also supply two related socket addresses.  The `bindAddr` is the address on which the server
will listen for connections from peers in the storage server, and it often does not specify the host
explicitly.  The `shareAddr` is the address other peers use to connect to this one, and it must be
given a concrete host.  In many cases, one would bind on `*:port` and share `hostname:port`, that is
one would bind and share the same port number.  In other more specialized cases, one might setup IP
chains or other network configuration that would require the two port numbers to differ.

The method requires several configuration objects, which we will not discuss in detail here.  For
each one, you can use `suggested`, for example `DiskConfig.suggested`.  The objects have a copy
method with default arguments, which allows you to change selected fields while easily retaining the
others.

The `recover` method returns a `Controller` which has a wealth of fields and methods.  One of the
fields is the `Store` with read, write and scan.  The other fields and methods are for
administration, and we are going to discuss those related to adding and removing disks.

Now let's open the database we created earlier:

<pre class="highlight">
java -jar finagle-server-{{site.vers}}.jar serve -solo store.3kv
<div class="go">INF [20141207-08:52:34.222] twitter: Finagle version 6.22.0 (rev=452c248f9e5bb71d13c48b90bb2e300f6e4fbf61) built at 20141013-200514
INF [20141207-08:52:36.736] treode: Opened drives store.3kv
INF [20141207-08:52:36.736] treode: Accepting peer connections to Host:F47F4AA7602F3857 for Cell:3B69376FF6CE2141 on 0.0.0.0/0.0.0.0:6278
INF [20141207-08:52:37.233] tracing: Tracer: com.twitter.finagle.zipkin.thrift.SamplingTracer</div>
</pre>

## Adding a drive

As your database grows, you will want to add a disk for more space, and the controller has the
`attach` method for this purpose.

```scala
case class DriveGeometry (segmentBits: Int, blockBits: Int, diskBytes: Long)

case class DriveAttachment (path: Path, geometry: DriveGeometry)

def attach (items: DriveAttachment*): Async [Unit]
```

We have created an HTTP endpoint for this method in our example server.  To try it out, startup a
server as explained above, and then:

<pre class="highlight">
curl -i -w'\n' -XPOST -d@- \
    -H'content-type: application/json' \
    http://localhost:9990/admin/treode/drives/attach &lt;&lt; EOF
[ {
    "path": "store2.3kv",
    "geometry": { "segmentBits": 30, "blockBits": 13, "diskBytes": 1099511627776}
  } ]
EOF
<div class="go">HTTP/1.1 200 OK
Content-Length: 0</div>
</pre>

You can list the drives to check that the new disk is included:

<pre class="highlight">
curl -i -w'\n' http://localhost:9990/admin/treode/drives
<div class="go">HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 246

[ {
    "path": "store2.3kv",
    "geometry": {"segmentBits":30, "blockBits":13, "diskBytes":1099511627776},
    "allocated": 2,
    "draining": false
  }, {
    "path": "store.3kv",
    "geometry": {"segmentBits":30, "blockBits":13, "diskBytes":1099511627776},
    "allocated":3,
    "draining": false
  } ]</div>
</pre>

## Removing a drive

There will be times when you want to remove a disk too.  Perhaps it's getting old and you want to
proactively replace it before it yields read errors.  Or maybe it has become too small and you want
to swap it for a larger one.  The controller has the `drain` method for this purpose.

```scala
def drain (paths: Path*): Async [Unit]
```

Before draining a disk, you should check your startup scripts first (or configuration files,
wherever you list the paths fed to `recover`).  If that includes a disk you want to drain, remove
the path from there first, then issue the `drain` command.  When TreodeDB has moved all data from
the old disk to others, it will update the list of paths in the superblocks.  This way, you don't
need to precisely synchronize your scripts and configuration with TreodeDB detaching drives.

We have also created an HTTP endpoint for this method.  To try it out, type:

<pre class="highlight">
curl -i -w'\n' -XPOST \
    -H'content-type: application/json' \
    -d'["store2.3kv"]' \
    http://localhost:9990/admin/treode/drives/drain
<div class="go">HTTP/1.1 200 OK
Content-Length: 0</div>
</pre>

If you have lots of data on the disk, you'll see it listed as draining until it's finally detached:

<pre class="highlight">
curl -i -w'\n' http://localhost:9990/admin/treode/drives
<div class="go">HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 245

[ {
    "path": "store2.3kv",
    "geometry": {"segmentBits":30, "blockBits":13, "diskBytes":1099511627776},
    "allocated": 2,
    "draining": true
  }, {
    "path": "store.3kv",
    "geometry": {"segmentBits":30, "blockBits":13, "diskBytes":1099511627776},
    "allocated":3,
    "draining": false
  } ]</div>
</pre>

After only this short tutorial though, you probably have little or no data on the disk, so TreodeDB
detached it quickly.  You will have seen a log message annoucing the detachment, and your output
from the previous command will look more like:

<pre class="highlight">
curl -i -w'\n' http://localhost:7070/drives
<div class="go">HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 123

[ {
    "path": "store2.3kv",
    "geometry": {"segmentBits":30, "blockBits":13, "diskBytes":1099511627776},
    "allocated": 2,
    "draining": true
  } ]</div>
</pre>

## Next

That's all there is to adding and removing drives.  Move on to [manage peers][manage-peers].

[manage-peers]: /managing-peers "Managing Peers"
