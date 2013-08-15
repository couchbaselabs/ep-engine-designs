
##CRAM-MD5 Client

Couchbase Server 2.2 will add a new SASL authentication mechanism to memcached that client applications can use in order to encrpyt authentication credientials over the wire when connecting to memcached. This document intends to explain how clients will implement this new feature and what interfaces will be offered to client users in order to specify which authentication mechanism to use when connecting to memcached.

####Implementation Details

In order for users to chose an appropriate mechanism for authentication they will specify security policies which will be used to choose the best authentication mechanism. To figure this out an SDK will first send a list mechanisms command in order to figure out which mechanisms the server supports. When this command returns the client will then look at it's user specified policy and then chose a mechanism. If multiple mechanisms can be used then the most secure mechanism is chosen. If no mechanisms fit the policy then the connection will be aborted and an error will be reported to the user.

Below is a list of policies that should be defined.

1. NO_PLAINTEXT - Specifies that the selected SASL mechanism must not be susceptible to simple plain passive attacks.
2. NO_ANONYMOUS - Specifies that the selected SASL mechanism must not accept anonymous logins.
2. NO_DICTIONARY - Specifies that the selected SASL mechanism must not be susceptible to passive dictionary attacks.
3. NO_ACTIVE - Specifies that the selected SASL mechanism must not be susceptible to active (non-dictionary) attacks. The mechanism might require mutual authentication as a way to prevent active attacks.

For the initial implementation we only need to add the first policy since policies 2 and 3 would mean that neither PLAIN nor CRAM-MD5 could be used for authentication. Also note that policies can be combined with one another in order to provide stricter security policies.

The client will also be guaranteed to receive the mechanisms in order of most secure to least secure. Therefore once a client filters the list of the mechanisms it receives from the server the mechanism at the front of the list will be the most secure if multiple mechanisms exist in the list.

####Client Flow

1. The client sends a series of HTTP requests with HTTP basic auth headers to get the streaming URI
2. The client send a sasl_mechs command
3. The available mechanisms supported by the server are compared  to any application specified rules
4. The client will authenticate with any remaining mechanisms that are valid in the sasl_mechs ordered response

For on the wire packet formats please [see this spec](completed/md5-sasl.md#listing-mechanisms).

####Differences from the current implementation

The current implentation works by simply sending a PLAIN sasl auth message to memcached becuase PLAIN authentication is the only supported mechanism. Starting with this feature we will begin adding different kinds of authentication mechanisms to memcached and in the future we will likely support users to enable and disable authentication mechanisms on the server side. As a result clients will need to figure out which mechanisms are available and which mechanism to authenticate with iof multiple are available. As a result the clients will now need to do a list mechanisms command in order to figure out what is supported and will also need to provide an interface for defining security policies that can be used by the clients so that they can pick an appropriate authentication mechanism.

####What if List Mechanisms is Not Supported

This should not be an issue since list mechanisms should be supported in all versions of memcached, but if the list mechanisms command is not supported for some reason the client should provide try to authenticate with all of the supported client-side mechanisms starting with the most secure mechanism.

####Not Covered in the spec

* Server side details: [See this spec](completed/md5-sasl.md#listing-mechanisms)
* Moxi implmentation
* Documentation Needs
* Testing

####Issues Filed

* Client-Side: [JCBC-346](http://www.couchbase.com/issues/browse/JCBC-346)
* Server-Side: [MB-8260](http://www.couchbase.com/issues/browse/MB-8260)
* Documentation: [MB-8633](http://www.couchbase.com/issues/browse/MB-8633)