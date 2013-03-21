
##VBucket Delete

The VBucket delete command allows a client to delete one or more VBuckets.

####Binary Implementation

There are two seperate ways of specifying a VBucket delete command depending on whether or not the client wants to delete one or multiple VBuckets. The server will interpret the packet format based on whether or not the packet contains a body section.

The first way of specifying a VBucket delete can only be used to delete a single VBucket. The VBucket to be deleted is specified in the vbucket field and contains no body section. Please note that this format is deprecated and future versions of Couchbase will interpret this format as an invalid packet.


    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0|       80      |    3F ('@')   |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     4|       04      |       00      |       00      |       0A      |
      +---------------+---------------+---------------+---------------+
     8|       00      |       00      |       00      |       04      |
      +---------------+---------------+---------------+---------------+
    12|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    16|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    20|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    24|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+

    Header breakdown
    VBucket delete command
    Field        (offset) (value)
    Magic        (0)    : 0x80 (Request)
    Opcode       (1)    : 0x3F
    Key length   (2,3)  : 0x0000 (0)
    Extra length (4)    : 0x04
    Data type    (5)    : 0x00                (field not used)
    vbucket      (6,7)  : 0x000A (10)
    Total body   (8-11) : 0x00000004 (4)
    Opaque       (12-15): 0x00000000
    CAS          (16-23): 0x0000000000000000  (field not used)
    Flags        (24-27): 0x00000000

The second way of specifying a VBucket delete will work for deleting one or more VBuckets. This packet does ignores the vbucket field, but does contain a body section. Below is an example of a VBucket command that delete vbuckets 0, 1, and 2.

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0|       80      |    3F ('@')   |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     4|       04      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     8|       00      |       00      |       00      |       0A      |
      +---------------+---------------+---------------+---------------+
    12|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    16|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    20|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    24|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    28|       00      |       00      |       00      |       01      |
      +---------------+---------------+---------------+---------------+
    32|       00      |       02      |

    Header breakdown
    VBucket delete command
    Field        (offset) (value)
    Magic        (0)    : 0x80 (Request)
    Opcode       (1)    : 0x3F
    Key length   (2,3)  : 0x0000 (0)
    Extra length (4)    : 0x04
    Data type    (5)    : 0x00                (field not used)
    vbucket      (6,7)  : 0x0000 (0)          (field not used)
    Total body   (8-11) : 0x0000000A (10)
    Opaque       (12-15): 0x00000000
    CAS          (16-23): 0x0000000000000000  (field not used)
    Flags        (24-27): 0x00000000
    vbucket      (28-29): 0
    vbucket      (30-31): 1
    vbucket      (32-33): 2

####Flags

**VBUCKET_DELETE_ASYNC 0x01**

Setting this flag will cause the client to return immediately after the server processes the request. The server will only schedule a background job to delete the VBucket specified in the request. It is up to the user to later check to see if the VBucket deletion was successful.

**VBUCKET_DELETE_FORCE 0x02**

By defualt a VBucket must be in "dead" state in order for the server to delete it. Setting this flag will tell the server to delete the VBucket no matter what state the VBucket is in.

