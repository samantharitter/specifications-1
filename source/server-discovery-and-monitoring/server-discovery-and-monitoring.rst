===============================
Server Discovery And Monitoring
===============================

:Spec: 101
:Title: Server Discovery And Monitoring
:Author: A\. Jesse Jiryu Davis
:Advisors: David Golden, Craig Wilson
:Status: Draft
:Type: Standards
:Last Modified: September 8, 2014

.. contents::

--------

Abstract
--------

This spec defines how a MongoDB client discovers and monitors one or more servers.
It covers monitoring a single server, a set of mongoses, or a replica set.
How does the client determine what type of servers they are?
How does it keep this information up to date?
How does the client find an entire replica set from a seed list,
and how does it respond to a stepdown, election, reconfiguration, or network error?

All drivers must answer these questions the same.
Or, where platforms' limitations require differences among drivers,
there must be as few answers as possible and each must be clearly explained in this spec.
Even in cases where several answers seem equally good, drivers must agree on one way to do it.

MongoDB users and driver authors benefit from having one way to discover and monitor servers.
Users can substantially understand their driver's behavior without inspecting its code or asking its author.
Driver authors can avoid subtle mistakes
when they take advantage of a design that has been well-considered, reviewed, and tested.

The server discovery and monitoring method is specified in four sections.
First, a client is `configured`_.
Second, it begins `monitoring`_ by calling ismaster on all servers.
(Multi-threaded and asynchronous monitoring is described first,
then single-threaded monitoring.)
Third, as ismaster calls are received
the client `parses them`_,
and fourth, it `updates its view of the topology`_.

Finally, this spec describes how `drivers update their topology view
in response to errors`_,
and includes generous implementation notes for driver authors.

This spec does not describe how a client selects a server for an operation;
that is the domain of the specs for Mongos High Availability
and for Read Preferences.
But there is a section describing
the `interaction between monitoring and server selection`_.

There is no discussion of driver architecture and data structures,
nor is there any specification of a user-facing API.
This spec is only concerned with the algorithm for monitoring the server topology.

Meta
----

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
`RFC 2119`_.

.. _RFC 2119: https://www.ietf.org/rfc/rfc2119.txt

Specification
-------------

General Requirements
''''''''''''''''''''

**Direct connections:**
A client MUST be able to connect to a single server of any type.
This includes querying hidden replica set members,
and connecting to uninitialized members (see `RSGhost`_) in order to run
"replSetInitiate".
Setting a read preference MUST NOT be necessary to connect to a secondary.
Of course,
the secondary will reject all operations done with the PRIMARY read preference
because the slaveOk bit is not set,
but the initial connection itself succeeds.
Drivers MAY allow direct connections to arbiters
(for example, to run administrative commands).

**Replica sets:**
A client MUST be able to discover an entire replica set from
a seed list containing one or more replica set members.
It MUST be able to continue monitoring the replica set
even when some members go down,
or when reconfigs add and remove members.
A client MUST be able to connect to a replica set
while there is no primary, or the primary is down.

**Mongos:**
A client MUST be able to connect to a set of mongoses
and monitor their availability and `round trip time`_.
This spec defines how mongoses are discovered and monitored,
but does not define which mongos is selected for a given operation.

**Master-slave:**
A client MUST be able to directly connect to a mongod begun with "--slave".
No additional master-slave features are described in this spec.

Terms
'''''

Server
``````

A mongod or mongos process.

Deployment
``````````

One or more servers:
either a standalone, a replica set, or one or more mongoses.

Topology
````````

The state of the deployment:
its type (standalone, replica set, or sharded),
which servers are up, what type of servers they are,
which is primary, and so on.

Client
``````

Driver code responsible for connecting to MongoDB.

Seed list
`````````

Server addresses provided to the client in its initial configuration,
for example from the `connection string`_.

Round trip time
```````````````

Also known as RTT.

The client's measurement of the duration of an ismaster call.
The round trip time is used to support the "secondaryAcceptableLatencyMS"
option in the Read Preferences spec.
Even though this measurement is called "ping time" in that spec,
`drivers MUST NOT use the "ping" command`_ to measure this duration.
`This spec does not mandate how round trip time is averaged`_.

ismaster outcome
````````````````

The result of an attempt to call the "ismaster" command on a server.
It consists of three elements:
a boolean indicating the success or failure of the attempt,
a document containing the command response (or null if it failed),
and the round trip time to execute the command (or null if it failed).

.. _checks:
.. _checking:
.. _checked:

check
`````

The client checks a server by attempting to call ismaster on it,
and recording the outcome.

.. _scans:
.. _rescan:

scan
````

The process of checking all servers in the deployment.

suitable
````````

A server is judged "suitable" for an operation if the client can use it.
For example, a write requires a standalone
(or the master of a master-slave set),
primary, or mongos,
and a read requires a server matching its read preference.

address
```````

The hostname or IP address, and port number, of a MongoDB server.

Data structures
'''''''''''''''

This spec uses a few data structures
to describe the client's view of the topology.
It must be emphasized that
a driver is free to implement the same behavior
using different data structures.
This spec uses these enums and structs in order to describe driver **behavior**,
not to mandate how a driver represents the topology,
nor to mandate an API.

Enums
`````

TopologyType
~~~~~~~~~~~~

Single, ReplicaSetNoPrimary, ReplicaSetWithPrimary, Sharded, or Unknown.

See `updating the TopologyDescription`_.

ServerType
~~~~~~~~~~

Standalone, Mongos,
PossiblePrimary, RSPrimary, RSSecondary, RSArbiter, RSOther, RSGhost,
or Unknown.

See `parsing an ismaster response`_.

.. note:: Single-threaded clients use the PossiblePrimary type
   to maintain proper `scanning order`_.
   Multi-threaded and asynchronous clients do not need this ServerType;
   it is synonymous with Unknown.

TopologyDescription
```````````````````

The client's representation of everything it knows about the deployment's topology.

Fields:

* type: a `TopologyType`_ enum value.
  The default is Unknown.
* setName: the replica set name. Default null.
* servers: a set of ServerDescription instances.
  Default contains one server: "localhost:27017", ServerType Unknown.
* compatible: a boolean.
  False if any server's wire protocol version range
  is incompatible with the client's.
  Default true.
* compatibilityError: a string.
  The error message if "compatible" is false, otherwise null.

Single-threaded drivers have an additional field:

* stale: true when the client needs to `rescan`_ soon.
  Initially true.

ServerDescription
`````````````````

The client's view of a single server,
based on the most recent `ismaster outcome`_.

Again, drivers may store this information however they choose;
this data structure is defined here
merely to describe the monitoring algorithm.

Fields:

* address: the hostname or IP, and the port number,
  that the client connects to.
  Note that this is **not** the server's ismaster.me field,
  in the case that the server reports an address different
  from the address the client uses.
* error: information about the last error related to this server. Default null.
* roundTripTime: the duration of the ismaster call. Default null.
* type: a `ServerType`_ enum value. Default Unknown.
* minWireVersion: the minimum wire protocol version supported by the server.
  Default 0.
* maxWireVersion: the maximum wire protocol version supported by the server.
  Default 0.
* hosts, passives, arbiters: Sets of addresses.
  This server's opinion of the replica set's members, if any.
  These `hostnames are normalized to lower-case`_.
  Default empty.
  The client `monitors all three types of servers`_ in a replica set.
* tags: map from string to string. Default empty.
* setName: string or null. Default null.
* primary: an address. This server's opinion of who the primary is.
  Default null.

"Passives" are priority-zero replica set members that cannot become primary.
The client treats them precisely the same as other members.

Single-threaded clients need an additional field
to implement the `single-threaded monitoring`_ algorithm:

* lastUpdateTime: when this server was last checked. Default "infinity ago".

Clients may find it useful to store additional fields
from the ismaster response,
such as:

* maxDocumentSize
* maxMessageSize
* maxWriteBatchSize

.. _configured:

Configuration
'''''''''''''

No breaking changes
```````````````````

