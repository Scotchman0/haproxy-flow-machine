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

```mermaid
sequenceDiagram
  participant Client
  participant HAProxy1
  participant HAProxy2
  participant Backend

  Client->>+HAProxy1: Send GET request
  Note over HAProxy1: Terminate TLS\nand decrypt request
  HAProxy1->>+Backend: Forward decrypted request
  Backend-->>-HAProxy1: Send response
  Note over HAProxy1: Encrypt response\nand create TLS tunnel to HAProxy2
  HAProxy1->>+HAProxy2: Forward TLS tunnel
  Note over HAProxy2: Decrypt tunnel\nand re-encrypt using own certificate
  HAProxy2-->>-HAProxy1: Forward re-encrypted tunnel
  Note over HAProxy1: Forward encrypted response\nto Client
  HAProxy1-->>-Client: Forward encrypted response
```
