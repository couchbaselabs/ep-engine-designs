#Indexing (3.x)

In Couchbase 3.x we will introduce a new replication protocol that can be used to stream mutations directly into the indexer. This means that the indexer will no longer be required to read mutations directly from disk and also means that there is no longer a need for the Couch-Notifier in ep-engine. As a result the indexer will no longer be able to 

To better understand how the indexes are updated in Couchbase 3.x let's assume that we have a single server with one bucket and that bucket has a design document with one view. The bucket initially contains no items and as a result the indexer is up to date with the data in the key-value store.