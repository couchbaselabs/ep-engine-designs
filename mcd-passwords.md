
##Adding a SASLPW Command

  In order to use sasl for authentication in memcached the user must supply a list of valid username/password combinations which are stored in an on-disk file. This on-disk file is then read by memcached on start-up and sasl authentication requests use the passwords in the sasl password file in order to provide security. Couchbase utilizes this feature in order to both specify which bucket a user is connected to and for secuirty reasons, but since buckets can be created at any point during the lifetime of a running server passwords must be able to be added, removed, and changed at runtime.

  In order to accomplish this the current approach is for ns_server to have access to the sasl password file and update it as buckets are created, deleted, or modified. As a result of these modifications memcached is forced to read the file each time a connection that requires authentication is made. The downsides of this scheme are that it requires more disk accesses then should be neccessary and that it requires multiple components to be aware of the on disk format of the sasl password file.

  A better scheme would be to add a new memcached command that would allow the cluster orchestrator to create, modify or delete username/password information from an in-memory structure. This would remove the need for disk reads and would allow the format of the stored usernames/passwords to be controled by a single component. Note that entries in memcached are not durably stored and must be added by the cluster orchestrator when the server is started.

####In Memory Authentication Table

  The in-memory authentication table will map usernames to passwords. Usernames are required to be unique in the this table, but password may be duplicated. Usernames and passwords will also be restricted in size. It is considered an error to specify a username or password which is longer than 256 bytes.

####SASLPW Command

  The SASLPW command is used to insert and remove entries from the username/password table that is stored in memory by memcached.

  In the binary requests below the data type field is user to specify whether or not the item should be added to or removed from the username/password table in memcached. The client should specify 0x00 for insert and 0x01 for remove.

    SASLPW Binary Request (insert)

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0|       80      |    23 ('#')   |       00      |       05      |
      +---------------+---------------+---------------+---------------+
     4|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     8|       00      |       00      |       00      |       0A      |
      +---------------+---------------+---------------+---------------+
    12|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    16|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    20|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    24|       6D      |       79      |       75      |       73      |
      +---------------+---------------+---------------+---------------+
    28|       65      |       72      |       6D      |       77      |
      +---------------+---------------+---------------+---------------+
    32|       70      |       61      |       73      |       73      |
      +---------------+---------------+---------------+---------------+


    Header breakdown
    SASLPW command
    Field        (offset) (value)
    Magic        (0)    : 0x80 (Request)
    Opcode       (1)    : 0x23 (SASLPW)
    Key length   (2,3)  : 0x0005 (5)
    Extra length (4)    : 0x00                (field not used)
    Data type    (5)    : 0x00 (0)
    VBucket      (6,7)  : 0x0000              (field not used)
    Total body   (8-11) : 0x0000000A (10)
    Opaque       (12-15): 0x00000000
    CAS          (16-23): 0x0000000000000000  (field not used)
    Username     (24-29): myuser
	Password     (30-35): mypass


    SASLPW Binary Request (remove)

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0|       80      |    23 ('#')   |       00      |       05      |
      +---------------+---------------+---------------+---------------+
     4|       00      |       01      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     8|       00      |       00      |       00      |       0A      |
      +---------------+---------------+---------------+---------------+
    12|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    16|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    20|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    24|       6D      |       79      |       75      |       73      |
      +---------------+---------------+---------------+---------------+
    28|       65      |       72      |
      +---------------+---------------+


    Header breakdown
    SASLPW command
    Field        (offset) (value)
    Magic        (0)    : 0x80 (Request)
    Opcode       (1)    : 0x23 (SASLPW)
    Key length   (2,3)  : 0x0005 (5)
    Extra length (4)    : 0x00                (field not used)
    Data type    (5)    : 0x00 (1)
    VBucket      (6,7)  : 0x0000              (field not used)
    Total body   (8-11) : 0x0000000A (10)
    Opaque       (12-15): 0x00000000
    CAS          (16-23): 0x0000000000000000  (field not used)
    Username     (24-29): myuser


    SASLPW Response

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0|       81      |    23 ('#')   |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     4|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     8|       00      |       00      |       00      |       0D      |
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
    SASL List Mechanisms command
    Field        (offset) (value)
    Magic        (0)    : 0x81 (Response)
    Opcode       (1)    : 0x23 (setrm)
    Key length   (2,3)  : 0x0000              (field not used)
    Extra length (4)    : 0x00                (field not used)
    Data type    (5)    : 0x00                (field not used)
    VBucket      (6,7)  : 0x0000              (field not used)
    Total body   (8-11) : 0x00000000          (field not used)
    Opaque       (12-15): 0x00000000
    CAS          (16-23): 0x0000000000000000  (field not used)

#####Returns

  The saslpw command returns success if the username/password entry was properly stored or removed for memcached otherwise and error is returned. Consult the errors section for a list of errors that can be returned and the reasons for the errors.

#####Errors

**PROTOCOL_BINARY_RESPONSE_KEY_EEXISTS (0x02)**

  This error is returned is if the saslpw command specifies a create on a username that already exists in the username/password table.

**PROTOCOL_BINARY_RESPONSE_E2BIG (0x03)**

  This error is returned if the username or password is longer than 256 bytes.

**PROTOCOL_BINARY_RESPONSE_EINVAL (0x04)**

  If data in this packet is malformed or incomplete then this error is returned and means that their is likely a bug in the client that sent the request.

**PROTOCOL_BINARY_RESPONSE_ENOMEM (0x82)**

  This error is returned if the request cannot be completed because the server is permanently out of memory.

###Affected External Components

  Upon completion of this task the cluster orchestrator will need to support the new command and remove any code that uses the current process of modifying the on-disk sasl password file. This work should be minor compared to the tasks in memcached. Other than the cluster orchestrator this feature should not have any effect on other parts of the system.

###Future Work

  A better implementation would be to wrap the functionality in the command proposed above into the current create/delete bucket commands. This will allow for a simpler interface while still achieving the same results. Doing this however will break backwards compatibility in the create/delete bucket commands and is also not a simple coding task given the current layout of the memcached/bucket-engine code. If memcached is re-written in the future this should be taken into account in the design.
