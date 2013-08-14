
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

####On the wire packet details

[See this spec](completed/md5-sasl.md)