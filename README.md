# TingMe Architecture Overview

## 1. Tổng quan kiến trúc

TingMe đang chạy theo mô hình **microservices hướng domain** trên nền NestJS, kết hợp:

- **Sync call**: HTTP + gRPC giữa các service.
- **Async/event-driven**: Outbox + SNS/SQS + consumer workers.
- **Realtime**: WebSocket gateway scale ngang bằng Redis adapter.

Ngoài các service độc lập (`ms-wallet`, `ms-video`, `ms-shopping`, `ms-chat`, `ms-notification`, `ms-airdrop`), repo `tingme-api` đóng vai trò kép:

- `api-gateway`: lớp HTTP/BFF cho client.
- `ms-user`: gRPC service (user/const-config) + HTTP phụ trợ.
- consumer/cron workers cho các tác vụ nền.

## 2. Runtime topology

```mermaid
flowchart LR
    C[Mobile/Web Client]
    AG[API Gateway\n(tingme-api main)]
    WG[WS Gateway]
    RG[RPC Gateway]

    U[MS User\n(tingme-api main-grpc)]
    W[MS Wallet]
    V[MS Video]
    S[MS Shopping]
    N[MS Notification]
    CH[MS Chat]
    A[MS Airdrop]
    BTC[MS BTC Transactions\n(cron worker)]

    SNS[SNS Topics]
    SQS[SQS Queues]
    EB[EventBridge Rules]
    OX[(Outbox Table)]

    PG[(PostgreSQL)]
    MG[(MongoDB)]
    RD[(Redis)]
    ES[(Elasticsearch)]
    S3[(S3 Buckets)]

    C --> AG
    C --> WG
    C --> RG

    AG -->|gRPC| U
    AG -->|gRPC| W
    AG -->|gRPC| V
    AG -->|gRPC| S
    AG -->|gRPC| N
    AG -->|gRPC| CH

    WG -->|consume cdc-notification-created| SQS
    WG --> RD
    RG -->|JSON-RPC proxy| C

    U --> PG
    U --> MG
    W --> PG
    V --> PG
    V --> MG
    S --> PG
    CH --> PG
    N --> MG
    A --> PG
    BTC --> PG

    U --> OX
    W --> OX
    V --> OX
    S --> OX
    CH --> OX
    A --> OX
    BTC --> OX

    OX -->|cron publish| SNS
    SNS --> SQS
    SQS -->|consumer modules| U
    SQS --> W
    SQS --> V
    SQS --> S
    SQS --> N
    SQS --> CH
    SQS --> A
    SQS --> BTC

    MG -->|partner event bus| EB
    EB --> SNS

    U --> ES
    W --> ES
    V --> ES
    S --> ES

    U --> S3
    W --> S3
    V --> S3
    S --> S3
```

## 3. Service boundaries (theo code hiện tại)

| Service | Vai trò chính | Ghi chú |
|---|---|---|
| `api-gateway` (`tingme-api` main) | API HTTP cho client, auth/guard, gọi downstream qua gRPC | Có Swagger, throttling, global guards |
| `ms-user` (`tingme-api` main-grpc) | User/account/const-config và domain liên quan người dùng | Triển khai riêng trong `k8s/ms-user` |
| `ms-wallet` | Wallet, token, chain, NFT market, token transactions | gRPC package lớn nhất |
| `ms-video` | Video/feed/hashtag/search/media pipeline | Có luồng kiểm duyệt + MediaConvert |
| `ms-shopping` | Product + product-order | Kết nối gRPC sang video |
| `ms-chat` | Conversation/message | gRPC chat service |
| `ms-notification` | Notification domain + CDC consumers | Dữ liệu notification trên Mongo |
| `ms-airdrop` | KYC/reward/airdrop campaign | Gọi sang `ms-user` và `ms-wallet` qua gRPC |
| `ws-gateway` | Push realtime notification qua socket namespace `notifications` | JWT handshake + Redis socket adapter |
| `rpc-gateway` | Proxy JSON-RPC blockchain (public/private endpoint) | Route theo chain (`polygon`, `ethereum`, `bsc`, ...) |
| `ms-btc-transactions` | Worker cron xử lý BTC transaction pipeline | Hiện thấy deployment cron trong k8s |

## 4. Communication patterns

### 4.1 Synchronous

- Client -> `api-gateway` (HTTP).
- `api-gateway` -> downstream services (gRPC, thông qua `ClientGrpcProxy`).
- `rpc-gateway` route JSON-RPC request theo method:
  - method public (`eth_call`, `eth_getBalance`) -> public RPC.
  - method private (`eth_sendTransaction`, ...) -> private RPC.

### 4.2 Asynchronous (event-driven)

- Các service ghi event vào bảng `outboxes`.
- Cron job `EVERY_SECOND` publish event mới lên SNS/SQS qua `QueueService`.
- Consumer modules của từng service subscribe topic tương ứng và xử lý eventual consistency.

## 5. Data & state architecture

- **PostgreSQL (TypeORM)**: transaction data/domain entities chính của phần lớn service.
- **MongoDB (Mongoose)**: notification/video metadata và một số stream/config flow.
- **Redis**: cache + distributed socket adapter.
- **Elasticsearch**: search/use-cases truy vấn nhanh theo domain.
- **S3**: assets, original/converted/paid videos, config artifacts.

## 6. Shared modules (platform layer)

Các package dùng chung trong repo:

- `module-common`: helpers, filter/exceptions, utilities.
- `module-cache`: Redis abstraction.
- `module-file`: S3/file abstraction.
- `module-search`: Elasticsearch abstraction.
- `module-queue`: unified queue interface (SQS/Rabbit/Postgres implementations; production đang dùng SQS).

## 7. Deployment architecture

### Kubernetes (`tingme-api/k8s`)

- Mỗi service có chart riêng (`ms-user`, `ms-wallet`, `ms-video`, `ms-shopping`, `ms-chat`, `ms-notification`, `ms-airdrop`, `ms-btc-transactions`, `api-gateway`, `ws-gateway`, `rpc-gateway`).
- `kustomization.yml` generate secrets theo từng service.
- `api-gateway`, `ws-gateway`, `rpc-gateway` dùng DaemonSet.
- Domain services chủ yếu dùng Deployment.

### Terraform (`Infrastructure/`)

- VPC + public/private subnets.
- RDS PostgreSQL (Multi-AZ).
- ElastiCache Redis.
- Elasticsearch/OpenSearch domain.
- S3 buckets + notifications.
- SNS/SQS topics, queue fan-out.
- EventBridge rules cho event bus partner (MongoDB trigger) và media events.
