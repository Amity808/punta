# 11 — Load Balancing in Modern System Design & Punta Architecture

Load balancing is a core pillar of distributed systems, enabling horizontal scalability, fault tolerance, and high availability. This article explores the fundamentals of load balancing, core routing algorithms, the differences between Layer 4 and Layer 7 load balancers, and how load balancing is implemented in the context of the **Punta** multi-tenant platform.

---

## 1. What is a Load Balancer?

A load balancer acts as a "traffic cop" sitting in front of your servers and routing client requests across all servers capable of fulfilling those requests. The goal is to maximize throughput, minimize latency, reduce resource utilization, and ensure that no single server becomes a bottleneck or single point of failure (SPOF).

```
                             ┌─────────────────┐
                             │     Clients     │
                             └────────┬────────┘
                                      │ (Traffic)
                                      ▼
                             ┌─────────────────┐
                             │  Load Balancer  │
                             └────────┬────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              │ (Route Request)       │ (Route Request)       │ (Route Request)
              ▼                       ▼                       ▼
    ┌───────────────────┐   ┌───────────────────┐   ┌───────────────────┐
    │    App Server 1   │   │    App Server 2   │   │    App Server 3   │
    │  (Healthy/Active) │   │  (Healthy/Active) │   │  (Healthy/Active) │
    └───────────────────┘   └───────────────────┘   └───────────────────┘
```

### Key Objectives
* **Scalability:** Allows adding or removing backend servers dynamically as traffic volume fluctuates.
* **High Availability (HA):** Automatically redirects traffic away from unhealthy instances to healthy ones.
* **Efficiency:** Prevents resource saturation on individual servers.
* **Security:** Offloads SSL/TLS decryption, hides backend infrastructure IP addresses, and cushions against DDoS attacks.

---

## 2. Where Do Load Balancers Sit?

Load balancers are not restricted to the edge of your infrastructure. In a tier-structured cloud application, they sit at multiple layers:

1. **User to Web Server (Edge):** Directs incoming user traffic to web servers, content delivery networks (CDNs), or API gateways.
2. **Web Server to Application Server (Internal):** Routes traffic from web/frontend servers (e.g., Next.js rendering instances) to internal application services (e.g., Rust Axum API instances).
3. **Application Server to Database/Cache:** Distributes queries or database connections across read replicas, caching nodes, or microservices.

---

## 3. Layer 4 vs. Layer 7 Load Balancing

Load balancers operate at different layers of the Open Systems Interconnection (OSI) model. The two most common types are **Layer 4 (Transport Layer)** and **Layer 7 (Application Layer)**.

| Feature | Layer 4 (L4) | Layer 7 (L7) |
| :--- | :--- | :--- |
| **OSI Layer** | Transport Layer (TCP, UDP) | Application Layer (HTTP, HTTPS, gRPC, WebSocket) |
| **Routing Decision basis** | IP addresses, TCP/UDP ports, protocol details. | HTTP headers, cookies, URL paths, query parameters, payloads. |
| **Awareness of Data** | None. It routes raw bytes without inspecting the application data payload. | High. It parses headers, SSL handshakes, cookies, and JSON/XML bodies. |
| **Performance** | Extremely fast and CPU efficient. Requires low memory since it doesn't parse payloads. | Slightly slower and more CPU/Memory intensive due to deep packet inspection. |
| **SSL Termination** | Cannot terminate SSL directly (must pass-through raw bytes, or terminate at L4 TCP level). | Can terminate SSL, inspect decrypted traffic, and re-encrypt or pass in plain HTTP. |
| **Features** | Simple routing, basic TCP health checks. | Smart routing, sticky sessions, rate limiting, header injection, path-based routing. |
| **Example Software** | HAProxy, Nginx, AWS NLB (Network Load Balancer), IPVS. | HAProxy, Nginx, Envoy, Traefik, AWS ALB (Application Load Balancer), Cloudflare. |

### L4 Routing Flow (Packet Level)
At L4, the load balancer receives a packet, changes the destination IP to one of the healthy backend servers (using NAT - Network Address Translation), and forwards it. It maintains a state table of connections based on the 4-tuple: `(Source IP, Source Port, Destination IP, Destination Port)`.

### L7 Routing Flow (Content-Aware)
At L7, the load balancer terminates the client's TCP connection, performs the SSL handshake, decrypts and parses the HTTP request, makes a routing decision (e.g., routing `/api/v1/orders` to the Order Service and `/static/*` to object storage), establishes a *new* TCP connection to the chosen backend server, and forwards the request.

---

