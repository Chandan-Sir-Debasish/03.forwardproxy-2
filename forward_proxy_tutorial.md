# Educational Tutorial: Nginx Forward Proxy and Content Filtering

Welcome to the Forward Proxy Lab. This tutorial is structured as an academic resource to teach students the core principles of proxy server architecture, transport protocols, IP-based access control, and content-filtering regex rules.

---

## Lesson 1: Understanding Forward Proxies

A Forward Proxy is an intermediary server that acts on behalf of a group of clients (usually located within a private internal network) to manage and regulate their outbound requests to the external internet.

When an internal client requests a resource hosted on the public internet:
1. Instead of establishing a direct connection to the target server, the client sends the outbound traffic to the Forward Proxy.
2. The proxy intercepts the request and evaluates it against set administrative policies (such as domain blocklists, IP whitelists, or content filters).
3. If the request is compliant, the proxy resolves the target server's IP address and forwards the request.
4. The destination server receives the request originating from the proxy server's IP address, masking the internal client's identity.

---

## Lesson 2: Forward Proxy vs. Reverse Proxy

Understanding the distinction between forward and reverse proxies depends on identifying which side of the network connection (client or server) the proxy represents.

### 1. Forward Proxy (Client-Side Proxy)
A Forward Proxy represents the client. It intercepts outbound traffic from a private network to the public internet. The destination server remains unaware of the specific client's IP address.

```mermaid
graph LR
    subgraph "Internal Client Network"
        ClientA["Client A"]
        ClientB["Client B"]
    end
    
    Proxy["Forward Proxy (Gatekeeper)"]
    Internet["Public Internet"]
    
    subgraph "External Servers"
        DestA["example.com"]
        DestB["google.com"]
    end

    ClientA -->|Outbound request| Proxy
    ClientB -->|Outbound request| Proxy
    Proxy -->|Forwards verified traffic| Internet
    Internet --> DestA
    Internet --> DestB
```

### 2. Reverse Proxy (Server-Side Proxy)
A Reverse Proxy represents the server farm. It intercepts inbound traffic from the public internet to internal servers. External clients believe they are communicating directly with the primary server, but the Reverse Proxy intercepts requests to handle load balancing, SSL/TLS termination, or caching.

```mermaid
graph LR
    subgraph "External Clients"
        Client1["User 1"]
        Client2["User 2"]
    end
    
    RevProxy["Reverse Proxy (Load Balancer)"]
    
    subgraph "Internal Server Farm"
        Srv1["App Server 1"]
        Srv2["App Server 2"]
        DB[("Database")]
    end

    Client1 -->|Inbound request| RevProxy
    Client2 -->|Inbound request| RevProxy
    RevProxy -->|Balances load| Srv1
    RevProxy -->|Balances load| Srv2
    Srv1 --> DB
    Srv2 --> DB
```

### Key Differences Comparison

| Architectural Dimension | Forward Proxy | Reverse Proxy |
| :--- | :--- | :--- |
| Primary Beneficiary | Clients (protects/controls internal network users) | Servers (protects/accelerates back-end infrastructure) |
| Configuration | Explicitly configured on client browser or system | Transparent to client (client accesses public DNS) |
| Visibility | Hides client IP from external servers | Hides server IPs from external clients |
| Common Uses | Access control, content filtering, employee audit logs | Load balancing, SSL termination, DDoS protection, Web Application Firewalls (WAF) |

---

## Lesson 3: Deep Dive into Forward Proxy Architecture

To understand forward proxying in depth, we must examine the network topology and the step-by-step request-response flow for both HTTP and HTTPS traffic.

### Network Topology Diagram

The following diagram illustrates the network isolation in this project. The client containers (`client1` and `client2`) are placed inside a private subnet and have no default route directly to the public internet. All outbound internet traffic must route through the `nginx-proxy` container, which acts as the gateway.

```mermaid
graph TD
    subgraph "Host Machine"
        HostInterface["Host Net Interface"]
    end

    subgraph "Docker Bridge Network (proxy-network: 172.20.0.0/16)"
        Gateway["Gateway (172.20.0.1)"]
        DNS["Docker DNS (127.0.0.11)"]
        
        subgraph "Allowed Clients Subnet"
            C1["Client 1 (172.20.0.10)"]
            C2["Client 2 (172.20.0.11)"]
        end

        subgraph "Nginx Proxy Service"
            Proxy["nginx-proxy (172.20.0.2)"]
        end
    end

    subgraph "External Internet"
        Target["httpbin.org"]
    end

    C1 -->|Forward request| Proxy
    C2 -->|Forward request| Proxy
    Proxy -->|Local DNS Query| DNS
    DNS -->|Upstream DNS Resolution| Gateway
    Proxy -->|HTTP Forwarded Session| Gateway
    Gateway -->|NAT routing| HostInterface
    HostInterface -->|Public Outbound| Target
```

---

## Lesson 4: Transport Protocol Mechanics (GET vs. CONNECT)

Forward proxies handle standard HTTP (unencrypted) and HTTPS (encrypted) traffic differently.

