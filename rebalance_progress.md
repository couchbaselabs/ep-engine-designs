Owner: Mike Wiederhold  
Created On: 3/12/13  
Last Modified: 3/12/13  
Time to Complete: 5 Days  
Scheduled Version: 2.0.2

##Estimate Warmup/Rebalance Progress

A common issue for developers and customers is trying to understand how long it will take for long running server operations to complete. In Couchbase 2.0 the two longest running operations are warmup and reblance. This feature aims to provide statistics that will allow users to gain more insight into what is happen during these operations and how long they will take.

###Affected Components

* Ep-Engine
* NS_Server

###Affected Modules

* Tap

### Protocol Changes
(None)

### Implementation Details

Implementation of this feature requires Ep-Engine providing stats that can be used by NS_Server in order to provide information about the progress of those operations so it can make predictions about their completion times.

#####Rebalance

The rebalance task requires that the Couchbase Admin Console displays the following information:

1. Estimated Number of keys to be transferred
2. Total Number of keys transferred
3. Active VBuckets currently being transferred
4. Replica VBuckets currently being transferred

**Estimated Number of keys to be transferred**

For each VBucket that needs to be moved Ep-Engine will need to provide an estimate of how many keys will need to be sent through a tap stream in order for a VBucket move to be completed. In order to do this we will add a new command called ESTIMATE_VB_MOVE which will take a tap name as an argument. This command will examine where the tap cursor for a particular tap connection is and whether backfill is needed and provide an estimate of how many keys need to be transmitted through the stream to get a destination VBucket up to date. This stat will take into account the following:

* Is backfill required?
* If we do backfill do we schedule full backfill or do disk fetches?
* How many items will be sent through memory backfill?
* How many items are in front of the tap cursor in the checkpoint queues?
* If the tap cursor doesn't exist then we need to handle this appropriately

Also note that NS_Server should update this estimate from time to time. When a VBucket has been moved the estimated keys to move for a particuar VBucket should be replaced by the total number of keys transferred stat which is discussed in the next section.

**Total Number of keys transferred**

For each tap connection EP-Engine will provide a new stat that contains the number of items transmitted by the tap stream. This stat will be available on the producer tap stream and will be incremented when the consumer has ACK'ed transmissions from the producer.

 **Active VBuckets currently being transferred**

NS_Server is aware of which VBuckets is is currently moving between nodes and will display this stat without the help of EP_Engine.

 **Replica VBuckets currently being transferred**

NS_Server is aware of which VBuckets is is currently moving between nodes and will display this stat without the help of EP_Engine.

#####Warmup

The warmup task requires that the Couchbase Admin Console displays the following information:

1. Total number of keys to warm-up
2. Number of keys already warmed-up
3. Total number of values to warm-up
4. Number of values already warmed-up
5. Approximate time to completion
6. State of warmup thread

In order to get information on warm-up Ep-Engine already has an API that has been available since 2.0 (stats warmup). NS_Server should use this API to get all warm-up related stats.

**Total number of keys to warm-up**

Use the 'ep_warmup_estimated_key_count' stats from the 'stats warmup' API.

**Number of keys already warmed-up**

Use the 'ep_warmup_key_count' stats from the 'stats warmup' API.

**Total number of values to warm-up**

Use the 'ep_warmup_estimated_value_count' stats from the 'stats warmup' API.

**Number of values already warmed-up**

Use the 'ep_warmup_value_count' stats from the 'stats warmup' API.

**Approximate time to completion**

This stats will need to be estimated by NS_Server and cannot be done accurately because it is not possible to know the sizes of values that are being lodaed from disk which means that different sequences of values loaded would yeild different amount ofkeys resident. NS_Server can use the estimated keys/value to warmup and key/value count provided in the warmup stats. Based on the rate of keys/values warmed-up per second we can generate an estimated warm-up time.

**State of warmup thread**

Use the 'ep_warmup_thread' stats from the 'stats warmup' API.


### Performance Impacts

Stats calls should not cause any noticable performance impact on any server operations. NS_Server should however make an effort to minimize the number of stats calls needed.

