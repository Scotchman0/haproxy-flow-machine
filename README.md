```mermaid
graph LR
    subgraph Client
        A[Start Connection]
    end

    subgraph Load Balancer
        B[Receive Connection]
        C[Decrypt TLS]
        D[Load Balance]
        E[Encrypt TLS]
    end

    subgraph Backend
        F[Receive Connection]
    end

    subgraph Client -> Load Balancer
        A --> B
        B --> C
        C --> D
        D --> E
    end

    subgraph Load Balancer -> Backend
        E --> F
    end
```

```mermaid
graph LR
  A[Client] -->|1. ClientHello| B(HAProxy);
  B -->|2. ClientHello| C[Server1 - passthrough];
  C -->|3. ServerHello| B;
  B -->|4. ServerHello| A;
  
  %% Passthrough
  B -->|5. ServerHello| A;
  
  %% Edge
  B -->|5. ClientHello| E[OpenSSL];
  E -->|6. ClientHello| F[Server2 - edge];
  F -->|7. ServerHello| E;
  E -->|8. ServerHello| B;
  B -->|9. ServerHello| A;
  
  %% Re-encrypt
  B -->|5. ClientHello| G[OpenSSL];
  G -->|6. ClientHello| H[Server3 - re-encrypt];
  H -->|7. ServerHello| G;
  G -->|8. ServerHello| B;
  B -->|9. ServerHello| I[OpenSSL];
  I -->|10. ClientHello| H;
  H -->|11. ServerHello| I;
  I -->|12. ServerHello| B;
  B -->|13. ServerHello| A;
```

```mermaid
graph LR
A[Client] --> B(HAProxy Load Balancer)
B --> C{Protocol}
C -->|HTTP| D[Backend Server]
C -->|HTTPS Edge| E(HAProxy terminates SSL)
E --> F[Backend Server]
C -->|HTTPS Passthrough| G(HAProxy passes encrypted traffic to backend)
G --> H[Backend terminates SSL]
C -->|HTTPS Reencrypt| I(HAProxy terminates then encrypts SSL)
I --> J(Backend terminates newly encrypted SSL)
```


===============

# connection lifecycle on unencryped route:
```mermaid
sequenceDiagram
    participant Client
    participant Haproxy
    participant Backend

    Client->>+Haproxy: SYN
    Haproxy->>-Client: SYN-ACK
    Client->>+Haproxy: ACK


    Haproxy->>+Backend: SYN
    Backend->>-Haproxy: SYN-ACK
    Haproxy->>-Backend: ACK
    Client->>+Haproxy: GET / HTTP/1.1
    Haproxy->>-Backend: GET / HTTP/1.1
    Backend->>+Haproxy: HTTP/1.1 200 OK
    Haproxy->>-Client: HTTP/1.1 200 OK

    Client->>+Haproxy: FIN
    Haproxy->>-Client: FIN-ACK
    Haproxy->>+Backend: FIN
    Backend->>-Haproxy: FIN-ACK
    Backend->>+Haproxy: ACK
    Haproxy->>-Backend: ACK
```

# edge encrypted:
```mermaid
sequenceDiagram
  participant Client
  participant HAProxy
  participant Backend

  Client->>+HAProxy: Send GET request
  Note over HAProxy: Terminate TLS\nand decrypt request
  HAProxy->>+Backend: Forward decrypted request
  Backend-->>-HAProxy: Send response
  Note over HAProxy: Encrypt response\nand create TLS tunnel
  HAProxy-->>-Client: Forward encrypted response
  ```

# re-encrypted
```mermaid
sequenceDiagram
  participant Client
  participant HAProxy
  participant Backend

  Client->>+HAProxy: Send encrypted GET request
  Note over HAProxy: Decrypt request\nand re-encrypt using own certificate
  HAProxy->>+Backend: Forward re-encrypted request
  Note over Backend: Terminate TLS\nand decrypt request
  Backend->>+HAProxy: Forward decrypted request
  Note over HAProxy: Encrypt response\nand create TLS tunnel to Client
  HAProxy-->>-Backend: Forward encrypted response
  Note over Backend: Encrypt response\nand create TLS tunnel to HAProxy
  Backend-->>-HAProxy: Forward encrypted response
  Note over HAProxy: Forward re-encrypted response\nto Client
  HAProxy-->>-Client: Forward re-encrypted response

```
# passthrough
```mermaid
sequenceDiagram
  participant Client
  participant HAProxy
  participant Backend

  Client->>+HAProxy: Send encrypted GET request
  HAProxy->>+Backend: Forward encrypted request
  Note over Backend: Terminate TLS\nand decrypt request
  Backend-->>-HAProxy: Send response
  Note over HAProxy: Forward encrypted response\nto Client
  HAProxy-->>-Client: Forward encrypted response
```


================

# sharding

```mermaid
graph LR
  subgraph sharded ingress controller
    shardedRoutes{{routes labeled 'sharded=true', shard-apps.mycluster.mydomain}} --> shardedIC[sharded ingress controller]
    shardedIC --> D[routeD]
    shardedIC --> E[routeE]
    shardedIC --> F[routeF]
    D --> G[pod1]
    D --> H[pod2]
    D --> I[pod3]
    E --> J[pod4]
    F --> K[pod5]
    F --> L[pod5]
  end
  
  subgraph default ingress controller
    unlabeledRoutes{{routes not labeled 'sharded=true', apps.mycluster.mydomain}} --> defaultIC[default ingress controller]
    defaultIC --> A[routeA]
    defaultIC --> B[routeB]
    defaultIC --> C[routeC]
    A --> M[pod6]
    A --> N[pod7]
    B --> O[pod8]
    C --> P[pod9]
    C --> Q[pod10]
    C --> R[pod11]
  end
  
  client[client http/https request] --> unlabeledRoutes
  client[client http/https request] --> unlabeledRoutes
  client[client http/https request] --> unlabeledRoutes
  client[client http/https request] --> shardedRoutes
  client[client http/https request] --> shardedRoutes
  client[client http/https request] --> shardedRoutes
```