### 1. HTTP Forwarding (GET/POST Proxying)
For unencrypted HTTP requests, the proxy can inspect and modify request headers and payloads.
- The client establishes a TCP connection to the proxy port (e.g., 8888).
- The client formats the HTTP request using the absolute URL of the target resource.
- The proxy parses the absolute URL, verifies that the target domain is permitted, establishes a separate connection to the host, retrieves the content, and passes the HTTP response headers and body back to the client.

```mermaid
sequenceDiagram
    autonumber
    actor Client as Client Container (172.20.0.10)
    participant Proxy as Nginx Forward Proxy (172.20.0.2)
    participant DNS as Docker Resolver (127.0.0.11)
    participant Target as External Host (httpbin.org)

    Client->>Proxy: TCP Connection established on Port 8888
    Client->>Proxy: HTTP GET http://httpbin.org/get HTTP/1.1
    Note over Proxy: Parse Host header (httpbin.org)
    Proxy->>DNS: Resolve DNS for httpbin.org (UDP Port 53)
    DNS-->>Proxy: Resolve IP (e.g., 34.223.124.45)
    Proxy->>Target: Establish TCP Connection on Port 80
    Proxy->>Target: HTTP GET /get HTTP/1.1
    Target-->>Proxy: HTTP/1.1 200 OK (Response Payload)
    Proxy-->>Client: HTTP/1.1 200 OK (Forwarded Payload)
```

### 2. HTTPS Forwarding (CONNECT Tunneling)
For encrypted HTTPS traffic, the proxy cannot decrypt the transmission without dedicated SSL-inspection certificates. Instead, it acts as a blind TCP tunnel using the HTTP CONNECT method.
- The client initiates connection by sending a plain-text CONNECT header to the proxy.
- The proxy reads the target host to evaluate the blocklist.
- If allowed, the proxy establishes a raw TCP socket connection to the target on port 443.
- The proxy returns a confirmation to the client.
- From this point, the proxy operates as a bi-directional pipe, routing raw TCP bytes back and forth. The TLS handshake and subsequent payload encryption take place directly between the client and target server, keeping the data confidential from the proxy.

```mermaid
sequenceDiagram
    autonumber
    actor Client as Client Container (172.20.0.10)
    participant Proxy as Nginx Forward Proxy (172.20.0.2)
    participant Target as External Host (httpbin.org)

    Client->>Proxy: TCP Connection established on Port 8888
    Client->>Proxy: HTTP CONNECT httpbin.org:443 HTTP/1.1
    Note over Proxy: Parse host (httpbin.org) and port (443)
    Proxy->>Target: Establish TCP Connection on Port 443
    Proxy-->>Client: HTTP/1.1 200 Connection Established
    Note over Client,Target: Proxy acts as a raw TCP tunnel
    Client->>Target: Client Hello (Start TLS Handshake)
    Target-->>Client: Server Hello & Certificate Exchange
    Note over Client,Target: TLS Handshake Completed (Encrypted session)
    Client->>Target: Encrypted HTTPS GET /user-agent
    Target-->>Client: Encrypted HTTPS Response
```

---

## Lesson 5: In-Depth Nginx Configuration

Let us look at how the configuration in `nginx/nginx.conf` translates into these architectural patterns.

### 1. Internal DNS Resolution (resolver)
```nginx
resolver 127.0.0.11;
```
Inside a Docker bridge network, Nginx must use Docker's internal DNS resolver located at `127.0.0.11`. Using public DNS like `8.8.8.8` prevents Nginx from resolving other containers on the same network (like `client2`) and can cause `502 Bad Gateway` errors.

### 2. IP-Based Access Control List (ACL)
```nginx
allow 172.20.0.10;
allow 172.20.0.11;
deny all;
```
Nginx evaluates access rules sequentially. Requests coming from client containers with whitelisted IPs (`172.20.0.10` and `172.20.0.11`) are allowed, while requests from other IPs (including the host machine gateway `172.20.0.1`) are denied with `403 Forbidden`.

### 3. Boundary-Safe Content Filtering Regex
```nginx
if ($http_host ~* (^|\.)(facebook\.com|twitter\.com|instagram\.com)(:|$)) {
    return 403 "Access to social media is blocked by company policy\n";
}
```
Using boundary-safe matching (`(^|\.)` at the start and `(:|$)` at the end) prevents false positive blocks. Without these boundary markers, a domain like `myfacebook.com` would match `facebook.com` and be blocked incorrectly.

---

## Lesson 6: Student Review Questions & Challenges

### Review Questions
1. Why is it important to use `127.0.0.11` as the DNS resolver when running Nginx as a forward proxy inside a Docker container?
2. Explain the difference in header visibility between proxying an HTTP request (GET) versus an HTTPS request (CONNECT).
3. If an organization wants to log every URL visited by employees, can they do this with a standard CONNECT proxy without decrypting SSL traffic? Why or why not?

### Lab Challenge
Modify the content filtering regex on port 8889 to block gaming domains (such as `steamcommunity.com` and `roblox.com`) while ensuring that informational blogs about these games (like `robloxblog.com`) remain accessible.
