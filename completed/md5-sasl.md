
##CRAM-MD5 Support for SASL Auth

Currently in memcached we provide sasl authentication in order to allow multiple buckets to be run from the same process. Clients can decide which buckets they are connecting to by using PLAIN SASL authentication in order to specify the bucket name and bucket password. In all previous releases this mechanism has been used solely to specify the bucket to connect to and was not intended to provide encryption for the bucket names and passwords being sent over the wire. In order to extend this functionality to be more secure we can add CRAM-MD5 as an authentication mechanism in order to encrypt the username and password when sending it over the wire.

####Scope

The intent of this feature is solely to encrypt username and password information over the wire on memcached connections. Couchbase SDK's make other connections to Couchbase over HTTP for cluster configuration and queries and these connections will not be addressed in this specification.

####Backwards compatibility

This feature will not cause any backwards compatibility issues since it will be up to the client to decide which authentication mechanism to use. We will not remove any of the current mechanisms so the SDK's should not experience any issues connectint to an upgraded server. If an SDK tries to authenticate with an unsupported server mechanism then the client should close the connection and report that the client does not contain a upported authentication mechanism. Applicaiton traffic would not be allowed in this case.

####Forward compatibility

SDK's released with CRAM-MD5 as an authentication mechanism will need to continue to support PLAIN authentication in order to be able to connect to versions of Couchbase prior to version 2.2.

In order to upgrade a cluster users of Couchbase SDK's will need to make sure that their SDK supports at least one authentication mechanism of each version that will be running in the cluster. The cluster administrator should make sure the appropriat mechanisms are actually enabled on all SDKs.

####CRAM-MD5 Known Vulnerabilities

See section 5 of the IETF draft on CRAM-MD5 below.

http://tools.ietf.org/html/draft-ietf-sasl-crammd5-10

####Client Server Flow

#####Listing Mechanisms

The first step for authentication is for a client connecting to memcached to get a list of acceptable authentication mechanisms that can be used by the client. The request message below is used to ask the server for this list.

    SASL List Mechanisms Binary Request

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0|       80      |    B2 ('+')   |       00      |       05      |
      +---------------+---------------+---------------+---------------+
     4|       08      |       00      |       00      |       03      |
      +---------------+---------------+---------------+---------------+
     8|       00      |       00      |       00      |       14      |
      +---------------+---------------+---------------+---------------+
    12|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    16|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    20|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    24|       00      |       00      |       00      |       07      |
      +---------------+---------------+---------------+---------------+

    Header breakdown
    SASL List Mechanisms command
    Field        (offset) (value)
    Magic        (0)    : 0x80                (Request)
    Opcode       (1)    : 0x20                (SASL List Mechanisms)
    Key length   (2,3)  : 0x0000              (field not used)
    Extra length (4)    : 0x00                (field not used)
    Data type    (5)    : 0x00                (field not used)
    VBucket      (6,7)  : 0x0000              (field not used)
    Total body   (8-11) : 0x00000000          (field not used)
    Opaque       (12-15): 0x00000000
    CAS          (16-23): 0x0000000000000000  (field not used)

The server will respond with a list of mechanisms. As of Couchbase 2.1.0 memcached currently only supports PLAIN authentication, but this feature will add CRAM-MD5 authentication as well. Each mechanism will be separated by a single space (' ').

    SASL List Mechanisms Binary Response

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0|       81      |    20 (' ')   |       00      |       00      |
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
    24|       50      |       4C      |       41      |       49      |
      +---------------+---------------+---------------+---------------+
    28|       4E      |       20      |       43      |       52      |
      +---------------+---------------+---------------+---------------+
    32|       41      |       4D      |       2D      |       4D      |
      +---------------+---------------+---------------+---------------+
    36|       44      |       35      |
      +---------------+---------------+

    Header breakdown
    SASL List Mechanisms command
    Field        (offset) (value)
    Magic        (0)    : 0x81                (Response)
    Opcode       (1)    : 0x20                (SASL List Mechanisms)
    Key length   (2,3)  : 0x0000              (field not used)
    Extra length (4)    : 0x00                (field not used)
    Data type    (5)    : 0x00                (field not used)
    VBucket      (6,7)  : 0x0000              (field not used)
    Total body   (8-11) : 0x0000000D          (14)
    Opaque       (12-15): 0x00000000
    CAS          (16-23): 0x0000000000000000  (field not used)
	Mechanisms   (24-37): "PLAIN CRAM-MD5"

