
A working guide to kestrel
==========================

Kestrel is a very simple message queue that runs on the JVM and uses the
memcache protocol (with some extensions) to talk to clients.

A single kestrel server has a set of queues identified by a name, which is
also the filename of that queue's journal file (usually in
`/var/spool/kestrel`). Each queue is a strictly-ordered FIFO of "items" of
binary data. Usually this data is in some serialized format like JSON or
ruby's marshal format.

Generally queue names should be limited to alphanumerics `[A-Za-z0-9]`, dash
(`-`) and underline (`_`). In practice, kestrel doesn't enforce any
restrictions other than the name can't contain slash (`/`) because that can't
be used in filenames, squiggle (`~`) because it's used for temporary files,
plus (`+`) because it's used for fanout queues, and dot (`.`) because it's
reserved for future use. Queue names are case-sensitive, but if you're running
kestrel on OS X or Windows, you will want to refrain from taking advantage of
this, since the journal filenames on those two platforms are *not*
case-sensitive.

A cluster of kestrel servers is like a memcache cluster: the servers don't
know about each other, and don't do any cross-communication, so you can add as
many as you like. Clients have a list of all servers in the cluster, and pick
one at random for each operation. In this way, each queue appears to be spread
out across every server, with items in a loose ordering.

When kestrel starts up, it scans the journal folder and creates queues based
on any journal files it finds there, to restore state to the way it was when
it last shutdown (or was killed or died). New queues are created by referring
to them (for example, adding or trying to remove an item). A queue can be
deleted with the "delete" command.


Configuration
-------------

All of the per-queue configuration can be set in the global scope of
`production.conf` as a default for all queues, or in the per-queue
configuration to override the defaults for a specific queue. You can see an
example of this in the default config file.

To confirm the current configuration of each queue, send "dump_config" to
a server (which can be done over telnet).

To reload the config file on a running server, send "reload" the same way.
You should immediately see the changes in "dump_config", to confirm.

- `max_items` (infinite)

  Set a hard limit on the number of items this queue can hold. When the queue
  is full, `discard_old_when_full` dictates the behavior when a client
  attempts to add another item.

- `max_size` (infinite)

  Set a hard limit on the number of bytes (of data in queued items) this
  queue can hold. When the queue is full, `discard_old_when_full` dictates
  the behavior when a client attempts to add another item.

- `max_item_size` (infinite)

  Set a hard limit on the number of bytes a single queued item can contain.
  An add request for an item larger than this will be rejected. 

- `discard_old_when_full` (false)

  If this is false, when a queue is full, clients attempting to add another
  item will get an error. No new items will be accepted. If this is true, old
  items will be discarded to make room for the new one. This settting has no
  effect unless at least one of `max_items` or `max_size` is set.

- `journal` (true)

  If false, don't keep a journal file for this queue. When kestrel exits, any
  remaining contents in the queue will be lost.

- `sync_journal` (false)

  If true, sync the journal file on disk after each write. This is usually
  not necessary but is available for the paranoid. It will probably reduce
  the maximum throughput of the server.

- `max_journal_size` (16MB)

  When a journal reaches this size, it will be rolled over to a new file as
  soon as the queue is empty. The value must be given in bytes.