This spec does not intend
to require any drivers to make breaking changes regarding
what configuration options are available,
how options are named,
or what combinations of options are allowed.

Initial TopologyDescription
```````````````````````````

The default values for `TopologyDescription`_ fields are described above.
Users may override the defaults as follows:

Initial Servers
~~~~~~~~~~~~~~~

The user MUST be able to set the initial servers list to a `seed list`_
of one or more addresses.

The hostname portion of each address MUST be normalized to lower-case.

Initial TopologyType
~~~~~~~~~~~~~~~~~~~~

The user MUST be able to set the initial TopologyType to Single or Unknown.

The user MAY be able to initialize it to ReplicaSetNoPrimary.
This provides the user a way to tell the client
it can only connect to replica set members.
Similarly the user MAY be able to initialize it to Sharded,
to connect only to mongoses.

The API for initializing TopologyType is not specified here.
Drivers might already have a convention, e.g. a single seed means Single,
a setName means ReplicaSetNoPrimary,
and a list of seeds means Unknown.
There are variations, however:
In the Java driver a single seed means Single,
but a **list** containing one seed means Unknown,
so it can transition to replica-set monitoring if the seed is discovered
to be a replica set member.
In contrast, PyMongo requires a non-null setName
in order to begin replica-set monitoring,
regardless of the number of seeds.
This spec does not imply existing driver APIs must change
as long as all the required features are somehow supported.

Initial setName
~~~~~~~~~~~~~~~

The user MUST be able to set the client's initial replica set name.
A driver MAY require the set name in order to connect to a replica set,
or it MAY be able to discover the replica set name as it connects.

Allowed configuration combinations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Drivers MUST enforce:

* TopologyType Single cannot be used with multiple seeds.
* If setName is not null, only TopologyType ReplicaSetNoPrimary,
  and possibly Single,
  are allowed.
  (See `verifying setName with TopologyType Single`_.)

heartbeatFrequencyMS
````````````````````

The interval between server `checks`_.
For multi-threaded and asynchronous drivers
it MUST default to 10 seconds and MAY be configurable.
For single-threaded drivers it MUST default to 60 seconds
and MUST be configurable.
It MUST be called heartbeatFrequencyMS
unless this breaks backwards compatibility.

(See `heartbeatFrequencyMS defaults to 10 seconds or 60 seconds`_
and `what's the point of periodic monitoring?`_)

socketCheckIntervalMS
`````````````````````

If a socket has not been used recently,
a single-threaded client MUST check it, by using it for an ismaster call,
before using it for any operation.
The default MUST be 5 seconds, and it MAY be configurable.

Only for single-threaded clients.

(See `single-threaded server selection`_,
`single-threaded server selection implementation`_,
and `what is the purpose of socketCheckIntervalMS?`_).

Client construction
'''''''''''''''''''

The client's constructor MUST NOT do any I/O.
This means that the constructor does not throw an exception
if servers are unavailable:
the topology is not yet known when the constructor returns.
Similarly if a server has an incompatible wire protocol version,
the constructor does not throw.
Instead, all subsequent operations on the client fail
as long as the error persists.

See `clients do no I/O in the constructor`_ for the justification.

Multi-threaded and asynchronous client construction
```````````````````````````````````````````````````

The constructor MAY start the monitors as background tasks
and return immediately.
Or the monitors MAY be started by some method separate from the constructor;
for example they MAY be started by some "initialize" method (by any name),
or on the first use of the client for an operation.

Single-threaded client construction
```````````````````````````````````

Single-threaded clients do no I/O in the constructor.
They MUST `scan`_ the servers on demand,
when the first operation is attempted.

Monitoring
''''''''''

The client monitors servers by `checking`_ them
roughly every `heartbeatFrequencyMS`_,
or in response to certain events.

The socket used to check a server MUST use the same
`connectTimeoutMS <http://docs.mongodb.org/manual/reference/connection-string/>`_
as regular sockets.
Multi-threaded clients SHOULD set monitoring sockets' socketTimeoutMS to the
connectTimeoutMS.
(See `socket timeout for monitoring is connectTimeoutMS`_.
Drivers MAY let users configure the timeouts for monitoring sockets
separately if necessary to preserve backwards compatibility.)

The client begins monitoring a server when:

* ... the client is initialized and begins monitoring each seed.
  See `initial servers`_.
* ... `updateRSWithoutPrimary`_ or `updateRSFromPrimary`_
  discovers new replica set members.

The following subsections specify how monitoring works,
first in multi-threaded or asynchronous clients,
and second in single-threaded clients.
This spec provides detailed requirements for monitoring
because it intends to make all drivers behave consistently.

.. _mt-monitoring:

Multi-threaded or asynchronous monitoring
`````````````````````````````````````````

(See also: `implementation notes for multi-threaded clients`_.)

Servers are monitored in parallel
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

All servers' monitors run independently, in parallel:
If some monitors block calling ismaster over slow connections,
other monitors MUST proceed unimpeded.

The natural implementation is a thread per server,
but the decision is left to the implementer.
(See `thread per server`_.)

Servers are monitored with dedicated sockets
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

`A monitor SHOULD NOT use the client's regular connection pool`_
to acquire a socket;
it uses a dedicated socket that does not count toward the pool's
maximum size.

Servers are checked periodically
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Each monitor `checks`_ its server and notifies the client of the outcome
so the client can update the TopologyDescription.

Checks SHOULD be scheduled every `heartbeatFrequencyMS`_;
a check MUST NOT run while a previous check is still in progress.

.. _request an immediate check:

Requesting an immediate check
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

At any time, the client can request that a monitor check its server immediately.
(For example, after a "not master" error. See `error handling`_.)
If the monitor is sleeping when this request arrives,
it MUST wake and check as soon as possible.
If an ismaster call is already in progress,
the request MUST be ignored.
If the previous check was less than `minHeartbeatFrequencyMS`_ ago,
the monitor MUST sleep until the minimum delay has passed,
then check the server.

Application operations are unblocked when a server is found
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Each time a check completes, threads waiting for a `suitable`_ server
are unblocked. Each unblocked thread MUST proceed if the new TopologyDescription
now contains a suitable server.

As an optimization, the client MAY leave threads blocked
if a check completes without detecting any change besides
`round trip time`_: no operation that was blocked will
be able to proceed anyway.

.. _st-monitoring:

Single-threaded monitoring
``````````````````````````

Scanning
~~~~~~~~

Single-threaded clients MUST `scan`_ all servers synchronously,
inline with regular application operations.
Before each operation, the client checks if `heartbeatFrequencyMS`_ has
passed since the previous scan; if so it scans all the servers before
selecting a server and performing the operation.

.. _request a scan:

Requesting a scan
~~~~~~~~~~~~~~~~~

A single-threaded client can mark its TopologyDescription "stale".
(See `error handling`_ for the circumstances.)
When the TopologyDescription is stale,
the client performs a scan before the next operation,
then clears the "stale" flag.

Scanning order
~~~~~~~~~~~~~~

If the topology is a replica set,
the client attempts to contact the primary as soon as possible
to get an authoritative list of members.
Otherwise, the client attempts to check all members it knows of,
in order from the least-recently to the most-recently checked.

When all servers have been checked the scan is complete.
New servers discovered **during** the scan
MUST be checked before the scan is complete.
Sometimes servers are removed during a scan
so they are not checked, depending on the order of events.

The scanning order is expressed in this pseudocode::

    scanStartTime = now()

    while true:
        serversToCheck = all servers with lastUpdateTime before scanStartTime
        if no serversToCheck:
            # This scan has completed.
            break

        if a server in serversToCheck is RSPrimary:
            check it
        else if there is a PossiblePrimary:
            check it
        else if any servers are not of type Unknown or RSGhost:
            check the one with the oldest lastUpdateTime
            if several servers have the same lastUpdateTime, choose one at random
        else:
            check the Unknown or RSGhost server with the oldest lastUpdateTime
            if several servers have the same lastUpdateTime, choose one at random

