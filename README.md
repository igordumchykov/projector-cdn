# projector-cdn
Creates DNS server, 2 load balancers, 4 nginx nodes that serve and cache static content

# Run
1. To enable DNS resolution on local machine, go to MAC OS `System Settings` > `Network` > choose your network > `Details` > `DNS` > add entry `127.0.0.1`
2. Run the script: 
```shell
docker-compose up -d
```

# Project structure
1. [cache](cache) - cache for nginx node-1, node-2, node-3, node-4 static content
2. [config](config) - configuration files for containers
3. [data](data) - static content served by nginx nodes
4. [docker-compose](docker-compose.yml) - docker compose file with all containers

# Containers
1. bind-dns - resolves `http://website.home:81` and `http://website.home:82` to `localhost:81`, `localhost:82`
2. load-balancer-1 - balances traffic from localhost:81 to node-1, node-2, node-3, node-4
3. load-balancer-2 - balances traffic from localhost:82 to node-1, node-2, node-3, node-4
4. node-1 - serves static content
5. node-2 - serves static content
6. node-3 - serves static content
7. node-4 - serves static content

# Test

```shell
curl http://website.home:81/image/cat.jpeg
```

```shell
curl http://website.home:82/image/cat.jpeg
```

or

use browser to get image

# Load Balancing Strategies

## Round Robin

This is default load balancing strategy.

Requests are distributed sequentially in the list of available servers.

### Pros:

- Simple and Predictable: The default method, it evenly distributes traffic across servers, making its behavior easy to anticipate.
- Fair Distribution: If all servers have similar performance characteristics, this ensures equal distribution of requests.

### Cons:

- No Server Health Checks: In the basic configuration, Nginx won't know if a backend server is down. (Though health checks can be added.)
- No Consideration for Server Load: Does not consider if one server is more loaded or has more active connections than another.

## Least Connections

- Directs traffic to the server with the least number of active connections.
- This is particularly useful when there are servers of varied capacities.

Set `least_conn` under `config/load-balancer-1/nginx.conf`,`config/load-balancer-2/nginx.conf` in section `http > upstream backend_servers`

### Pros:

- Optimal for Varied Server Capacities: Ensures that less powerful servers aren’t overwhelmed with requests.
- Dynamically Balanced: Responds to real-time server loads.

### Cons:

- Potential Latency: Might have a slight delay in decision-making as it checks for the number of connections on each server.

## IP Hash

- A hash-function is used to determine which server should be selected for the next request based on the client's IP address.
- Useful for session persistence in applications where the user should connect to the same backend server for the duration of their session.

Set `ip_hash` under `config/load-balancer-1/nginx.conf`,`config/load-balancer-2/nginx.conf` in section `http > upstream backend_servers`

### Pros:

- Session Persistence: Ensures that a client is consistently connected to the same backend server, beneficial for applications that store session state locally on the server.
- Predictable: Given a client IP, it will always hash to the same server (as long as the server list remains unchanged).

### Cons:

- Potential Imbalance: Might not distribute traffic evenly, especially if a subset of IP addresses generate a majority of the traffic.
- Server Addition/Removal: If servers are added or removed, the hash distribution changes, which can disrupt sessions.

## Generic Hash

- Allows load balancing based on a combination of values like client IP, request URI, etc.
- Good for more granular control.

Set `hash <number of attributes, e.g. $remote_addr$request_uri consistent>` under `config/load-balancer-1/nginx.conf`,`config/load-balancer-2/nginx.conf` in section `http > upstream backend_servers`

### Pros:

- Granular Control: Can balance based on multiple variables, not just IP.
- Flexible: Can adapt to various applications and their specific needs.

### Cons:

- Complexity: Requires more thoughtful setup and testing to ensure the desired distribution of traffic.

## Least Time

Sends requests to the server that has the lowest average latency and least number of active connections.

Set `least_time header` under `config/load-balancer-1/nginx.conf`,`config/load-balancer-2/nginx.conf` in section `http > upstream backend_servers`

### Pros:

- Performance-centric: Prioritizes servers based on response times.
- Combines Two Metrics: Considers both connection and response times.

### Cons:

- Module Requirement: Requires a specific module which is not included in the default Nginx build.
- Overhead: Introduces a slight overhead due to time tracking.

## Weighted

- Used in conjunction with Round Robin or IP Hash.
- Assigns a weight to each server to influence its proportion of requests. A server with weight 2 will get double the requests as a server with weight 1.

Example:
```nginx configuration
http {
    upstream backend {
        server backend1.example.com weight=3;
        server backend2.example.com weight=2;
    }
}
```

### Pros:

- Control Over Distribution: Provides manual control over traffic distribution based on server weights.
- Tailored to Environment: Useful when you know the exact capabilities of your servers.

### Cons:

- Manual Configuration: Requires manual adjustments if server capacities change.
- No Dynamic Adjustments: Weights are static and do not adapt to real-time server performance.

## Zoned

- Allows shared memory zones to store group states. It can be used to store the state and manage consistent hashing.
- Useful for more advanced configurations and when adding/removing servers without affecting the existing cache.

Example:
```nginx configuration
http {
    upstream backend {
        zone backend 64k;
        server backend1.example.com;
        server backend2.example.com;
    }
}
```

### Pros:

- Consistent Hashing: Useful when adding or removing servers, ensuring minimal disruption to the current cache.
- Shared State: Allows state sharing across worker processes, ensuring more accurate load balancing decisions.

### Cons:

- Memory Usage: Requires allocation of shared memory.
- Configuration Complexity: Adds an extra layer of complexity to the configuration.