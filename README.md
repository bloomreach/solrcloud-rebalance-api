# SolrCloud Rebalance API 
The goal of the Rebalance API is to provide a zero downtime mechanism to perform data manipulation and efficient core allocation in solrcloud. This API was envisioned to be the base layer that enables Solrcloud to be an auto scaling platform. 
This document will focus purely on the API Specification. 

Internals of the Rebalance API:  http://engineering.bloomreach.com/solrcloud-rebalance-api/

# Authors  
 **Nitin Sharma** - nitin.sharma@bloomreach.com  
 **Suruchi Shah** - suruchi.shah@bloomreach.com


# API Specification
  Rebalance API allows two major constructs in its API 
  - **Scaling Strategy** : How to manipulate data in a cluster
    * **Re-distribute**  - Move around data in the cluster based on capacity/allocation
    * **Auto Shard**  - Dynamically shard a collection to any size
    * **Smart Merge - Distributed Mode** - Helps merging data from a larger shard setup into smaller one
    * **Scale up** -  Add replicas on the fly
    * **Scale Down** - Remove replicas on the fly 

  - **Allocation Strategy** : Where to place the new core
    * **Least Used** - Use the host with least number of cores
    *  **Unused** - Use the host that does not have any core for this collection
 
The api has been built with hooks so that you can write your own Scaling and Allocation Strategies and it will
seamlessly fit inside the Rebalance paradigm.


| API                                                                 | Action                                       |
|---------------------------------------------------------------------|----------------------------------------------|
| /admin/collections?**action**=REBALANCE&**scaling_strategy**=AUTO_SHARD   | Dynamically shards a collection              |
| /admin/collections?**action**=REBALANCE&**scaling_strategy**=REDISTRIBUTE | Redistributes cores across live no           |
| /admin/collections?**action**=REBALANCE&**scaling_strategy**=REPLACE      | Replaces all cores on a node to another node |
| /admin/collections?**action**=REBALANCE&**scaling_strategy**=SCALE_UP     | Increases the number of replicas on shards   |
| /admin/collections?**action**=REBALANCE&**scaling_strategy**=SCALE_DOWN   | Decreases the number of replicas on shards   |
| /admin/collections?**action**=REBALANCE&**scaling_strategy**=SMART_MERGE_DISTRIBUTED |  Distributed Merge of solr cores  |
| /admin/collections?**action**=REDISTRIBUTEALL   | Redistributes all collections           |


# Scaling Strategies

### AUTO SHARD 

Automatically re-shards a given collection in the desired number of shards based on the user defined allocation strategy:

/admin/collections?**action**=REBALANCE  
&**scaling_strategy**=AUTO_SHARD  
&**collection**=collection_name    
&**num_shards**=number of shards    
&**split_only**=true/false    
&**size_cap**=1G    
&**flip_alias**=true    
&**allocation_strategy**=least_used    
&**dest_collection**=new_collection  

| Key              | Type    | Required  | Description                                                                                                                                                                                                       |
|------------------|---------|-----------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| scaling_strategy | string  | Yes       | Exact value to set: AUTO_SHARD                                                                                                                                                                                    |
| collection       | string  | Yes       | The name of the collection to be re-sharded                                                                                                                                                                       |
| num_shards       | integer | Yes       | The number of shards to have in the destination collection                                                                                                                                                        |
| split_only       | boolean | No        | <true/false> value which defines whether the logic should avoid merge step thus optimizing the process. Can be set to true,if the destination number of shards is greater than AND divisible by the original number of shards. |
| size_cap         | integer | No        | This argument is for Size-based sharding. The value of this argument defines the max size of index data to be stored in a given shard. Consequently, the number of shards are decided based on the size cap       |
| flip_alias       | boolean | No        | <true/false> value which defines whether the alias of the original collection be flipped to the new one                                                                                                                        |
| dest_collection  | string  | No        | The name of the destination collection with the new sharding setup. The default name is originalCollectionName_rebalanced                                                                                         |
| allocation_strategy | string | No      | <unused/least_used> values. The allocation strategy defines WHERE the new nodes reside. If least_used strategy is selected, amongst all nodes, the node with least number of index data segments is selected. If unused strategy is selected, amongst all nodes,the node which has NOT been used for a given collection is selected. The default allocation strategy is least_used|

 **Example 1**:  [With default allocation strategy]  

  **Before**:  
   ![Alt text](/1.png?raw=true "AutoShard Example Default")

  **Command**:  
  ```
  /solr/admin/collections?action=REBALANCE&scaling_strategy=AUTO_SHARD&collection=suruchi_test_collection&num_shards=4
  ```

  **After**:  
    ![Alt text](/2.png?raw=true "AutoShard Example After")

 **Example 2**:  [With a specified allocation strategy]
**Before**:  
   ![Alt text](/4.png?raw=true "AutoShard Example With Allocation")

  **Command**:  
   ```
  /solr/admin/collections?action=REBALANCE&scaling_strategy=AUTO_SHARD&collection=nitin_test_collection&num_shards=4&allocation_strategy=least_used
   ```

  **After**:  
    ![Alt text](/3.png?raw=true "AutoShard Example After Allocation")


### REDISTRIBUTE  

 Automatically redistribute the collection across available nodes based on user-defined strategy  
 
 /admin/collections?action=REBALANCE  
 &**scaling_strategy**=REDISTRIBUTE  
 &**collection**=collectionName  