This algorithm might be better understood with an example:

#. The client is configured with one seed and TopologyType Unknown.
   It begins a scan.
#. When it checks the seed, it discovers a secondary.
#. The secondary's ismaster response includes the "primary" field
   with the address of the server that the secondary thinks is primary.
#. The client creates a ServerDescription with that address,
   type PossiblePrimary, and lastUpdateTime "infinity ago".
   (See `updateRSWithoutPrimary`_.)
#. On the next iteration, there is still no RSPrimary,
   so the new PossiblePrimary is the top-priority server to check.
#. The PossiblePrimary is checked and replaced with an RSPrimary.
   The client has now acquired an authoritative host list.
   Any new hosts in the list are added to the TopologyDescription
   with lastUpdateTime "infinity ago".
   (See `updateRSFromPrimary`_.)
#. The client continues scanning until all known hosts have been checked.

Another common case might be scanning a pool of mongoses.
When the client first scans its seed list,
they all have the default lastUpdateTime "infinity ago",
so it scans them in random order.
This randomness provides some load-balancing if many clients start at once.
A client's subsequent scans of the mongoses
are always in the same order,
since their lastUpdateTimes are always in the same order
by the time a scan ends.

minHeartbeatFrequencyMS
```````````````````````

If a client frequently rechecks a server,
it MUST wait at least minHeartbeatFrequencyMS milliseconds
since the previous check to avoid pointless effort.
This value MUST be 10 ms, and it MUST NOT be configurable.
(See `no knobs`_.)

.. _parses them:

Parsing an ismaster response
''''''''''''''''''''''''''''

The client represents its view of each server with a `ServerDescription`_.
Each time the client `checks`_ a server,
it replaces its description of that server with a new one.

ServerDescriptions are created from ismaster outcomes as follows:

type
````

The new ServerDescription's type field is set to a `ServerType`_.
Note that these states do **not** exactly correspond to
`replica set member states
<http://docs.mongodb.org/manual/reference/replica-states/>`_.
For example, some replica set member states like STARTUP and RECOVERING
are identical from the client's perspective, so they are merged into "RSOther".
Additionally, states like Standalone and Mongos
are not replica set member states at all.

+-------------------+---------------------------------------------------------------+
| State             | Symptoms                                                      |
+===================+===============================================================+
| Unknown           | Initial, or after a network error or failed ismaster call,    |
|                   | or "ok: 1" not in ismaster response.                          |
+-------------------+---------------------------------------------------------------+
| Standalone        | No "msg: isdbgrid", no setName, and no "isreplicaset: true".  |
+-------------------+---------------------------------------------------------------+
| Mongos            | "msg: isdbgrid".                                              |
+-------------------+---------------------------------------------------------------+
| PossiblePrimary   | Not yet checked, but another member thinks it is the primary. |
+-------------------+---------------------------------------------------------------+
| RSPrimary         | "ismaster: true", "setName" in response.                      |
+-------------------+---------------------------------------------------------------+
| RSSecondary       | "secondary: true", "setName" in response.                     |
+-------------------+---------------------------------------------------------------+
| RSArbiter         | "arbiterOnly: true", "setName" in response.                   |
+-------------------+---------------------------------------------------------------+
| RSOther           | "setName" in response, "hidden: true" or not primary,         |
|                   | secondary, nor arbiter.                                       |
+-------------------+---------------------------------------------------------------+
| RSGhost           | "isreplicaset: true" in response.                             |
+-------------------+---------------------------------------------------------------+

A server can transition from any state to any other.
For example, an administrator could shut down a secondary
and bring up a mongos in its place.

.. _RSGhost:

RSGhost and RSOther
~~~~~~~~~~~~~~~~~~~

The client MUST monitor replica set members
even when they cannot be queried.
These members are in state RSGhost or RSOther.

**RSGhost** members occur in at least three situations:

* briefly during server startup,
* in an uninitialized replica set,
* or when the server is shunned (removed from the replica set config).

An RSGhost server has no hosts list nor setName.
Therefore the client MUST NOT attempt to use its hosts list
nor check its setName
(see `JAVA-1161 <https://jira.mongodb.org/browse/JAVA-1161>`_
or `CSHARP-671 <https://jira.mongodb.org/browse/CSHARP-671>`_.)
However, the client MUST keep the RSGhost member in its TopologyDescription,
in case the client's only hope for staying connected to the replica set
is that this member will transition to a more useful state.

RSGhosts may report their setNames in the future
(see `SERVER-13458 <https://jira.mongodb.org/browse/SERVER-13458>`_).
For simplicity, this is the rule:
any server is an RSGhost that reports "isreplicaset: true".

Non-ghost replica set members have reported their setNames
since MongoDB 1.6.2.
See `only support replica set members running MongoDB 1.6.2 or later`_.

.. note:: The Java driver does not have a separate state for RSGhost;
   it is an RSOther server with no hosts list.

**RSOther** servers may be hidden, starting up, or recovering.
They cannot be queried, but their hosts lists are useful
for discovering the current replica set configuration.

If a `hidden member <http://docs.mongodb.org/manual/core/replica-set-hidden-member/>`_
is provided as a seed,
the client can use it to find the primary.
Since the hidden member does not appear in the primary's host list,
it will be removed once the primary is checked.

error
`````

If the client experiences any error when checking a server,
it stores error information in the ServerDescription's error field.

roundTripTime
`````````````

Drivers MUST record the server's `round trip time`_ (RTT)
after each successful call to ismaster,
but `this spec does not mandate how round trip time is averaged`_.

If an ismaster call fails, the RTT is not updated.
Furthermore, while a server's type is Unknown its RTT is null,
and if it changes from a known type to Unknown its RTT is set to null.
However, if it changes from one known type to another
(e.g. from RSPrimary to RSSecondary) its RTT is updated normally,
not set to null nor restarted from scratch.

Hostnames are normalized to lower-case
``````````````````````````````````````

The same as with seeds provided in the initial configuration,
all hostnames in the ismaster response's "hosts", "passives", and "arbiters"
entries must be lower-cased.

This prevents unecessary work rediscovering a server
if a seed "A" is provided and the server
responds that "a" is in the replica set.

`RFC 4343 <http://tools.ietf.org/html/rfc4343>`_:

    Domain Name System (DNS) names are "case insensitive".

Other ServerDescription fields
``````````````````````````````

Other required and optional fields
defined in the `ServerDescription`_ data structure
are parsed from the ismaster response in the obvious way.

