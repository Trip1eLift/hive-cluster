# hive-cluster
PoC of cluster with redis to handle cluster functionality

## Count engaged containers

Having an accurate number of containers that's currently taking network traffic can enable many functionality such as cluster rate limiting.

### Queen architecture

Work flow of taking network traffic

1. Whenever a container in the cluster is hit by a traffic except `/health`, register itself to redis with TTL (time to live).

| Redis index     | value                |
| --------------- | -------------        |
| container-id    | engaged-until        |
| container-(uid) | (current-time) + 30s |

2. Check if there is a queen.

- If there is a queen, do nothing and skip until step 5.
- If there is not a queen or the queen is dead, continue to step 3.
- To check if the queen is dead, pull `queen-container-id` then check if queen is still enaged.

3. All container contest to be a queen

- Need to somehow handle race condition to make sure there is only one queen.
- To be a queen, set `queen-container-id`

| Redis index        | value           |
| ------------------ | --------------- |
| queen-container-id | container-(uid) |

4. Queen container will peridocally (5s) determine the number of engaged container.

- Pull all redis index with prefix `container-`, registered in step 1, and check if each container is alive.
- If container is dead, remove the redis index and value.
- If container is alive, count it.
- Write the engaged container count to redis.

| Redis index       | value        |
| ----------------- | ------------ |
| engaged-container | (an-integer) |

5. For every 30s, each container will pull from engaged-container, if 0 container is engaged, treat it as 1.