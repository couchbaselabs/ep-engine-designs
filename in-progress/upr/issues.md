
#UPR Issue Tracking

The issue tracking section contains a list of todo items as well as links to filed Jira bugs. The purpose of this section is to ensure that nothing gets lost.

##Open Bugs/Issues

* [MB-5147 Include VB Filter in consumer stats](http://www.couchbase.com/issues/browse/MB-5147)
* [MB-5146 Consumer shows useful name](http://www.couchbase.com/issues/browse/MB-5146)
* [MB-2721 Be able to specify filters on key and value](http://www.couchbase.com/issues/browse/MB-2721)
* [MB-3493 bg_backlog_size keeps growing on an idle system](http://www.couchbase.com/issues/browse/MB-3493)
* [MB-3424 Allow tap to send keys only](http://www.couchbase.com/issues/browse/MB-3424)
* [CBD-510 Observe replication of a specific mutation](http://www.couchbase.com/issues/browse/CBD-510)
* How can we handle vbucket ownership transfers?
* Need to add protocol examples
* Discuss error opcodes
* Better stats accounting when it comes to aggregrating tap stats. This is an issue in the current tap implementation too.
* Add a RESPONSE_ERROR_DECODE error opcode
* Specify which vbucket we are willing to accept

##Resolved Issues

This section is intended to provide a list of open issues that are addressed by the UPR Specification. Each issue listed will provide a short summary of how the issue will be addressed and why it is being addressed.

* [MB-7558 Replicate lock time](http://www.couchbase.com/issues/browse/MB-7558)<br>
Couchbase currently provides a "get and lock" command that can be used in order to make sure that only one client has access to a piece of data associated with a given key. By default the maximum lock time is 30 seconds, but this setting is user configurable and some users have increased the maximum lock time in their applications. This can create issues during failover since the current tap implementation does not replicate the lock time. The UPR specification will resolve this issue by including a lock time field in all UPR Mutation Messages.

* [MB-6077 Differentiate between deleted and expired items](http://www.couchbase.com/issues/browse/MB-6077)<br>
The current tap implementation only provides a deleted message and this means that if an item expires on the active node then we will register it as a delete in the stats of the replica node. This behavior makes the stats between active and replica VBuckets in consistent. This issue is easily resolved by adding a seperate UPR Expiration Message to the current specification.

* [MB-8015 Avoid full rematerialization when cluster is restarted](http://www.couchbase.com/issues/browse/MB-8015)<br>
If a replication stream dies in the middle of a backfill then the current architecture requires that we restart the entire stream from the beginning. The UPR specification resolves this issue by utilizing the by "sequence number" field and enables a failed connection to resume data streaming from where ever it left off.