Network error when calling ismaster
'''''''''''''''''''''''''''''''''''

When a server `check`_ fails due to a network error,
the client SHOULD clear its connection pool for the server:
if the monitor's socket is bad it is likely that all are.
(See `JAVA-1252 <https://jira.mongodb.org/browse/JAVA-1252>`_.)

Once a server is connected, the client MUST change its type
to Unknown
only after it has retried the server once.

In this pseudocode, "description" is the prior ServerDescription::

    def checkServer(description):
        try:
            call ismaster
            return new ServerDescription
        except NetworkError as e0:
            clear connection pool for the server

            if description.type is Unknown or PossiblePrimary:
                # Failed on first try to reach this server, give up.
                return new ServerDescription with type=Unknown, error=e0
            else:
                # We've been connected to this server in the past, retry once.
                try:
                    call ismaster
                    return new ServerDescription
                except NetworkError as e1:
                    return new ServerDescription with type=Unknown, error=e1

(See `retry ismaster calls once`_ and
`JAVA-1159 <https://jira.mongodb.org/browse/JAVA-1159>`_.)

.. _updates its view of the topology:

Updating the TopologyDescription
''''''''''''''''''''''''''''''''

Each time the client checks a server,
it processes the outcome (successful or not)
to create a `ServerDescription`_,
and then it processes the ServerDescription to update its `TopologyDescription`_.

The TopologyDescription's `TopologyType`_ influences
how the ServerDescription is processed.
The following subsection
specifies how the client updates its TopologyDescription
when the TopologyType is Single.
The next subsection treats the other types.

TopologyType Single
```````````````````

The TopologyDescription's type was initialized as Single
and remains Single forever.
There is always one ServerDescription in TopologyDescription.servers.

Whenever the client checks a server (successfully or not)
the ServerDescription in TopologyDescription.servers
is replaced with the new ServerDescription.

.. _updates the "compatible" and "compatibilityError" fields:

If the server's wire protocol version range overlaps with the client's,
TopologyDescription.compatible is set to true.
Otherwise it is set to false
and the "compatibilityError" field is filled out like,
"Server at HOST:PORT uses wire protocol versions SERVER_MIN through SERVER_MAX,
but CLIENT only supports CLIENT_MIN through CLIENT_MAX."

Verifying setName with TopologyType Single
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A client MAY allow the user to supply a setName
with an initial TopologyType of Single.
In this case, if the ServerDescription's setName is null or wrong,
the client MUST throw an error on every operation.

Other TopologyTypes
```````````````````

If the TopologyType is **not** Single, there can be one or more seeds.

Whenever a client completes an ismaster call,
it creates a new ServerDescription with the proper `ServerType`_.
It replaces the server's previous description in TopologyDescription.servers
with the new one.

If any server's wire protocol version range does not overlap with the client's,
the client `updates the "compatible" and "compatibilityError" fields`_
as described above for TopologyType Single.
Otherwise "compatible" is set to true.

It is possible for a multi-threaded client to receive an ismaster outcome
from a server after the server has been removed from the TopologyDescription.
For example, a monitor begins checking a server "A",
then a different monitor receives a response from the primary
claiming that "A" has been removed from the replica set,
so the client removes "A" from the TopologyDescription.
Then, the check of server "A" completes.

In all cases, the client MUST ignore ismaster outcomes from servers
that are not in the TopologyDescription.

The following subsections explain in detail what actions the client takes
after replacing the ServerDescription.

TopologyType table
~~~~~~~~~~~~~~~~~~

The new ServerDescription's type is the vertical axis,
and the current TopologyType is the horizontal.
Where a ServerType and a TopologyType intersect,
the table shows what action the client takes.

"no-op" means,
do nothing **after** replacing the server's old description
with the new one.

.. csv-table::
  :header-rows: 1
  :stub-columns: 1

  ,TopologyType Unknown,TopologyType Sharded,TopologyType ReplicaSetNoPrimary,TopologyType ReplicaSetWithPrimary
  ServerType Unknown,no-op,no-op,no-op,`checkIfHasPrimary`_
  ServerType Standalone,`updateUnknownWithStandalone`_,`remove`_,`remove`_,`remove`_ and `checkIfHasPrimary`_
  ServerType Mongos,Set type to TopologyType Sharded,no-op,`remove`_,`remove`_ and `checkIfHasPrimary`_
  ServerType RSPrimary,`updateRSFromPrimary`_,`remove`_, `updateRSFromPrimary`_,`updateRSFromPrimary`_
  ServerType RSSecondary,Set type to ReplicaSetNoPrimary then `updateRSWithoutPrimary`_,`remove`_,`updateRSWithoutPrimary`_,`updateRSWithPrimaryFromMember`_
  ServerType RSArbiter,Set type to ReplicaSetNoPrimary then `updateRSWithoutPrimary`_,`remove`_,`updateRSWithoutPrimary`_,`updateRSWithPrimaryFromMember`_
  ServerType RSOther,Set type to ReplicaSetNoPrimary then `updateRSWithoutPrimary`_,`remove`_,`updateRSWithoutPrimary`_,`updateRSWithPrimaryFromMember`_
  ServerType RSGhost,no-op [#]_,`remove`_,no-op,`checkIfHasPrimary`_

.. [#] `TopologyType remains Unknown when an RSGhost is discovered`_.

TopologyType explanations
~~~~~~~~~~~~~~~~~~~~~~~~~

This subsection complements the `TopologyType table`_
with prose explanations of the TopologyTypes (besides Single).

TopologyType Unknown
  A starting state.

  **Actions**:

  * If the incoming ServerType is Unknown (that is, the ismaster call failed),
    keep the server in TopologyDescription.servers.
    The TopologyType remains Unknown.
  * The `TopologyType remains Unknown when an RSGhost is discovered`_, too.
  * If the type is Standalone, run `updateUnknownWithStandalone`_.
  * If the type is Mongos, set the TopologyType to Sharded.
  * If the type is RSPrimary, record its setName
    and call `updateRSFromPrimary`_.
  * If the type is RSSecondary, RSArbiter or RSOther, record its setName,
    set the TopologyType to ReplicaSetNoPrimary,
    and call `updateRSWithoutPrimary`_.

TopologyType Sharded
  A steady state. Connected to one or more mongoses.

  **Actions**:

  * If the server is Unknown or Mongos, keep it.
  * Remove others.

TopologyType ReplicaSetNoPrimary
  A starting state.
  The topology is definitely a replica set,
  but no primary is known.

  **Actions**:

  * Keep Unknown servers.
  * Keep RSGhost servers: they are members of some replica set,
    perhaps this one, and may recover.
    (See `RSGhost and RSOther`_.)
  * Remove any Standalones or Mongoses.
  * If the type is RSPrimary call `updateRSFromPrimary`_.
  * If the type is RSSecondary, RSArbiter or RSOther,
    run `updateRSWithoutPrimary`_.

TopologyType ReplicaSetWithPrimary
  A steady state. The primary is known.

  **Actions**:

  * If the server type is Unknown, keep it,
    and run `checkIfHasPrimary`_.
  * Keep RSGhost servers: they are members of some replica set,
    perhaps this one, and may recover.
    (See `RSGhost and RSOther`_.)
    Run `checkIfHasPrimary`_.
  * Remove any Standalones or Mongoses
    and run `checkIfHasPrimary`_.
  * If the type is RSPrimary run `updateRSFromPrimary`_.
  * If the type is RSSecondary, RSArbiter or RSOther,
    run `updateRSWithPrimaryFromMember`_.

Actions
~~~~~~~

.. _updateUnknownWithStandalone:

updateUnknownWithStandalone
  This subroutine is executed
  with the ServerDescription from Standalone (including a slave)
  when the TopologyType is Unkown::

    if description.address not in topologyDescription.servers:
        return

    if settings.seeds has one seed:
        topologyDescription.type = Single
    else:
        remove this server from topologyDescription and stop monitoring it

  See `TopologyType remains Unknown when one of the seeds is a Standalone`_.

.. _updateRSWithoutPrimary:

updateRSWithoutPrimary
  This subroutine is executed
  with the ServerDescription from any replica set member
  when the TopologyType is ReplicaSetNoPrimary::

    if description.address not in topologyDescription.servers:
        return

    if topologyDescription.setName is null:
        topologyDescription.setName = description.setName

    else if topologyDescription.setName != description.setName:
        remove this server from topologyDescription and stop monitoring it
        return

    for each address in description's "hosts", "passives", and "arbiters":
        if address is not in topologyDescription.servers:
            add new default ServerDescription of type "Unknown"
            begin monitoring the new server

    if description.primary is not null:
        find the ServerDescription in topologyDescription.servers whose
        address equals description.primary

        if its type is Unknown, change its type to PossiblePrimary

  Unlike `updateRSFromPrimary`_,
  this subroutine does **not** remove any servers from the TopologyDescription.

  The special handling of description.primary
  ensures that a single-threaded client
  `scans`_ the possible primary before other members.

  See `replica set monitoring with and without a primary`_.

.. _updateRSWithPrimaryFromMember:

updateRSWithPrimaryFromMember
  This subroutine is executed
  with the ServerDescription from any replica set member besides the primary
  when the TopologyType is ReplicaSetWithPrimary::

    if description.address not in topologyDescription.servers:
        # While we were checking this server, another thread heard from the
        # primary that this server is not in the replica set.
        return

    # SetName is never null here.
    if topologyDescription.setName != description.setName:
        remove this server from topologyDescription and stop monitoring it

    # Had this member been the primary?
    if there is no primary in topologyDescription.servers:
        topologyDescription.type = ReplicaSetNoPrimary

        if description.primary is not null:
            find the ServerDescription in topologyDescription.servers whose
            address equals description.primary

            if its type is Unknown, change its type to PossiblePrimary

  The special handling of description.primary
  ensures that a single-threaded client
  `scans`_ the possible primary before other members.

.. _updateRSFromPrimary:

updateRSFromPrimary
  This subroutine is executed with the ServerDescription from a primary::

    if description.address not in topologyDescription.servers:
        return

    if topologyDescription.setName is null:
        topologyDescription.setName = description.setName

    else if topologyDescription.setName != description.setName:
        # We found a primary but it doesn't have the setName
        # provided by the user or previously discovered.
        remove this server from topologyDescription and stop monitoring it
        checkIfHasPrimary()
        return

    for each server in topologyDescription.servers:
        if server.address != description.address:
            if server.type is RSPrimary:
                # See note below about invalidating an old primary.
                replace the server with a default ServerDescription of type "Unknown"

    for each address in description's "hosts", "passives", and "arbiters":
        if address is not in topologyDescription.servers:
            add new default ServerDescription of type "Unknown"
            begin monitoring the new server

    for each server in topologyDescription.servers:
        if server.address not in description's "hosts", "passives", or "arbiters":
            remove the server and stop monitoring it

    checkIfHasPrimary()

  A note on invalidating the old primary:
  when a new primary is discovered,
  the client finds the previous primary (there should be none or one)
  and replaces its description
  with a default ServerDescription of type "Unknown."
  A multi-threaded client MUST check that server as soon as possible.
  (The Monitor provides a "request refresh" feature for this purpose,
  see `multi-threaded or asynchronous monitoring`_.)

  The client SHOULD clear its connection pool for the old primary, too:
  the connections are all bad because the old primary has closed its sockets.

  See `replica set monitoring with and without a primary`_.

.. _checkIfHasPrimary:

checkIfHasPrimary
  Set TopologyType to ReplicaSetWithPrimary if there is a primary
  in TopologyDescription.servers, otherwise set it to ReplicaSetNoPrimary.

  For example, if the TopologyType is ReplicaSetWithPrimary
  and the client is processing a new ServerDescription of type Unknown,
  that could mean the primary just disconnected,
  so checkIfHasPrimary must run to check if the TopologyType should become
  ReplicaSetNoPrimary.

  Another example is if the client first reaches the primary via its external
  IP, but the response's host list includes only internal IPs.
  In that case the client adds the primary's internal IP to the
  TopologyDescription and begins monitoring it, and removes the external IP.
  Right after removing the external IP from the description,
  the TopologyType MUST be ReplicaSetNoPrimary, since no primary is
  available at this moment.

.. _remove:

remove
  Remove the server from TopologyDescription.servers and stop monitoring it.

  In multi-threaded clients, a monitor may be currently checking this server
  and may not immediately abort.
  Once the check completes, this server's ismaster outcome MUST be ignored,
  and the monitor SHOULD halt.

Reference implementation
~~~~~~~~~~~~~~~~~~~~~~~~

The Java driver's
`MultiClusterServer.onChange
<https://github.com/mongodb/mongo-java-driver/blob/3.0.x/driver/src/main/org/mongodb/connection/MultiServerCluster.java#L112>`_
is a reference implementation of all logic
for TopologyTypes besides Single.

.. _interaction between monitoring and server selection:

Server selection
''''''''''''''''

