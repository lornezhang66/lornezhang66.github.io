###  kong关于pgsql的几个参数







```shell
# This section allows for configuring the behavior of Kong regarding the
# caching of such configuration entities.

#db_update_frequency = 5         # Frequency (in seconds) at which to check for
                                 # updated entities with the datastore.
                                 # When a node creates, updates, or deletes an
                                 # entity via the Admin API, other nodes need
                                 # to wait for the next poll (configured by
                                 # this value) to eventually purge the old
                                 # cached entity and start using the new one.

#db_update_propagation = 0       # Time (in seconds) taken for an entity in the
                                 # datastore to be propagated to replica nodes
                                 # of another datacenter.
                                 # When in a distributed environment such as
                                 # a multi-datacenter Cassandra cluster, this
                                 # value should be the maximum number of
                                 # seconds taken by Cassandra to propagate a
                                 # row to other datacenters.
                                 # When set, this property will increase the
                                 # a multi-datacenter Cassandra cluster, this
                                 # value should be the maximum number of
                                 # seconds taken by Cassandra to propagate a
                                 # row to other datacenters.
                                 # When set, this property will increase the
                                 # time taken by Kong to propagate the change
                                 # of an entity.
                                 # Single-datacenter setups or PostgreSQL
                                 # servers should suffer no such delays, and
                                 # this value can be safely set to 0.

#db_cache_ttl = 3600             # Time-to-live (in seconds) of an entity from
                                 # the datastore when cached by this node.
                                 # Database misses (no entity) are also cached
                                 # according to this setting.
                                 # If set to 0, such cached entities/misses
                                 # never expire.	 
```