## 4. Load Balancing Algorithms

The logic used to distribute requests determines the load balancer's efficiency. The choice of algorithm depends heavily on the workload and state persistence requirements.

### A. Static Algorithms
* **Round Robin:** Requests are distributed sequentially across servers (1 → 2 → 3 → 1). Assumes all servers have equal capacity.
* **Weighted Round Robin:** Servers are assigned weights based on capacity (e.g., Server A has weight 3, Server B has weight 1; Server A receives 3 requests for every 1 Server B receives).
* **Random:** Distributes requests randomly. Simple and performs well when backend nodes are homogeneous.

### B. Dynamic Algorithms
* **Least Connections:** Routes traffic to the server with the fewest active TCP connections. Ideal for long-lived connections (e.g., WebSockets, SQL transactions).
* **Weighted Least Connections:** Combines the least connections metric with server weights.
* **Least Response Time (Latency-Based):** Directs requests to the server with the lowest combination of active connections and response time.
* **IP Hash:** Computes a hash of the client's IP address and maps it to a specific server. Ensures that a client is consistently routed to the same backend node (useful for stateful sessions).

### C. Consistent Hashing
Essential for distributed caches (like Memcached or Redis) and distributed databases. Standard hashing (`hash(Key) % N`) causes massive cache misses if the number of servers $N$ changes. 

Consistent hashing maps both servers and keys to a virtual ring ($0$ to $2^{32}-1$). A key is assigned to the first server it encounters moving clockwise. When a server is added or removed, only a small fraction of keys ($1/N$) need to be remapped.

```
                  Server 1 (Hash: 90)
                     \       /
                      \     /
     Key A ──▶         \   /          ◀── Key B
   (Hash: 40)           \ /             (Hash: 110)
    [Routed to           X             [Routed to
     Server 1]          / \             Server 2]
                       /   \
                      /     \
                Server 2 (Hash: 210)
```

---

## 5. High Availability of the Load Balancer

If the load balancer goes down, your entire system becomes unavailable. To prevent the load balancer from becoming a Single Point of Failure (SPOF), you must implement High Availability (HA):

### Active-Passive Configuration (Failover)
* **Setup:** Two load balancer nodes are configured. One is Active (handling all traffic), and the other is Passive (idle/monitoring).
* **Mechanism:** A protocol like **VRRP** (Virtual Router Redundancy Protocol) is used. The load balancers share a single Virtual IP (VIP).
* **Action:** The Passive node polls the Active node via keepalive heartbeats. If the Active node fails, the Passive node takes over the VIP and begins processing traffic immediately.

### Active-Active Configuration (DNS / Anycast)
* **DNS Round-Robin:** The DNS registrar returns multiple IP addresses corresponding to different load balancers. The client's browser picks one. If one goes down, the client experiences a delay until their browser retries or DNS caches expire.
* **BGP Anycast:** Multiple load balancers announce the exact same IP address using the Border Gateway Protocol (BGP). Routers direct traffic to the closest load balancer geographically. If one fails, routers route traffic to the next closest one dynamically.

---

## 6. Implementation Reference (Nginx Configuration)

Nginx is commonly used as a reverse proxy and load balancer. Below is a configuration example representing both Layer 4 and Layer 7 capabilities.

### Layer 7 HTTP Load Balancer with Nginx
```nginx
http {
    # Define backend pool (Upstream)
    upstream rust_api_cluster {
        # Least connections algorithm
        least_conn; 

        server 10.0.0.10:8080 weight=3; # Powerful node
        server 10.0.0.11:8080;          # Standard node
        server 10.0.0.12:8080 backup;   # Only receives traffic when others fail
        
        keepalive 32;                   # Maintain open connections to backends
    }

    server {
        listen 80;
        listen [::]:80;
        server_name api.punta.shop;

        # Redirect plain HTTP to HTTPS
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name api.punta.shop;

        # SSL Configuration
        ssl_certificate /etc/letsencrypt/live/punta.shop/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/punta.shop/privkey.pem;

        # Path-based routing (L7 feature)
        location /api/v1/ {
            proxy_pass http://rust_api_cluster;
            proxy_http_version 1.1;
            
            # Connection settings
            proxy_set_header Connection "";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Timeouts
            proxy_connect_timeout 5s;
            proxy_read_timeout 60s;
        }

        # Static assets served directly (L7 feature)
        location /static/ {
            root /var/www/punta/static;
            expires 30d;
            add_header Cache-Control "public, no-transform";
        }
    }
}
```