The client uses the TopologyDescription to select a server `suitable`_
for a given operation.

.. note::
   Server selection criteria are not this spec's domain.
   They are described in the specs for
   Read Preferences and Mongos High Availability.
   However, when TopologyType is Single
   the client should always select the current server,
   and for other TopologyTypes
   it seems sane to select servers only of these types:
   Standalone, Mongos, RSPrimary, RSSecondary.

.. _mt-server-selection:

.. _multi-threaded server selection:

Multi-threaded or asynchronous server selection
```````````````````````````````````````````````

A driver that uses `multi-threaded or asynchronous monitoring`_
MUST unblock waiting operations
as soon as a suitable server is found,
rather than waiting for all servers to be `checked`_.

For example, if the client is discovering a replica set
and the application attempts a write operation,
the write operation MUST proceed as soon as there is an RSPrimary,
rather than blocking until the client has checked all members.

While no suitable server is available for an operation,
`the client MUST re-check all servers every minHeartbeatFrequencyMS`_.
(See `requesting an immediate check`_
and `multi-threaded server selection implementation`_.)

Once some maximum wait time has passed, the client MUST abort the operation
and throw an exception.
The client SHOULD use the error fields of all ServerDescriptions
to determine a useful error class and message if possible.

.. _st-server-selection:

Single-threaded server selection
````````````````````````````````

A driver that uses `single-threaded monitoring`_
selects a server for an operation like so:

#. If the TopologyDescription has any `suitable`_ servers,
   iterate them in random order. For each suitable server:

    - If the client has an open socket to the server
      that has been used more recently than `socketCheckIntervalMS`_, use it.
      Server selection is complete.
      (`Network error when reading or writing`_ specifies what happens
      if the operation fails.)
    - Otherwise, if there is no socket to the server, attempt to open one.
      If this fails, mark the server "Unknown" and continue iterating the
      list of suitable servers.
    - Otherwise, if the client has an open socket to the server
      but the socket has not been used in `socketCheckIntervalMS`_,
      the client MUST check the socket by calling ismaster on it.
    - If the ismaster call succeeds the client MUST update the
      ServerDescription. If it fails the client MUST mark the server Unknown.
    - If the ServerDescription is still suitable for the operation,
      use the server.
      Server selection is complete.
    - If it is not (because the server changed type, or
      because the ismaster call failed and the ServerType is Unknown),
      continue iterating the list of suitable servers.

#. If some maximum wait time has passed, throw an exception.
   The client SHOULD use the error fields of all ServerDescriptions
   to determine a useful error class and message if possible.
#. `Scan`_ the servers.
   (See `single-threaded monitoring`_ for the scan algorithm).
#. Go to #1, to see if the scan discovered suitable servers. 

See the pseudocode for `single-threaded server selection implementation`_,
and the justification for
`what is the purpose of socketCheckIntervalMS?`_.

.. _drivers update their topology view in response to errors:

Error handling
''''''''''''''

This section is about errors when reading or writing to a server.
For errors when checking servers, see `network error when calling ismaster`_.

Network error when reading or writing
`````````````````````````````````````

When an application operation fails because of
any network error besides a socket timeout,
the client MUST replace the server's description
with a default ServerDescription of type Unknown,
and fill the ServerDescription's error field with useful information.

The specific operation that discovered the error
MUST abort and raise an exception if it was a write.
It MAY be retried if it was a read.
(The Read Preferences spec includes retry rules for reads.)

The client SHOULD clear its connection pool for the server:
if one socket is bad, it is likely that all are.

A multi-threaded client MUST NOT request an immediate check of the server,
and a single-threaded client MUST NOT request a scan before the next operation:
since application sockets are used frequently, a network error likely means
the server has just become unavailable,
so an immediate refresh is likely to get a network error, too.

The server will not remain Unknown forever.
It will be refreshed by the next periodic check or,
if an application operation needs the server sooner than that,
then a re-check will be triggered by the `server selection`_ algorithm.

