
#UPR Protocol

Introduction.

##Terminology

* **Consumer** - The endpoint in a connection that is responsible for requesting different kinds of data. The consumer is responsible for doing some sort of processing with the data sent from the consumer.
* **Producer** - The endpoint in a connection that is responsible for producing data for given requests.
* **Stream** - A stream is a series of messages sent over a given period of time that are generate from a particular request message.
* **Snapshot** - A unique sequence of keys that is sent by a stream.

##Open Bugs/Issues

* [MB-7558 Replicate lock time](http://www.couchbase.com/issues/browse/MB-7558)
* [MB-6077 Differentiate between deleted and expired items](http://www.couchbase.com/issues/browse/MB-6077)
* [MB-5147 Include VB Filter in consumer stats](http://www.couchbase.com/issues/browse/MB-5147)
* [MB-5146 Consumer shows useful name](http://www.couchbase.com/issues/browse/MB-5146)
* [MB-2721 Be able to specify filters on key and value](http://www.couchbase.com/issues/browse/MB-2721)
* [MB-8015 Avoid full rematerialization when cluster is restarted](http://www.couchbase.com/issues/browse/MB-8015)
* [MB-3493 bg_backlog_size keeps growing on an idle system](http://www.couchbase.com/issues/browse/MB-3493)
* [MB-3424 Allow tap to send keys only](http://www.couchbase.com/issues/browse/MB-3424)
* [CBD-510 Observe replication of a specific mutation](http://www.couchbase.com/issues/browse/CBD-510)
* We need to deal with dead connections in our specification
* How can we handle vbucket ownership transfers?
* We need to handle getting all high sequence numbers from a server
* We need a bucket flush packet
* How to do packet parsing? Current plan is protobufs, but why protobufs? Why not Thrift or Avro? What are pros/cons of doing parsing ourselves?
* Need to add protocol examples
* Discuss error opcodes
* Discuss Damien's rollback vs. Aarons (Wanted by Damien)
* Discuss whether request id is needed (Wanted by Damien)
* What happens if an item is replicated through XDCR to a remote cluster and the the local cluster crashes before the replicated item is persisted. This will result in inconsistency. Discuss this issue with Junyi.
* Address backwards compatability
* Better stats accounting when it comes to aggregrating tap stats. This is an issue in the current tap implementation too.
* UPR security issues
* We need to add a way to stop tap streams

##Messages

###UPR Header

	UPR Header

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0|       01      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     4|       00      |       00      |       00      |       0A      |
      +---------------+---------------+---------------+---------------+
     8|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+

    Header breakdown
    UPR Header
    Field          (offset)  (value)
    Opcode         (0)     : 0x01 (Request)
    Reserved       (1-3)   : 0x000000
	Request ID     (4-7)   : 0x0000000A
    Length         (8-11)  : 0x00000000

#####Opcode

The opcode field is used in order to specify the what type of message the client is receiving. Below are the currently available opcodes:

* **Request** (1) - Sent by the consumer side to the producer specifying that the consumer want some piece of data (Ex. XDCR).
* **Response** (2) - A producer message that is sent to the consumer for a certain requeste piece of data (Ex. XDCR).
* **Stream Begin** (3) - Means that you will recieve multiple responses for a request.
* **Stream Message** (4) - A single response that is contained in a stream
* **Stream Close** (5) - Says that the stream has ended.
* **Request Error Only** (6) - 
* **Response Decode Error** (7) - 

#####Request ID

The request ID is a 32-bit number that is used to identify the request that caused the producer to send a given piece of data.

#####Length

The length of the data that follows this header message.

###Request and Response Messages

UPR request messages are used by the consumer in order to specify a piece of data or a stream of data that the consumer wishes to obtain from the producer. Each request message will be answered with a response message. This message will contain a succes code or an error code indicating that the data requested by the consumer cannot be obtained at that time.

#####Request Types

* **Stream Request** (1) - Requests that data be streamed from a given VBucket.
* **Failover Log Request** (2) - Requests that failover log information be provided by the producer.

#####Response Types

* **OK** (1) - Specifies that the request was successful.
* **Not My VBucket** (2) - Specifies the the request was for a non-existent VBucket.
* **Rollback** (3) - Specifies that the consumer needs to rollback data before it can receive the data requested.

####Starting a Stream

In order to initial a stream from a vbucket the consumer must send the following command below. In order to initiate multiple stream the consumer needs to send multiple commands.

	UPR Stream Request Command

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0|       01      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     4|       04      |       00      |       00      |       2D      |
      +---------------+---------------+---------------+---------------+
     8|       00      |       00      |       00      |       18      |
      +---------------+---------------+---------------+---------------+
    12|       01      |       00      |       00      |       0C      |
      +---------------+---------------+---------------+---------------+
    16|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    20|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    24|       FF      |       FF      |       FF      |       FF      |
      +---------------+---------------+---------------+---------------+
    28|       FF      |       FF      |       FF      |       FF      |
      +---------------+---------------+---------------+---------------+
    32|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    36|       00      |       03      |       C5      |       8A      |
      +---------------+---------------+---------------+---------------+
    40|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    44|       00      |       00      |       0A      |       78      |
      +---------------+---------------+---------------+---------------+
    48|       72      |       65      |       70      |       6C      |
      +---------------+---------------+---------------+---------------+
    52|       69      |       63      |       61      |       74      |
      +---------------+---------------+---------------+---------------+
    56|       69      |       6F      |       6E      |       5F      |
      +---------------+---------------+---------------+---------------+
    60|       31      |       32      |
      +---------------+---------------+

    UPR Stream Request
    Field          (offset)  (value)
    Opcode         (0)     : 0x01                 (Request)
    Reserved       (1-3)   : 0x000000
	Request ID     (4-7)   : 0x0000002D           (46)
    Length         (8-11)  : 0x00000018           (24)
    Request Type   (12)    : 0x01                 (STREAM_REQUEST)    
    Reserved       (13)    : 0x00
	VBucket        (14-15) : 0x000C               (12)
	Start By Seqno (16-23) : 0x0000000000000000   (0)
    End By Seqno   (24-31) : 0xFFFFFFFFFFFFFFFF   (2^64)
    VBucket UUID   (32-39) : 0x000000000003C58A   (247178)
    High Seqno     (40-47) : 0x0000000000000A78   (2680)
    Group ID       (48-61) : "replication_12"

#####Fields

* **VBucket** - Specified the vbucket that data should be streamed from.
* **Start By Seqno** -  Specified the last by sequence number that has been seen by the consumer.
* **End By Seqno** - Specifies that the stream should be closed when the sequence number with this ID has been sent.
* **VBucket UUID** -
* **High Seqno** -
* **Group ID** -


After a consumer sends a request to the server for a stream the server can response in one of the following ways.

    UPR Stream OK Response Message

	Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0|       02      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     4|       00      |       00      |       00      |       2D      |
      +---------------+---------------+---------------+---------------+
     8|       00      |       00      |       00      |       01      |
      +---------------+---------------+---------------+---------------+
    12|       01      |
      +---------------+

    Header breakdown
    UPR Header command
    Field        (offset) (value)
    Opcode         (0)     : 0x02 (Response)
    Reserved       (1-3)   : 0x000000
	Request ID     (4-7)   : 0x0000002D           (46)
    Length         (8-11)  : 0x00000001           (1)
    Type           (12)    : 0x01                 (1)

    UPR Stream Not My VBucket Response Message

	Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0|       02      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     4|       00      |       00      |       00      |       2D      |
      +---------------+---------------+---------------+---------------+
     8|       00      |       00      |       00      |       01      |
      +---------------+---------------+---------------+---------------+
    12|       02      |
      +---------------+

    Header breakdown
    UPR Header command
    Field        (offset) (value)
    Opcode         (0)     : 0x02 (Response)
    Reserved       (1-3)   : 0x000000
	Request ID     (4-7)   : 0x0000002D           (46)
    Length         (8-11)  : 0x00000001           (1)
    Type           (12)    : 0x02                 (2)

    UPR Stream Rollback Response Message

	Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0|       02      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     4|       00      |       00      |       00      |       2D      |
      +---------------+---------------+---------------+---------------+
     8|       00      |       00      |       00      |       01      |
      +---------------+---------------+---------------+---------------+
    12|       03      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    16|       00      |       00      |       00      |       45      |
      +---------------+---------------+---------------+---------------+
    20|       A8      |
      +---------------+

    Header breakdown
    UPR Header command
    Field        (offset) (value)
    Opcode         (0)     : 0x02 (Response)
    Reserved       (1-3)   : 0x000000
	Request ID     (4-7)   : 0x0000002D           (46)
    Length         (8-11)  : 0x00000009           (9)
    Type           (12)    : 0x03                 (3)
	Rollback Seqno (13-20) : 0x00000000000045A8   (17832)

#####Fields

* **Rollback Seqno** - The rollback sequence number tells the consumer that it has newer mutations than the producer does. As a result the consumer needs to remove those new mutations. The sequence number can be used by the consumer to figure our which mutations to remove.

####Requesting A Failover Log

In order to request failover log information the following packet should be sent to the producer.

    UPR Failover Log Request

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0|       01      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     4|       04      |       00      |       00      |       2D      |
      +---------------+---------------+---------------+---------------+
     8|       00      |       00      |       00      |       01      |
      +---------------+---------------+---------------+---------------+
    12|       02      |
      +---------------+

    UPR Failover Log Request
    Field          (offset)  (value)
    Opcode         (0)     : 0x01                 (Request)
    Reserved       (1-3)   : 0x000000
	Request ID     (4-7)   : 0x0000002D           (46)
    Length         (8-11)  : 0x00000001           (1)
    Request Type   (12)    : 0x02                 (FAILOVER_LOG_REQUEST)    

The consumer should expect to receive a response from the producer that contains a list of zero or more Vbucket UUID and High Seqno pairs.

    UPR Stream Rollback Response Message

	Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0|       02      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     4|       00      |       00      |       00      |       2D      |
      +---------------+---------------+---------------+---------------+
     8|       00      |       00      |       00      |       21      |
      +---------------+---------------+---------------+---------------+
    12|       01      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    16|       00      |       00      |       00      |       0D      |
      +---------------+---------------+---------------+---------------+
    20|       F2      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    24|       00      |       00      |       00      |       F2      |
      +---------------+---------------+---------------+---------------+
    28|       94      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    32|       00      |       00      |       00      |       7B      |
      +---------------+---------------+---------------+---------------+
    36|       43      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    40|       00      |       00      |       00      |       93      |
      +---------------+---------------+---------------+---------------+
    44|       21      |
      +---------------+

    Header breakdown
    UPR Header command
    Field        (offset) (value)
    Opcode         (0)     : 0x02 (Response)
    Reserved       (1-3)   : 0x000000
	Request ID     (4-7)   : 0x0000002D           (46)
    Length         (8-11)  : 0x00000021           (33)
    Type           (12)    : 0x01                 (1)
    VBucket UUID   (13-20) : 0x0000000000000DF2   (3570)
    High Seqno     (21-28) : 0x000000000000F294   (62100)
    VBucket UUID   (29-36) : 0x0000000000007B43   (31555)
    High Seqno     (37-44) : 0x0000000000009321   (37665)


###Streams

Stream can be used in order to request a sequence of data from a given VBucket. A stream is created by sending the stream request message that has been defined above. Below we define the packets that can may be received by the consumer after a stream has been created successfully.

#####Stream Message Types

* Snapshot Start (1)
* Snapshot End (2)
* Mutation (3)
* Deletion (4)
* Expiration (5)
* Flush (6)

After a stream has been successfully created the first packet that is seen by the client will be a stream start message. This packet structure is defined below.

    UPR Stream Start Message

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0|       03      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     4|       00      |       00      |       00      |       2D      |
      +---------------+---------------+---------------+---------------+
     8|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+

    Header breakdown
    UPR Header command
    Field        (offset) (value)
    Opcode         (0)     : 0x03 (Stream Start)
    Reserved       (1-3)   : 0x000000
	Request ID     (4-7)   : 0x0000002D           (46)
    Length         (8-11)  : 0x00000000

An open stream will send packets in a series of snapshots. A snaphot is simply a series of packets that is guarenteed to contain a unique set of keys. Snapshots a signified by snapshot start and end messages. These messages are defined below.

    UPR Stream Snapshot Start Message

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0|       04      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     4|       00      |       00      |       00      |       2D      |
      +---------------+---------------+---------------+---------------+
     8|       00      |       00      |       00      |       01      |
      +---------------+---------------+---------------+---------------+
    12|       01      |
      +---------------+

    Header breakdown
    UPR Header command
    Field        (offset) (value)
    Opcode         (0)     : 0x04 (Stream Message)
    Reserved       (1-3)   : 0x000000
	Request ID     (4-7)   : 0x0000002D           (46)
    Length         (8-11)  : 0x00000001           (1)
    Type           (12)    : 0x01 (Snapshot Start)

    UPR Stream Snapshot End Message

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0|       04      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     4|       00      |       00      |       00      |       2D      |
      +---------------+---------------+---------------+---------------+
     8|       00      |       00      |       00      |       01      |
      +---------------+---------------+---------------+---------------+
    12|       02      |
      +---------------+

    Header breakdown
    UPR Header command
    Field        (offset) (value)
    Opcode         (0)     : 0x04 (Stream Message)
    Reserved       (1-3)   : 0x000000
	Request ID     (4-7)   : 0x0000002D           (46)
    Length         (8-11)  : 0x00000001           (1)
    Type           (12)    : 0x02 (Snapshot End)

After receiving an UPR stream start message the consumer will receive a series of UPR stream messsage that will specify mutations, deletes, and expirations.

    UPR Stream Message

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0|       04      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     4|       00      |       00      |       00      |       2D      |
      +---------------+---------------+---------------+---------------+
     8|       00      |       00      |       00      |       2D      |
      +---------------+---------------+---------------+---------------+
    12|       03      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    16|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    20|       A5      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    24|       00      |       00      |       00      |       08      |
      +---------------+---------------+---------------+---------------+
    28|       CB      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    32|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    36|       03      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    40|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    44|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    48|       00      |       6D      |       79      |       6B      |
      +---------------+---------------+---------------+---------------+
    52|       65      |       79      |       6D      |       79      |
      +---------------+---------------+---------------+---------------+
    56|       76      |       61      |       6C      |       75      |
      +---------------+---------------+---------------+---------------+
    58|       65      |
      +---------------+

    Header breakdown
    UPR Header command
    Field        (offset) (value)
    Opcode         (0)     : 0x04 (Stream Message)
    Reserved       (1-3)   : 0x000000
	Request ID     (4-7)   : 0x0000002D           (46)
    Length         (8-11)  : 0x0000002D           (46)
    Type           (12)    : 0x03                 (Mutation)
	Cas            (13-20) : 0x00000000000000A5   (165)
    By Seqno       (21-28) : 0x00000000000008CB   (2251)
    Rev Seqno      (29-36) : 0x0000000000000003   (3)
	Item Flags     (37-40) : 0x00000000           (0)
    Item Exp       (41-44) : 0x00000000           (0)
    Lock time      (45-48) : 0x00000000           (0)
    Key            (49-53) : "mykey"
	Value          (54-58) : "myvalue"


    UPR Stream Delete Message

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0|       04      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     4|       00      |       00      |       00      |       2D      |
      +---------------+---------------+---------------+---------------+
     8|       00      |       00      |       00      |       29      |
      +---------------+---------------+---------------+---------------+
    12|       04      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    16|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    20|       A5      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    24|       00      |       00      |       00      |       08      |
      +---------------+---------------+---------------+---------------+
    28|       CB      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    32|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    36|       03      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    40|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    44|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    48|       00      |       6D      |       79      |       6B      |
      +---------------+---------------+---------------+---------------+
    52|       65      |       79      |
      +---------------+---------------+


    Header breakdown
    UPR Header command
    Field        (offset) (value)
    Opcode         (0)     : 0x04 (Stream Message)
    Reserved       (1-3)   : 0x000000
	Request ID     (4-7)   : 0x0000002D           (46)
    Length         (8-11)  : 0x00000029           (41)
    Type           (12)    : 0x04                 (Deletion)
	Cas            (13-20) : 0x00000000000000A5   (165)
    By Seqno       (21-28) : 0x00000000000008CB   (2251)
    Rev Seqno      (29-36) : 0x0000000000000003   (3)
	Item Flags     (37-40) : 0x00000000           (0)
    Item Exp       (41-44) : 0x00000000           (0)
    Lock time      (45-48) : 0x00000000           (0)
    Key            (49-53) : "mykey"

    UPR Stream Expiration Message

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0|       04      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     4|       00      |       00      |       00      |       2D      |
      +---------------+---------------+---------------+---------------+
     8|       00      |       00      |       00      |       29      |
      +---------------+---------------+---------------+---------------+
    12|       05      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    16|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    20|       A5      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    24|       00      |       00      |       00      |       08      |
      +---------------+---------------+---------------+---------------+
    28|       CB      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    32|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    36|       03      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    40|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    44|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    48|       00      |       6D      |       79      |       6B      |
      +---------------+---------------+---------------+---------------+
    52|       65      |       79      |
      +---------------+---------------+


    Header breakdown
    UPR Header command
    Field        (offset) (value)
    Opcode         (0)     : 0x04 (Stream Message)
    Reserved       (1-3)   : 0x000000
	Request ID     (4-7)   : 0x0000002D           (46)
    Length         (8-11)  : 0x00000029           (41)
    Type           (12)    : 0x05                 (Expiration)
	Cas            (13-20) : 0x00000000000000A5   (165)
    By Seqno       (21-28) : 0x00000000000008CB   (2251)
    Rev Seqno      (29-36) : 0x0000000000000003   (3)
	Item Flags     (37-40) : 0x00000000           (0)
    Item Exp       (41-44) : 0x00000000           (0)
    Lock time      (45-48) : 0x00000000           (0)
    Key            (49-53) : "mykey"


##Use Cases

##FAQ
