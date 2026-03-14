# System Design Diagram

```mermaid
flowchart TB

    C[Client Applications]

    subgraph Edge Layer
        APIGW[API Gateway]
        CDN[CDN]
    end

    subgraph Application Layer
        LB[Load Balancer]
        API[Upload API Service]
    end

    subgraph Data Layer
        DB[(PostgreSQL Metadata DB)]
        S3RAW[(Object Storage - Raw Images)]
        S3PROCESSED[(Object Storage - Processed Images)]
    end

    subgraph Async Processing Layer
        MQ[[RabbitMQ Queue]]
        DLQ[[Dead Letter Queue]]
        W[Image Processing Workers]
    end

    C --> APIGW
    APIGW --> LB
    LB --> API

    API --> S3RAW
    API --> DB
    API --> MQ

    MQ --> W
    W --> S3RAW
    W --> S3PROCESSED
    W --> DB

    W --> DLQ

    C --> CDN
    CDN --> S3PROCESSED
```

## Flow Explanation

### Upload Path
- Client sends upload request through API Gateway
- Load Balancer forwards the request to Upload API Service
- Upload API stores the original image in raw object storage
- Upload API stores metadata in PostgreSQL
- Upload API publishes an image processing job to RabbitMQ

### Processing Path
- Worker consumes the processing job from RabbitMQ
- Worker fetches the raw image
- Worker performs resize, crop, watermark, compression, or format conversion
- Worker stores processed image variants in processed object storage
- Worker updates image status and metadata in PostgreSQL
- If processing repeatedly fails, the job is pushed to Dead Letter Queue

### Delivery Path
- Client requests processed images through CDN
- CDN serves cached images where available
- If cache is missed, CDN fetches from processed object storage