Once the client receives a list of supported server mechanisms the client will choose a mechanism for authentication. The message below shows what the client would send if they chose CRAM-MD5.

    SASL Client Authenticate Binary Request

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0|       81      |    21 (' ')   |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     4|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     8|       00      |       00      |       00      |       08      |
      +---------------+---------------+---------------+---------------+
    12|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    16|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    20|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    24|       43      |       52      |       41      |       4D      |
      +---------------+---------------+---------------+---------------+
    28|       2D      |       4D      |       44      |       35      |
      +---------------+---------------+---------------+---------------+

    Header breakdown
    SASL List Mechanisms command
    Field        (offset) (value)
    Magic        (0)    : 0x81                (Response)
    Opcode       (1)    : 0x21                (SASL Authenticate)
    Key length   (2,3)  : 0x0000              (field not used)
    Extra length (4)    : 0x00                (field not used)
    Data type    (5)    : 0x00                (field not used)
    VBucket      (6,7)  : 0x0000              (field not used)
    Total body   (8-11) : 0x00000008          (8)
    Opaque       (12-15): 0x00000000
    CAS          (16-23): 0x0000000000000000  (field not used)
	Mechanisms   (24-31): "CRAM-MD5"

The server will then respond to the client with a challenge. The challenge will be a random string sent by the server to the client that needs to be used when hashing the password with MD5. This random string is used to make sure that an attacker cannot gain access to Couchbase's memcached port by using a replay attack. This random string will be unique for each connection.

    SASL Server Challenge Binary Response

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0|       81      |    21 (' ')   |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     4|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     8|       00      |       00      |       00      |       12      |
      +---------------+---------------+---------------+---------------+
    12|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    16|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    20|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    24|       68      |       73      |       61      |       30      |
      +---------------+---------------+---------------+---------------+
    28|       62      |       66      |       32      |       38      |
      +---------------+---------------+---------------+---------------+
    32|       39      |       32      |       62      |       66      |
      +---------------+---------------+---------------+---------------+
    36|       77      |       66      |       6B      |       6B      |
      +---------------+---------------+---------------+---------------+

    Header breakdown
    SASL List Mechanisms command
    Field        (offset) (value)
    Magic        (0)    : 0x81                (Response)
    Opcode       (1)    : 0x21                (SASL Authenticate)
    Key length   (2,3)  : 0x0000              (field not used)
    Extra length (4)    : 0x00                (field not used)
    Data type    (5)    : 0x00                (field not used)
    VBucket      (6,7)  : 0x0000              (field not used)
    Total body   (8-11) : 0x00000012          (18)
    Opaque       (12-15): 0x00000000
    CAS          (16-23): 0x0000000000000000  (field not used)
	Mechanisms   (24-39): "hsa0bf2892bfwfkk"

The client will then provide an answer to the challenge. As noted above the challenge string should be used with the password when hashing it with MD5. The result of this hashing should be provided in the password field as shown below.

    SASL Client Challenge Binary Request

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0|       81      |    20 (' ')   |       00      |       08      |
      +---------------+---------------+---------------+---------------+
     4|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
     8|       00      |       00      |       00      |       24      |
      +---------------+---------------+---------------+---------------+
    12|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    16|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    20|       00      |       00      |       00      |       00      |
      +---------------+---------------+---------------+---------------+
    24|       75      |       73      |       65      |       72      |
      +---------------+---------------+---------------+---------------+
    28|       6E      |       61      |       6D      |       65      |
      +---------------+---------------+---------------+---------------+
    32|       66      |       33      |       32      |       37      |
      +---------------+---------------+---------------+---------------+
    36|       68      |       66      |       67      |       6A      |
      +---------------+---------------+---------------+---------------+
    40|       69      |       62      |       66      |       33      |
      +---------------+---------------+---------------+---------------+
    44|       39      |       34      |       38      |       66      |
      +---------------+---------------+---------------+---------------+

    Header breakdown
    SASL List Mechanisms command
    Field        (offset) (value)
    Magic        (0)    : 0x81                (Response)
    Opcode       (1)    : 0x22                (SASL Step)
    Key length   (2,3)  : 0x0008              (8)
    Extra length (4)    : 0x00                (field not used)
    Data type    (5)    : 0x00                (field not used)
    VBucket      (6,7)  : 0x0000              (field not used)
    Total body   (8-11) : 0x00000018          (24)
    Opaque       (12-15): 0x00000000
    CAS          (16-23): 0x0000000000000000  (field not used)
	Username     (24-31): "username"
    Password     (32-47): "f327hfqjibf3948f"

If the authentication is successful the client will receive a message from the server containing the memcached success error code.