"not master" and "node is recovering"
`````````````````````````````````````

These errors are detected from a getLastError response,
write command response, or query response. Clients MUST consider a server
error to be a "not master" error if the server's error contains the
substring "not master" or "node is recovering" anywhere within the error
message::

    def is_notmaster(err_message):
        return ("not master" in err_message
                or "node is recovering" in err_message)

    def parse_gle(response):
        if "err" in response:
            if is_notmaster(response["err"]):
                handle_not_master_or_recovering(response["err"])

    # Parse response to any command besides getLastError.
    def parse_command_response(response):
        if not response["ok"]:
            if is_notmaster(response["errmsg"]):
                handle_not_master_or_recovering(response["errmsg"])

    def parse_query_response(response):
        if the "QueryFailure" bit is set in response flags:
            if is_notmaster(response["$err"]):
                handle_not_master_or_recovering(response["$err"])

    def handle_not_master_or_recovering(message):
        replace server's description with
        new ServerDescription(type=Unknown, error=message)

        request immediate check (multi-threaded),
        or mark topologyDescription "stale" (single-threaded)

        clear connection pool for server

See the test scenario called
"parsing 'not master' and 'node is recovering' errors"
for example response documents.

When the client sees a "not master" or "node is recovering" error
it MUST replace the server's description
with a default ServerDescription of type Unknown.
It MUST store useful information in the new ServerDescription's error field,
including the error message from the server.
Multi-threaded and asynchronous clients MUST `request an immediate check`_
of the server,
and single-threaded clients MUST `request a scan`_ before the next operation.
Unlike in the "network error" scenario above,
a "not master" or "node is recovering" error means the server is available
but the client is wrong about its type,
thus an immediate re-check is likely to provide useful information.

The client SHOULD clear its connection pool for the server.

(See `when does a client see "not master" or "node is recovering"?`_.
and `use error messages to detect "not master" and "node is recovering"`_.)

Implementation notes
''''''''''''''''''''

This section intends to provide generous guidance to driver authors.
It is complementary to the reference implementations.
Words like "should", "may", and so on are used more casually here.

Documentation
`````````````

Giant seed lists
~~~~~~~~~~~~~~~~

Drivers' manuals should warn against huge seed lists,
since it will slow initialization for single-threaded clients
and generate load for multi-threaded and asynchronous drivers.

.. _implementation notes for multi-threaded clients:

Multi-threaded
``````````````

Monitor thread
~~~~~~~~~~~~~~

Most platforms can use an event object
to control the monitor thread.
The event API here is assumed to be like the standard `Python Event
<https://docs.python.org/2/library/threading.html#event-objects>`_.
`heartbeatFrequencyMS`_ is configurable,
`minHeartbeatFrequencyMS`_ is always 10 milliseconds::

    def run():
        while this monitor is not stopped:
            check server and create newServerDescription
            onServerDescriptionChanged(newServerDescription)

            start = gettime()

            # Can be awakened by requestCheck().
            event.wait(heartbeatFrequencyMS)
            event.clear()

            waitTime = gettime() - start
            if waitTime < minHeartbeatFrequencyMS:
                # Cannot be awakened.
                sleep(minHeartbeatFrequencyMS - waitTime)

`Requesting an immediate check`_::

    def requestCheck():
        event.set()

Immutable data
~~~~~~~~~~~~~~

Multi-threaded drivers should treat
ServerDescriptions and
TopologyDescriptions as immutable:
the client replaces them, rather than modifying them,
in response to new information about the topology.
Thus readers of these data structures
can simply acquire a reference to the current one
and read it, without holding a lock that would block a monitor
from making further updates.

Process one ismaster outcome at a time
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Although servers are checked in parallel,
the function that actually creates the new TopologyDescription
should be synchronized so only one thread can run it at a time.

.. _onServerDescriptionChanged:

Replacing the TopologyDescription
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Drivers may use the following pseudocode to guide
their implementation.
The client object has a lock and a condition variable.
It uses the lock to ensure that only one new ServerDescription is processed
at a time.
Once the client has taken the lock it must do no I/O::

    def onServerDescriptionChanged(server):
        # "server" is the new ServerDescription.

        # This thread cannot do any I/O until the lock is released.
        client.lock.acquire()

        if server.address not in client.topologyDescription.servers:
            # The server was once in the topologyDescription, otherwise
            # we wouldn't have been monitoring it, but an intervening
            # state-change removed it. E.g., we got a host list from
            # the primary that didn't include this server.
            client.lock.release()
            return

        newTopologyDescription = client.topologyDescription.copy()

        # Replace server's previous description.
        address = server.address
        newTopologyDescription.servers[address] = server

        take any additional actions,
        depending on the TopologyType and server...

        # Replace TopologyDescription and notify waiters.
        client.topologyDescription = newTopologyDescription
        client.condition.notifyAll()
        client.lock.release()

.. https://github.com/mongodb/mongo-java-driver/blob/5fb47a3bf86c56ed949ce49258a351773f716d07/src/main/com/mongodb/BaseCluster.java#L160

Notifying the condition unblocks waiters in `getServer`_.

.. note::
   The Java driver uses a CountDownLatch instead of a condition variable,
   and it atomically swaps the old and new CountDownLatches
   so it does not need "client.lock".
   It does, however, use a lock to ensure that only one thread runs
   onServerDescriptionChanged at a time.

.. _getServer:

Multi-threaded server selection implementation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A function is sketched here
to fit this spec's requirements about
`multi-threaded server selection`_.
The function wakes all monitors every minHeartbeatFrequencyMS
until a suitable server is found or a timeout is reached.
The timeout is called "serverWaitTime" here,
but this spec does not define what it is called or how it is configured::

    def getServer(criteria, serverWaitTime):
        client.lock.acquire()

        now = gettime()
        endTime = now + serverWaitTime
        servers = all servers in client.topologyDescription matching criteria

        while true:
            # We check wire protocol compatibility with all known servers,
            # not just suitable ones that match the criteria.
            if not client.topologyDescription.compatible:
                client.lock.release()
                throw error(client.topologyDescription.compatibilityError)

            if servers is not empty:
                client.lock.release()
                return a random server from servers

            # No suitable servers.
            if now after endTime:

                # TODO: more error diagnostics.
                # TODO: logging, with some severity levels: ERROR, WARNING, INFO, DEBUG?
                # E.g., if state is ReplicaSet but every server is Unknown,
                # and the host list is non-empty, and doesn't intersect with settings.seeds,
                # the set is probably configured with internal hostnames or IPs
                # and we're connecting from outside.
                # Or if state is ReplicaSet and topologyDescription.servers is empty,
                # we have the wrong setName.
                # Include topologyDescription's stringification in exception msg.

                # Sketch of error message logic.
                if there are any known servers:
                    error_message = "No servers are suitable for " + criteria
                else if all ServerDescriptions' errors are the same:
                    error_message = a ServerDescription.error value
                else:
                    error_message = ', '.join(all ServerDescriptions' errors)

                client.lock.release()
                throw error(error_message)

            request that all monitors check immediately

            # Wait for a new TopologyDescription. condition.wait() releases
            # client.lock while waiting and reacquires it before returning.
            timeout = endTime - now
            client.condition.wait(min(serverWaitTime, minHeartbeatFrequencyMS))

            # A new TopologyDescription was published, or we timed out.
            now = gettime()
            servers = all servers in client.topologyDescription matching criteria

While a thread is waiting on client.condition, it is awakened early
whenever a check completes. See `onServerDescriptionChanged`_.

Single-threaded
```````````````

Single-threaded server selection implementation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Pseudocode for `single-threaded server selection`_::

    def getServer(criteria, serverWaitTime):
        now = gettime()
        endTime = now + serverWaitTime
        servers = all servers in client.topologyDescription matching criteria

        while true:
            # We check wire protocol compatibility with all known servers,
            # not just suitable ones that match the criteria.
            if not client.topologyDescription.compatible:
                throw error(client.topologyDescription.compatibilityError)

            for server in randomized(servers):
                if the pool has a socket for the server:
                    if the socket has not been used in socketCheckIntervalMS:
                        try:
                            call ismaster with socket
                            create new ServerDescription with ismaster response
                        except NetworkError:
                            close socket
                            replace ServerDescription with new one of type Unknown
    
                        if new ServerDescription does not match criteria:
                            # Continue looping over the randomized servers.
                            continue
                else:
                    try:
                        open new socket to the server
                    except NetworkError:
                        replace ServerDescription with new one of type Unknown
                        continue

                # Success.
                return socket

            if now after endTime:
                # Sketch of error message logic.
                if there are any known servers:
                    error_message = "No servers are suitable for " + criteria
                else if all ServerDescriptions' errors are the same:
                    error_message = a ServerDescription.error value
                else:
                    error_message = ', '.join(all ServerDescriptions' errors)

                throw error(error_message)

            rescan
            servers = all servers in client.topologyDescription matching criteria

This spec does not define "serverWaitTime".

Rationale
---------

Clients do no I/O in the constructor
''''''''''''''''''''''''''''''''''''

An alternative proposal was to distinguish between "discovery" and "monitoring".
When discovery begins, the client checks all its seeds,
and discovery is complete once all servers have been checked,
or after some maximum time.
Application operations cannot proceed until discovery is complete.

If the discovery phase is distinct,
then single- and multi-threaded drivers
could accomplish discovery in the constructor,
and throw an exception from the constructor
if the deployment is unavailable or misconfigured.
This is consistent with prior behavior for many drivers.
It will surprise some users that the constructor now succeeds,
but all operations fail.

Similarly for misconfigured seed lists:
the client may discover a mix of mongoses and standalones,
or find multiple replica set names.
It may surprise some users that the constructor succeeds
and the client attempts to proceed with a compatible subset of the deployment.

Nevertheless, this spec prohibits I/O in the constructor
for the following reasons:

Common case
```````````

In the common case, the deployment is available and usable.
This spec favors allowing operations to proceed as soon as possible
in the common case,
at the cost of surprising behavior in uncommon cases.

Simplicity
``````````

It is simpler to omit a special discovery phase
and treat all server `checks`_ the same.

Consistency
```````````

Asynchronous clients cannot do I/O in a constructor,
so it is consistent to prohibit I/O in other clients' constructors as well.

Restarts
````````

If clients can be constructed when the deployment is in some states
but not in other states,
it leads to an unfortunate scenario:
When the deployment is passing through a strange state,
long-running clients may keep working,
but any clients restarted during this period fail.

Say an administrator changes one replica set member's setName.
Clients that are already constructed remove the bad member and stay usable,
but if any client is restarted its constructor fails.
Web servers that dynamically adjust their process pools
will show particularly undesirable behavior.

heartbeatFrequencyMS defaults to 10 seconds or 60 seconds
'''''''''''''''''''''''''''''''''''''''''''''''''''''''''

Many drivers have different values. The time has come to standardize.
Lacking a rigorous methodology for calculating the best frequency,
this spec chooses 10 seconds for multi-threaded or asynchronous drivers
because some already use that value.
It MAY be configurable, but it need not be.

Because scanning has a greater impact on
the performance of single-threaded drivers,
they MUST default to a longer frequency (60 seconds)
and the frequency MUST be configurable.

An alternative is to check servers less and less frequently
the longer they remain unchanged.
This idea is rejected because
it is a goal of this spec to answer questions about monitoring such as,

* "How rapidly can I rotate a replica set to a new set of hosts?"
* "How soon after I add a secondary will query load be rebalanced?"
* "How soon will a client notice a change in round trip time, or tags?"

Having a constant monitoring frequency allows us to answer these questions
simply and definitively.
Losing the ability to answer these questions is not worth
any minor gain in efficiency from a more complex scheduling method.

The client MUST re-check all servers every minHeartbeatFrequencyMS
''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''

While an application is waiting to do an operation
for which there is no suitable server,
a multi-threaded client MUST re-check all servers very frequently.
The slight cost is worthwhile in many scenarios. For example:

#. A client and a MongoDB server are started simultaneously.
#. The client checks the server before it begins listening,
   so the check fails.
#. The client waits in `getServer`_ for the topology to change.

In this state, the client should check the server very frequently,
to give it ample opportunity to connect to the server before
timing out in getServer.

What is the purpose of socketCheckIntervalMS?
'''''''''''''''''''''''''''''''''''''''''''''

Single-threaded clients need to make a compromise:
if they check servers too frequently it slows down regular operations,
but if they check too rarely they cannot proactively avoid errors.

Errors are more disruptive for single-threaded clients than for multi-threaded.
If one thread in a multi-threaded process encounters an error,
it warns the other threads not to use the disconnected server.
But single-threaded clients are deployed as many independent processes
per application server, and each process must throw an error
until all have discovered that a server is down.

The compromise specified here
balances the cost of frequent checks against the disruption of many errors.
The client preemptively checks individual sockets
that have not been used in the last `socketCheckIntervalMS`_,
which is more frequent by default than `heartbeatFrequencyMS`_.

No knobs
''''''''

This spec does not intend to introduce any new configuration options
unless absolutely necessary.

.. _monitors all three types of servers:

The client MUST monitor arbiters
''''''''''''''''''''''''''''''''

Mongos 2.6 does not monitor arbiters,
but it costs little to do so,
and in the rare case that
all data members are moved to new hosts in a short time,
an arbiter may be the client's last hope
to find the new replica set configuration.

Only support replica set members running MongoDB 1.6.2 or later
'''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''

Replica set members began reporting their setNames in that version.
Supporting earlier versions is impractical.

Drivers must not use the "ping" command
'''''''''''''''''''''''''''''''''''''''

Since discovery and monitoring require calling the "ismaster" command anyway,
drivers MUST standardize on the ismaster command instead of the "ping" command
to measure round-trip time to each server.

Additionally, the ismaster command is widely viewed as a special command used
when a client makes its initial connection to the server,
so it is less likely than "ping" to require authentication soon.

This spec does not mandate how round trip time is averaged
''''''''''''''''''''''''''''''''''''''''''''''''''''''''''

The Read Preferences spec requires drivers to calculate a round trip time
for each server to support the secondaryAcceptableLatencyMS option.
That spec calls this measurement the "ping time".
The measurement probably should be a moving average of some sort,
but it is not in the scope of this spec to mandate how drivers
should average the measurements.

TopologyType remains Unknown when an RSGhost is discovered
''''''''''''''''''''''''''''''''''''''''''''''''''''''''''

If the TopologyType is Unknown and the client receives an ismaster response
from an`RSGhost`_, the TopologyType could be set to ReplicaSetNoPrimary.
However, an RSGhost does not report its setName,
so the setName would still be unknown.
This adds an additional state to the existing list:
"TopologyType ReplicaSetNoPrimary **and** no setName."
The additional state adds substantial complexity
without any benefit, so this spec says clients MUST NOT change the TopologyType
when an RSGhost is discovered.

TopologyType remains Unknown when one of the seeds is a Standalone
''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''

If TopologyType is Unknown and there are multiple seeds,
and one of them is discovered to be a standalone,
it MUST be removed.
The TopologyType remains Unknown.

This rule supports the following common scenario:

#. Servers A and B are in a replica set.
#. A seed list with A and B is stored in a configuration file.
#. An administrator removes B from the set and brings it up as standalone
   for maintenance, without changing its port number.
#. The client is initialized with seeds A and B,
   TopologyType Unknown, and no setName.
#. The first ismaster response is from B, the standalone.

What if the client changed TopologyType to Single at this point?
It would be unable to use the replica set; it would have to remove A
from the TopologyDescription once A's ismaster response comes.

The user's intent in this case is clearly to use the replica set,
despite the outdated seed list. So this spec requires clients to remove B
from the TopologyDescription and keep the TopologyType as Unknown.
Then when A's respone arrives, the client can set its TopologyType
to ReplicaSet (with or without primary).

On the other hand,
if there is only one seed and the seed is discovered to be a Standalone,
the TopologyType MUST be set to Single.

See the "member brought up as standalone" test scenario.

Thread per server
'''''''''''''''''

Mongos uses a monitor thread per replica set, rather than a thread per server.
A thread per server is impractical if mongos is monitoring a large number of
replica sets.
But a driver only monitors one.

In mongos, threads trying to do reads and writes join the effort to scan
the replica set.
Such threads are more likely to be abundant in mongos than in drivers,
so mongos can rely on them to help with monitoring.

In short: mongos has different scaling concerns than
a multi-threaded or asynchronous driver,
so it allocates threads differently.

Socket timeout for monitoring is connectTimeoutMS
'''''''''''''''''''''''''''''''''''''''''''''''''

When a client waits for a server to respond to a connection,
the client does not know if the server will respond eventually or if it is down.
Users can help the client guess correctly
by supplying a reasonable connectTimeoutMS for their network:
on some networks a server is probably down if it hasn't responded in 10 ms,
on others a server might still be up even if it hasn't responded in 10 seconds.

The socketTimeoutMS, on the other hand, must account for both network latency
and the operation's duration on the server.
Applications should typically set a very long or infinite socketTimeoutMS
so they can wait for long-running MongoDB operations.

Multi-threaded clients use distinct sockets for monitoring and for application
operations.
A socket used for monitoring does two things: it connects and calls ismaster.
Both operations are fast on the server, so only network latency matters.
Thus both operations SHOULD use connectTimeoutMS, since that is the value
users supply to help the client guess if a server is down,
based on users' knowledge of expected latencies on their networks.

A monitor SHOULD NOT use the client's regular connection pool
'''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''

If a multi-threaded driver's connection pool enforces a maximum size
and monitors use sockets from the pool,
there are two bad options:
either monitors compete with the application for sockets,
or monitors have the exceptional ability
to create sockets even when the pool has reached its maximum size.
The former risks starving the monitor.
The latter is more complex than it is worth.
(A lesson learned from PyMongo 2.6's pool, which implemented this option.)

Since this rule is justified for drivers that enforce a maximum pool size,
this spec recommends that all drivers follow the same rule
for the sake of consistency.

Replica set monitoring with and without a primary
'''''''''''''''''''''''''''''''''''''''''''''''''

The client strives to fill the "servers" list
only with servers that the **primary**
said were members of the replica set,
when the client most recently contacted the primary.

The primary's view of the replica set is authoritative for two reasons:

1. The primary is never on the minority side of a network partition.
   During a partition it is the primary's list of
   servers the client should use.
2. Since reconfigs must be executed on the primary,
   the primary is the first to know of them.
   Reconfigs propagate to non-primaries eventually,
   but the client can receive ismaster responses from non-primaries
   that reflect any past state of the replica set.
   See the "Replica set discovery" test scenario.

If at any time the client believes there is no primary,
the TopologyDescription's type is set to ReplicaSetNoPrimary.
While there is no known primary,
the client MUST **add** servers from non-primaries' host lists,
but it MUST NOT remove servers from the TopologyDescription.

Eventually, when a primary is discovered, any hosts not in the primary's host
list are removed.

Ignore setVersion
'''''''''''''''''

It was thought that if all replica set members report a setVersion,
and a secondary's response has a higher setVersion than any seen,
that the secondary's host list could be considered as authoritative
as the primary's. (See previous section.)

This scenario illustrates the problem with setVersion:

* We have a replica set with servers A, B, and C.
* Server A is the primary, with setVersion 4.
* An administrator runs replSetReconfig on A,
  which increments its setVersion to 5.
* The client checks Server A and receives the new config.
* Server A crashes before any secondary receives the new config.
* Server B is elected primary. It has the old setVersion 4.
* The client ignores B's version of the config
  because its setVersion is not greater than 5.

The client may never correct its view of the topology.

Even worse:

* An administrator runs replSetReconfig
  on Server B, which increments its setVersion to 5.
* Server A restarts.
  This results in *two* versions of the config,
  both claiming to be version 5.

If the client trusted the setVersion in this scenario,
it would trust whichever config it received first.

mongos 2.6 ignores setVersion and only trusts the primary.
This spec requires all clients to ignore setVersion.

Retry ismaster calls once
'''''''''''''''''''''''''

A monitor's connection to a server is long-lived
and used only for ismaster calls.
So if a server has responded in the past,
a network error on the monitor's connection likely means there was
a network glitch or a server restart since the last check,
rather than that the server is down.
Marking the server Unknown in this case costs unnecessary effort.

However,
if the server still doesn't respond when the monitor attempts to reconnect,
then it is probably down.

Use error messages to detect "not master" and "node is recovering"
''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''

An alternative idea is to determine all relevant server error codes,
instead of searching for substrings in the error message.
But for "not master" and "node is recovering" errors,
driver authors have found the substrings to be **more** stable
than error codes.

The substring method has worked for drivers for years
so this spec does not propose a new method.

Backwards Compatibility
-----------------------

The Java driver 2.12.1 has a "heartbeatConnectRetryFrequency".
Since this spec recommends the option be named "minHeartbeatFrequencyMS",
the Java driver must deprecate its old option
and rename it minHeartbeatFrequency (for consistency with its other options
which also lack the "MS" suffix).

Reference Implementation
------------------------

* Java driver 3.x
* PyMongo 3.x
* Perl driver 1.0.0 (in progress)

Future Work
-----------

MongoDB is likely to add some of the following features,
which will require updates to this spec:

* Eventually consistent collections (SERVER-2956)
* Mongos discovery (SERVER-1834)
* Require auth for ismaster command (SERVER-12143)
* Put individual databases into maintenance mode,
  instead of the whole server (SERVER-7826)
* Put setVersion in write-command responses (SERVER-13909)

Questions and Answers
---------------------

When does a client see "not master" or "node is recovering"?
''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''

These errors indicate one of these:

* A write was attempted on an unwritable server
  (arbiter, secondary, slave, ghost, or recovering).
* A read was attempted on an unreadable server
  (arbiter, ghost, or recovering)
  or a read was attempted on a read-only server without the slaveOk bit set.

In any case the error is a symptom that
a ServerDescription's type no longer reflects reality.

A primary closes its connections when it steps down,
so in many cases the next operation causes a network error
rather than "not master".
The driver can see a "not master" error in the following scenario:

#. The client discovers the primary.
#. The primary steps down.
#. Before the client checks the server and discovers the stepdown,
   the application attempts an operation.
#. The client's connection pool is empty,
   either because it has
   never attempted an operation on this server,
   or because all connections are in use by other threads.
#. The client creates a connection to the old primary.
#. The client attempts to write, or to read without the slaveOk bit,
   and receives "not master".

See `"not master" and "node is recovering"`_,
and the test scenario called
"parsing 'not master' and 'node is recovering' errors".

What's the point of periodic monitoring?
''''''''''''''''''''''''''''''''''''''''

Why not just wait until a "not master" error or
"node is recovering" error informs the client that its
TopologyDescription is wrong? Or wait until `server selection`_
fails to find a suitable server, and only scan all servers then?

Periodic monitoring accomplishes three objectives:

* Update each server's type, tags, and `round trip time`_.
  Read preferences and the mongos selection algorithm
  require this information remains up to date.
* Discover new secondaries so that secondary reads are evenly spread.
* Detect incremental changes to the replica set configuration,
  so that the client remains connected to the set
  even while it is migrated to a completely new set of hosts.

If the application uses some servers very infrequently,
monitoring can also proactively detect state changes
(primary stepdown, server becoming unavailable)
that would otherwise cause future errors.

Acknowledgments
---------------

Jeff Yemin's code for the Java driver 2.12,
and his patient explanation thereof,
is the major inspiration for this spec.
Mathias Stearn's beautiful design for replica set monitoring in mongos 2.6
contributed as well.
Bernie Hackett gently oversaw the specification process.

.. _connection string: http://docs.mongodb.org/manual/reference/connection-string/
