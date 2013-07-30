
##XDCR (Version 2.x)

When cross data center replication is set up between two nodes the user has the option of setting up either unidirectional or bidirectional cross data center replication. A bidirectional replication just contains two unidirectional streams that are independent on eachother so in the example below we will only examine the unidirectional case.

TODO: Section on setup

Once a unidirectional stream is set up the XDCR replicators will read from


1. Read items from a vbucket on disk
2. Send a batch of 500 keys and their meta data from the local cluster to the remote cluster
3. For each item the remote cluster does a getMeta
4. The data returned from the getMeta goes through a conflict resolution mechanism. If the item is considered newer then we add a "send list". Otherwise we just forget about the key.
5. Everything in the "send list" is sent from the remote cluster back to the local cluster. The local cluster then gets the values for each of these items and sends the keys/values/meta data back to the remote cluster
6. The remote cluster does another getMeta for each item it received.
7. The response from the getMeta is used to perform conflict resolution
8. If the item from the local cluster wins then we do a setMeta on that item and we include the cas value of the remote item that we used to perform conflict resolution wins. If the cas value has changed then we go back to step 6.
9. Last we go back to step one and repeat this entire process.