- `max_journal_overflow` (10)

  If a journal file grows to this many times its desired maximum size, and
  the total queue contents (in bytes) are smaller than the desired maximum
  size, the journal file will be rewritten from scratch, to avoid using up
  all disk space. For example, using the default `max_journal_size` of 16MB
  and `max_journal_overflow` of 10, if the journal file ever grows beyond
  160MB (and the queue's contents are less than 16MB), the journal file will
  be re-written.

- `max_memory_size` (128MB)

  If a queue's contents grow past this size, only this part will be kept in
  memory. Newly added items will be written directly to the journal file and
  read back into memory as the queue is drained. This setting is a release
  valve to keep a backed-up queue from consuming all memory. The value must
  be given in bytes.

- `max_age` (0 = off)

  Expiration time (in milliseconds) for items on this queue. Any item that
  has been sitting on the queue longer than this amount will be discarded.
  Clients may also attach an expiration time when adding items to a queue,
  but if the expiration time is longer than `max_age`, `max_age` will be
  used instead.

- `move_expired_to` (none)

  Name of a queue to add expired items to. If set, expired items are added to
  the requested queue as if by a `SET` command. This can be used to implement
  special processing for expired items, or to implement a simple "delayed
  processing" queue.


The journal file
----------------

The journal file is the only on-disk storage of a queue's contents, and it's
just a sequential record of each add or remove operation that's happened on
that queue. When kestrel starts up, it replays each queue's journal to build
up the in-memory queue that it uses for client queries.

The journal file is rotated in one of two conditions:

1. the queue is empty and the journal is larger than `max_journal_size`

2. the queue is smaller than `max_journal_size` but the journal is larger
   than `max_journal_overflow` times `max_journal_size`

For example, if `max_journal_size` is 16MB (the default), and
`max_journal_overflow` is 10 (also the default), then if the queue is empty
and the journal is larger than 16MB, it will be rotated into a new (empty)
file. If the queue is smaller than 16MB, but the journal is larger than 160MB,
the journal will be rotated to contain just the live items.

You can turn the journal off for a queue (`journal` = false) and the queue
will exist only in memory. If the server restarts, all enqueued items are
lost. You can also force a queue's journal to be sync'd to disk after every
write operation (`sync_journal` = true) at a performance cost.

If a queue grows past `max_memory_size` bytes (128MB by default), only the
first 128MB is kept in memory. The journal is used to track later items, and
as items are removed, the journal is played forward to keep 128MB in memory.
This is usually known as "read-behind" mode, but Twitter engineers sometimes
refer to it as the "square snake" because of the diagram used to brainstorm
the implementation. When a queue is in read-behind mode, removing an item will
often cause 2 disk operations instead of one: one to record the remove, and
one to read an item in from disk to keep 128MB in memory. This is the
trade-off to avoid filling memory and crashing the JVM.


Item expiration
---------------

When they come from a client, expiration times are handled in the same way as
memcache: if the number is small (less than one million), it's interpreted as
a relative number of seconds from now. Otherwise it's interpreted as an
absolute unix epoch time, in seconds since the beginning of 1 January 1970
GMT.

Expiration times are immediately translated into an absolute time, in
*milliseconds*, and if it's further in the future than the queue's `max_age`,
the `max_age` is used instead. An expiration of 0, which is usually the
default, means an item never expires.

Expired items are flushed from a queue whenever a new item is added or
removed. An idle queue won't have any items expired, but you can trigger a
check by doing a "peek" on it.

The global config option `expiration_timer_frequency_seconds` can be used to
start a background thread that periodically removes expired items from the
head of each queue. See the `README.md` file for more.


Fanout Queues
-------------

If a queue name has a `+` in it (like "`orders+audit`"), it's treated as a
fanout queue, using the format `<parent>+<child>`. These queues belong to a
parent queue -- in this example, the "orders" queue. Every item written into
a parent queue will also be written into each of its children.

Fanout queues each have their own journal file (if the parent queue has a
journal file) and otherwise behave exactly like any other queue. You can get
and peek and even add items directly to a child queue if you want. It uses the
parent queue's configuration instead of having independent child queue
configuration blocks.

When a fanout queue is first referenced by a client, the journal file (if any)
is created, and it will start receiving new items written to the parent queue.
Existing items are not copied over. A fanout queue can be deleted to stop it
from receiving new items.


Memcache commands
-----------------

- `SET <queue-name> <flags (ignored)> <expiration> <# bytes>`

  Add an item to a queue. It may fail if the queue has a size or item limit
  and it's full.

- `GET <queue-name>[options]`

  Remove an item from a queue. It will return an empty response immediately if
  the queue is empty. The queue name may be followed by options separated
  by `/`:

    - `/t=<milliseconds>`

      Wait up to a given time limit for a new item to arrive. If an item arrives
      on the queue within this timeout, it's returned as normal. Otherwise,
      after that timeout, an empty response is returned.

    - `/open`

      Tentatively remove an item from the queue. The item is returned as usual
      but is also set aside in case the client disappears before sending a
      "close" request. (See "Reliable Reads" below.)

    - `/close`

      Close any existing open read. (See "Reliable Reads" below.)

    - `/abort`

      Cancel any existing open read, returing that item to the head of the
      queue. It will be the next item fetched. (See "Reliable Reads" below.)

    - `/peek`

      Return the first available item from the queue, if there is one, but don't
      remove it. You can't combine this with any of the reliable read options.

  For example, to open a new read, waiting up to 500msec for an item:

        GET work/t=500/open

  Or to close an existing read and open a new one:

        GET work/close/open

- `DELETE <queue-name>`

  Drop a queue, discarding any items in it, and deleting any associated
  journal files.

- `FLUSH <queue-name>`

  Discard all items remaining in this queue. The queue remains live and new
  items can be added. The time it takes to flush will be linear to the current
  queue size, and any other activity on this queue will block while it's being
  flushed.

- `FLUSH_ALL`

  Discard all items remaining in all queues. The queues are flushed one at a
  time, as if kestrel received a `FLUSH` command for each queue.

- `VERSION`

  Display the kestrel version in a way compatible with memcache.

- `SHUTDOWN`

  Cleanly shutdown the server and exit.

- `RELOAD`

  Reload the config file and reconfigure all queues. This should have no
  noticable effect on the server's responsiveness.

- `DUMP_CONFIG`

  Dump a list of each queue currently known to the server, and list the config
  values for each queue. The format is:

        queue 'master' {
          max_items=2147483647
          max_size=9223372036854775807
          max_age=0
          max_journal_size=16277216
          max_memory_size=134217728
          max_journal_overflow=10
          max_journal_size_absolute=9223372036854775807
          discard_old_when_full=false
          journal=true
          sync_journal=false
        }

  The last queue will be followed by `END` on a line by itself.

- `STATS`

  Display server stats in memcache style. They're described below.

- `DUMP_STATS`

  Display server stats in a more readable style, grouped by queue. They're
  described below.


Reliable reads
--------------

Normally when a client removes an item from the queue, kestrel immediately
discards the item and assumes the client has taken ownership. This isn't
always safe, because a client could crash or lose the network connection
before it gets the item. So kestrel also supports a "reliable read" that
happens in two stages, using the `/open` and `/close` options to `GET`.

When `/open` is used, and an item is available, kestrel will remove it from
the queue and send it to the client as usual. But it will also set the item
aside. If a client disconnects while it has an open read, the item is put back
into the queue, at the head, so it will be the next item fetched. Only one
item can be "open" per client connection.

A previous open request is closed with `/close`. The server will reject any
attempt to open another read when one is already open, but it will ignore
`/close` if there's no open request, so that you can add `/close` to every
`GET` request for convenience.

If for some reason you want to abort a read without disconnecting, you can use
`/abort`. But because aborted items are placed back at the head of the queue,
this isn't a good way to deal with client errors. Since the error-causing item
will always be the next one available, you'll end up bouncing the same item
around between clients instead of making progress.

There's always a trade-off: either potentially lose items or potentially
receive the same item multiple times. Reliable reads choose the latter option.
To use this tactic successfully, work items should be idempotent, meaning the
work could be done 2 or 3 times and have the same effect as if it had been
done only once (except wasting some resources).

Example:

    GET dirty_jobs/close/open
    (receives job 1)
    GET dirty_jobs/close/open
    (closes job 1, receives job 2)
    ...etc...


Server stats
------------

Global stats reported by kestrel are:

- `uptime` - seconds the server has been online
- `time` - current time in unix epoch
- `version` - version string, like "1.2"
- `curr_items` - total of items waiting in all queues
- `total_itmes` - total of items that have ever been added in this server's
  lifetime
- `bytes` - total byte size of items waiting in all queues
- `curr_connections` - current open connections from clients
- `total_connections` - total connections that have been opened in this
  server's lifetime
- `cmd_get` - total `GET` requests
- `cmd_set` - total `SET` requests
- `cmd_peek` - total `GET/peek` requests
- `get_hits` - total `GET` requests that received an item
- `get_misses` - total `GET` requests on an empty queue
- `bytes_read` - total bytes read from clients
- `bytes_written` - total bytes written to clients

For each queue, the following stats are also reported:

- `items` - items waiting in this queue
- `bytes` - total byte size of items waiting in this queue
- `total_items` - total items that have been added to this queue in this
  server's lifetime
- `logsize` - byte size of the queue's journal file
- `expired_items` - total items that have been expired from this queue in this
  server's lifetime
- `mem_items` - items in this queue that are currently in memory
- `mem_bytes` - total byte size of items in this queue that are currently in
  memory (will always be less than or equal to `max_memory_size` config for
  the queue)
- `age` - time, in milliseconds, that the last item to be fetched from this
  queue had been waiting; that is, the time between `SET` and `GET`; if the
  queue is empty, this will always be zero
- `discarded` - number of items discarded because the queue was too full
- `waiters` - number of clients waiting for an item from this queue (using
  `GET/t`)
- `open_transactions` - items read with `/open` but not yet confirmed


Kestrel as a library
--------------------

You can use kestrel as a library by just sticking the jar on your classpath.
It's a cheap way to get a durable work queue for inter-process or inter-thread
communication. Each queue is represented by a `PersistentQueue` object:

    class PersistentQueue(persistencePath: String, val name: String, val config: ConfigMap)

and must be initialized before using:

    def setup(): Unit

specifying the path for the journal files (if the queue will be journaled),
the name of the queue, and a configgy `ConfigMap` block with any special
configuration. (See "Configuration" above.)

To add an item to a queue:

    def add(value: Array[Byte], expiry: Long): Boolean

It will return `false` if the item was rejected because the queue was full.

Queue items are represented by a case class:

    case class QItem(addTime: Long, expiry: Long, data: Array[Byte], var xid: Int)

and several operations exist to remove or peek at the head item:

    def peek(): Option[QItem]
    def remove(): Option[QItem]

To open a reliable read, set `transaction` true, and later confirm or unremove
the item by its `xid`:

    def remove(transaction: Boolean): Option[QItem]
    def unremove(xid: Int)
    def confirmRemove(xid: Int)

You can also asynchronously remove or peek at items using actors, as either
a `receive` or `react` callback:

    def removeReact(timeoutAbsolute: Long, transaction: Boolean)(f: Option[QItem] => Unit): Unit
    def removeReceive(timeoutAbsolute: Long, transaction: Boolean): Option[QItem]
    def peekReact(timeoutAbsolute: Long)(f: Option[QItem] => Unit): Unit
    def peekReceive(timeoutAbsolute: Long): Option[QItem]

When done, you should close the queue:

    def close(): Unit
    def isClosed: Boolean

Here's a short example:

    var queue = new PersistentQueue("/var/spool/kestrel", "work", config)
    queue.setup()

    // add an item with no expiration:
    queue.add("hello".getBytes, 0)

    // start to remove it, then back out:
    val item = queue.remove(true)
    queue.unremove(item.xid)

    // remove an item with a 500msec timeout, and confirm it:
    queue.removeReact(System.currentTimeMillis + 500, true) { x =>
      x match {
        case None =>
          println("nothing. :(")
        case Some(item) =>
          println("got: " + new String(item.data))
          queue.confirmRemove(item.xid)
      }
    }

    queue.close()
