Owner: Jin Lim

Created On: 03/14/2013

Last Modified: 03/15/2013

Time to Complete:  

Scheduled Version: 2.1

###Delete vBucket (aka FLUSH) Optimization

Shufflemaster, one of Couchbase customers, has brount up an issue where deleting 1024 vBuckets (flushing database) takes more than a few minutes to complete. Please see MB-6232 for more details. This document briefly summarizes a few enhancment approaches that reduces the prolonged process of sequentially deleting individual vBuckets.

####Affected Components

* NS Server
* EP Engine
* CouchDB

####Short-Term Solution - Disable Write Barrier
This approach does not requires any changes in Couchbase components or current protocol for deleting vBuckets. Write Barrier, a OS filesystem mechanism, basically ensure that data being written via fsync() is persistent to storage devices via disk write through method. For this data access that creates and deletes many small files, vBucket files for Couchbase, incur perfroamnce slowness. By disabling wrtie barrie (barrier=0) the engineering team measures that both deleting and creating vBuckets get significantly faster. 

* For a Linux system, one can disable the barrier at mount time using "-o nobarrier" option for "mount" command. Please note that some devices may not support write barriers. Check for an error message in /var/log/messages. 

Read more information about the storage write through method at http://hub.internal.couchbase.com/confluence/display/~jin/Random+Thoughts+on+Disk+Caching.

#### Skip snapshotVBuckets for dead vBuckets

######Affected Modules (EP Engine)
* ep.cc
* couch-kvstore.cc

EP Engine snapshotVBucket() incurs fsync() to underlying storage devices (vBucket files) in order to persist the vBucket state. The state is written inside a special document called  "local_doc" and every vBucket file has one "local_doc". 

Under the current design, EP Engine does snapshotVBucket() twice for each vBucket deletion. It schedules snapshotVBucket() for each deleting vBucket when it first moves the vBucket's state to "dead". Then it schedules snapshotVBucket() again after completion of each vBucket deleteion to restore the "dead" state. However, if all the vBucket deletions are part of flushing database, EP Engine doesn't need to persiste the "dead" state twice hence can skip all the snapshotVBucket() calls. By eliminating these unnecessary snapshotVBucket() the process of flushing database then will get much faster because there is no fsync in the way.

Proposed new steps of EP Engine for flushing database: 
* NS Server send CMD_DISABLE_TRAFFIC in order to hint EP Engine its intention of flushing database
* EP Engine starts return TEMP_FAIL to incoming client requests
* NS Server starts sending DEL_VBUCKET for each vBucket
* EP Engine cleans up the vBucket (outgoing queue, etc)
* EP Engine sends DEL_VBUCKET to MCCOUCH
* EP Engine responses completion of DEL_VBUCKET to NS Server   

#### Remove MCCOUCH DEL_VBUCKET
In addtion to the proposed change above, EP Engine and NS Server can further optimize the process of flushing database by removing MCCOUCHS DEL_VBUCKET. MCCOUCH is a part of NS Server's erlang process and NS Server orchestrates the entire process of flusing database. Therefore, it should be NS Server not EP Engine that invokes MCCOUCH's DEL_VBUCKET for each deleting vBucket. And, if it is found to be efficient, the protocole of MCCOUCH  DEL_VBUCKET should also change to allow a batch of vBuckets instead of a single vBucket for its delete method.
   
### Performance Impacts
Deleting vBuckets as part of flushing database should get faster.

### Schedule
* Skip snapshotVBucket() For Dead VBucket - 2.0.1
* Remove MCCOUCH DEL_VBUCKET - 2.1