| Key              | Type   | Required  | Description                                 |
|------------------|--------|-----------|---------------------------------------------|
| scaling_strategy | string | Yes       | Exact value to set: REDISTRIBUTE            |
| collection       | string | Yes       | The name of the collection to be re-sharded |

**Example 1**:

  **Before**:  
   ![Alt text](/5.png?raw=true "Re-distribute Example Default")

  **Command**:  
    ```
    /solr/admin/collections?action=REBALANCE&scaling_strategy=REDISTRIBUTE&collection=nitin_test_collection_rebalanced
    ```

  **After**:  
    ![Alt text](/6.png?raw=true "Redistribute Example After")

### REPLACE   
 	
  REPLACE option will automatically migrate all the cores (shards and replicas)  for that collection from one node to another node  
  		
  /admin/collections?action=REBALANCE  		
  &**scaling_strategy**=REDISTRIBUTE  		
  &**collection**=collectionName  		
 		
| Key              | Type   | Required  | Description                                 |
|------------------|--------|-----------|---------------------------------------------|
| scaling_strategy | string | Yes       | Exact value to set: REPLACE                 |
| collection       | string | Yes       | The name of the collection to be re-sharded |
| source_host      | string | Yes       | The name of the source host to be replaced |
| destination_host | string | Yes       | The name of the destination host to place new cores|


### Smart Merge Distributed  

Dynamically merge cores from a given collection x to a new collection y by distributing the workload across machines in the cluster.
There are cases where we might want to have a different setup for indexing vs serving. Indexing can have 40 shards (for throughput) and serving can just have 5 (for latency reasons). This api will come in handy to merge data from an indexing setup to a serving setup. (larger to smaller).  

/admin/collections?**action**=REBALANCE    
&**scaling_strategy**=SMART_MERGE_DISTRIBUTED  
&**collection**=collectionName    
&**num_shards**=numRequiredShards  
&**num_replicas**=number_of_replicas    
&**bucketize**=true    



| Key              | Type    | Required  | Description                                    |
|------------------|---------|-----------|------------------------------------------------|
| scaling_strategy | string  | Yes       | Exact value to set: SMART_MERGE_DISTRIBUTED    |
| collection       | string  | Yes       | Name of the collection                         |
| num_shards       | int     | Yes       | Number of shards of the destination collection |
| num_replicas     | int     | No        | Number of replicas per shard                   |
| bucketize        | boolean | Yes       | true                                           |

**Example 1**:

  **Before**:  
   ![Alt text](/12.png?raw=true "Merge Example Default")

  **Command**:  

  ```
   /solr/admin/collections?action=REBALANCE&scaling_strategy=SMART_MERGE_DISTRIBUTED&collection=test_collection&num_shards=4&bucketize=true
  ```

  **After**:  
    ![Alt text](/14.png?raw=true "Merge Example After")


### Scale Up

Dynamically increases the number of replicas for a given collection on the nodes defined by the user-selection for allocation strategy  

/admin/collections?**action**=REBALANCE  
&**scaling_strategy**=SCALE_UP    
&**collection**=collection_name    
&**num_replicas**=numReplicas    
&**allocation_strategy**=unused    

| Key                 | Type   | Required  | Description                                                  |
|---------------------|--------|-----------|--------------------------------------------------------------|
| scaling_strategy    | string | Yes       | Exact value to set: SCALE_UP                                 |
| collection          | string | Yes       | The name of the collection to be scaled up                   |
| num_replicas        | int    | Yes       | Number of replicas by which each shard be scaled up          |
| allocation_strategy | string | No        | The allocation strategy defines where the new nodes reside.  |

**Example 1**:  

  **Before**:
   ![Alt text](/7.png?raw=true "Scale Up Example Default")

  **Command**:  

    ```
     /solr/admin/collections?action=REBALANCE&scaling_strategy=SCALE_UP&collection=suruchi_test_collection&num_replicas=1
    ```

  **After**:  
    ![Alt text](/8.png?raw=true "Scale Up Example After")


### Scale Down

Decreases the number of replicas for a given collection. If it is the only replica available, then it will not proceed with delete.

/admin/collections?**action**=REBALANCE  
&**scaling_strategy**=SCALE_DOWN  
&**collection**=collection_name    
&**num_replicas**=numReplicas    
&**allocation_strategy**=unused    

| Key                 | Type   | Required  | Description                                                  |
|---------------------|--------|-----------|--------------------------------------------------------------|
| scaling_strategy    | string | Yes       | Exact value to set: SCALE_UP                                 |
| collection          | string | Yes       | The name of the collection to be scaled up                   |
| num_replicas        | int    | Yes       | Number of replicas by which each shard be scaled down        |
| allocation_strategy | string | No        | The allocation strategy defines where the new nodes reside.  |

**Example 1**:    

  **Before**:  
   ![Alt text](/9.png?raw=true "Scale Down Example Default")

  **Command**:    

   ```
   /solr/admin/collections?action=REBALANCE&scaling_strategy=SCALE_DOWN&collection=suruchi_test_collection&num_replicas=1
   ```
 
  **After**:    
    ![Alt text](/10.png?raw=true "Scale Down Example After")

# Building your own Allocation Strategy  
   
   You can build your own allocation strategy by extending the Allocation Strategy class and implementing the getNode() method. The node that is returned by the method will contain the placement for the new core (replica, shard) for that scaling strategy. An example is shown below  

   ![Alt text](/11.png?raw=true "Writing your own Allocation Strategy")