### Layer 4 TCP Load Balancer with Nginx
For routing raw TCP connections (such as database pools or raw socket connections):
```nginx
stream {
    upstream postgres_cluster {
        hash $remote_addr consistent; # Consistent hashing by client IP
        server db-primary.internal:5432;
        server db-replica.internal:5432;
    }

    server {
        listen 5432;
        proxy_pass postgres_cluster;
        proxy_timeout 10m;
        proxy_connect_timeout 2s;
    }
}
```

---

## 7. Load Balancing in the Punta Architecture

In **Punta's** cloud infrastructure, load balancing is implemented at multiple levels to handle multi-tenancy, custom domains, and edge delivery:

```
                  ┌───────────────────────────────┐
                  │            Client             │
                  └───────────────┬───────────────┘
                                  │ (Custom or *.punta.shop domain)
                                  ▼
                  ┌───────────────────────────────┐
                  │        Cloudflare Edge        │  <-- Global L7 Load Balancing, SSL/TLS, CDN
                  └───────────────┬───────────────┘
                                  │ (Anycast Routing)
                                  ▼
                  ┌───────────────────────────────┐
                  │       Fly.io Proxy (L4)       │  <-- Anycast IP, routes to nearest active region
                  └───────────────┬───────────────┘
                                  │ (Internal Routing)
                                  ▼
                  ┌───────────────────────────────┐
                  │    Rust API Node (Axum)       │  <-- Multi-Tenant Context Resolver Middleware
                  └───────────────┬───────────────┘
                                  │
         ┌────────────────────────┼────────────────────────┐
         ▼                        ▼                        ▼
 ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
 │ Neon Postgres│         │ Redis Cache  │         │ Cloudflare R2│  <-- Databases & Storage Layer
 └──────────────┘         └──────────────┘         └──────────────┘
```

### 1. The Global Edge Layer (Cloudflare)
* **Anycast Network:** Cloudflare maps `*.punta.shop` and merchant custom domains (e.g., `mybrand.com.ng`) to its global Anycast IPs. Client DNS requests are automatically routed to the closest Cloudflare data center (edge node).
* **Layer 7 Actions:** Cloudflare inspects headers, terminates SSL at the edge, serves cached Next.js storefront pages, filters malicious traffic (DDoS mitigation), and forwards requests to the origin.

### 2. Compute Load Balancing (Fly.io / ECS)
* **Fly Proxy:** Fly.io uses a custom L4/L7 proxy architecture. A single Anycast IP points to the Fly.io network. The proxy automatically forwards incoming TCP/HTTP traffic to the instance of the Rust API container running closest to the client.
* **Auto-Scaling:** If the load on the Rust API instances exceeds threshold levels (e.g., CPU > 70% or concurrent requests > 100 per container), Fly.io spins up additional containers. The proxy instantly recognizes the new instances via internal health checks and begins load balancing traffic to them.
* **Health Checks:** Every container exposes a health check endpoint (e.g., `/health`). The load balancer polls this endpoint. If a container panics or goes offline, it is instantly removed from the routing pool.

### 3. Tenant Resolution & Routing (Application Layer)
Standard load balancers do not understand multi-tenant application boundaries (e.g., isolating a tenant's database connection or routing queries based on subscription limits).
* In Punta, this is solved by routing all tenant traffic to the same general backend pool of Rust/Next.js containers.
* The **Rust API (Axum) Middleware** resolves the tenant dynamically:
  1. It reads the incoming `Host` header (e.g., `acme.punta.shop` or `mybrand.com.ng`).
  2. It resolves this host against a cached database registry in Redis.
  3. It sets the session variable `app.current_tenant` in PostgreSQL to enforce Row-Level Security (RLS).
* Therefore, logical multi-tenant load distribution happens inside the application code, while physical request load distribution is handled by the network load balancers.

---

## 8. Summary Checklist for System Design Interviews
If designing a system with load balancers in an interview, keep this progression in mind:
1. **Identify the scaling point:** Introduce a load balancer as soon as you transition from 1 server to 2+ servers.
2. **Choose L4 vs. L7:** Use L4 for high performance, simple routing (e.g., databases, UDP streaming). Use L7 for HTTP features, path-based microservices, and SSL termination.
3. **Decide on routing algorithm:** Round Robin for identical servers, Least Connections for long sessions, IP Hash/Consistent Hashing for caching and data routing.
4. **Make it highly available:** Avoid SPOF by setting up active-passive pairs (VRRP/keepalived) or active-active global setups (DNS, Anycast).
5. **Add health checks:** Never assume backends are always healthy. Configure passive/active health monitoring.
