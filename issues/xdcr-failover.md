##XDCR-Failover Data Loss Issue

One of the main features of the Couchbase 2.x series is cross data center replication. This feature works by reading items off of disk on the active vbucket and sending them to a remote cluster through the use of front end memcached setMeta/delMeta operations. One ramification of the current approach to XDCR is that failovers can cause data to be overwritten when doing any replication that involves a circular dependency. Two cases are outlined below in which a failover occurs in a cluster that uses XDCR, resulting in data being overwritten.

First, consider the bi-directional XDCR use case shown below in Figure 1. Both the local and remote cluster each have two nodes and, to simplify this example, each cluster has 1 VBucket. This means that in both clusters Node A contains the active VBucket copy and Node B contains the replica VBucket copy. The application for this setup is connected only to the local cluster, so the remote cluster will only see incoming mutations through the XDCR stream.

![Security Issues Diagram](../images/xdcr_failover_figure_1.jpg)

As the application runs it will make updates on the local cluster. Figure 2 shows updates to a specific document, k1, over time. In this scenario, the application sends five updates to this document (signified in the diagram with kv pairs). These items are quickly persisted to disk and replicated to the remote datacenter through the XDCR stream. Node B on the local cluster, however, is overloaded, and as a result the intra-cluster stream used for replication between Node A and Node B becomes slow. This slowness results in only the second of five updates making it to the replica node in the local cluster, even though all of the mutations were streamed to the remote cluster through XDCR.

![Security Issues Diagram](../images/xdcr_failover_figure_2.jpg)

Shortly after the state shown in Figure 2 is achieved power is lost to Node A and the machine goes down. When power is restored to Node A, a complete disk failure occurs, and as a result the node is failed over. The application traffic resumes to Node B since it becomes the active server, and Node B receives two more updates to document k1. The state up to this point is shown in Figure 3 below.

![Security Issues Diagram](../images/xdcr_failover_figure_3.jpg)

The server state diagram above also includes the revision sequence number and cas value for each of the updates that take place to document k1. A clear problem arises when the server is in this state due to the way XDCR conflict resolution works. The current  conflict resolution mechanism first compares the revision sequence numbers. If one document has a high revision sequence number then that document wins. If the revision sequence numbers are the same then the cas value is compared to see which document has a higher cas number. If the cas numbers are the same then this process continues with checks to the expiration time and the flags values. If all of these values are equal then the documents are considered the same.

During the comparison of document k1 the remote cluster sees that the local cluster's k1 has a lower revision sequence number. Because of this, the remote cluster will overwrite the local cluster's document even though the document on the local cluster is actually newer. The two clusters then ends up in the state shown in Figure 4.

![Security Issues Diagram](../images/xdcr_failover_figure_4.jpg)

When failover occurs and there is bi-directional XDCR, this behavior leads to a situation where older data overwrites newer data. The amount of data that can be overwritten is dependent on how much data is lost due to failover of a node on the local cluster. This means that the loss generally depends on the health of the system at the time of the failover.

Similarly, it should also be noted that this situation can occur when two clusters switch from unidirectional XDCR to bidirectional XDCR. If there were failovers in the local cluster prior to the switch, then the behavior discussed in the example above may be seen since the remote cluster might have doucments that contain a higher sequence number than their counterpart documents on the local cluster.
