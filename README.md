# Scalable Image Upload, Processing, and Delivery System

## Overview

This repository presents the system design of a scalable image upload, processing, storage, and delivery platform.

The goal of the system is to support high-volume image uploads, background image processing, durable storage, and low-latency content delivery at scale.

This design is suitable for platforms such as:

- marketplaces
- social media products
- media libraries
- content platforms
- listing or classified apps

---

## Problem Statement

Design a system that allows users to upload and retrieve images efficiently.

The system should support:

- single image uploads
- bulk image uploads
- image transformations such as resize, crop, and watermark
- asynchronous image processing
- global image delivery through CDN
- scalable storage for millions of images

The system must handle:

- **100,000+ uploads per day**
- **10M+ stored images**
- high read traffic for image delivery
- fault tolerance during upload and processing
- future horizontal scaling

---

## Functional Requirements

### Core Requirements

- Users can upload a single image
- Users can upload multiple images in bulk
- System stores original uploaded images
- System generates processed variants
- Users can retrieve processed image URLs
- System tracks processing status for each image

### Image Transformation Requirements

- Resize
- Crop
- Watermark
- Compression
- Format conversion

### Operational Requirements

- Failed processing jobs should be retried
- Large upload spikes should not break the system
- Slow image processing should not block uploads

---

## Non-Functional Requirements

- high availability
- horizontal scalability
- durability
- low-latency image delivery
- fault tolerance
- maintainability
- observability
- cost-efficient storage

---

## High-Level Design

The system separates responsibilities into distinct layers:

1. **API Layer** for request handling
2. **Storage Layer** for raw and processed images
3. **Metadata Layer** for structured records
4. **Queue Layer** for decoupled asynchronous processing
5. **Worker Layer** for image transformations
6. **Delivery Layer** for CDN-based image serving

See the system diagram here:

[Architecture Diagram](architecture.md)

---

## High-Level Flow

### Upload Flow

1. Client sends upload request
2. Request passes through API Gateway
3. Load Balancer forwards request to Upload API
4. Upload API stores raw image in object storage
5. Upload API stores metadata in PostgreSQL
6. Upload API publishes processing job to RabbitMQ
7. Worker consumes the job
8. Worker generates transformed image variants
9. Processed variants are stored in object storage
10. Metadata status is updated
11. Images are served through CDN

---

## Core Components

### 1. Client Applications

These may include:

- web clients
- mobile apps
- internal admin tools
- partner integrations

Clients are responsible for submitting uploads and consuming final image URLs.

---

### 2. API Gateway

The API Gateway is the public entry point to the system.

#### Responsibilities

- authentication
- authorization
- rate limiting
- request validation
- request routing
- abuse protection

#### Why it exists

It centralizes cross-cutting concerns so backend services remain focused on business logic.

---

### 3. Load Balancer

The load balancer distributes traffic across multiple Upload API instances.

#### Responsibilities

- distribute traffic
- remove unhealthy instances
- support horizontal scaling
- improve availability

---

### 4. Upload API Service

This service handles upload-related business logic.

#### Responsibilities

- accept upload requests
- validate metadata
- create image records
- store upload metadata
- publish processing jobs
- return image references and statuses

#### Design Principle

The Upload API should remain lightweight. It should not do CPU-heavy image transformation work directly.

That work is offloaded to asynchronous workers.

---

### 5. Object Storage

Object storage is used for storing image files.

#### Stores

- raw/original uploaded images
- processed image variants

#### Why object storage

It is a better fit than relational databases for large binary files because it provides:

- high durability
- low cost
- simple scalability
- CDN integration
- lifecycle policy support

#### Example Storage Paths

```text
/raw/{user_id}/{image_id}/original.jpg
/processed/{user_id}/{image_id}/thumbnail.jpg
/processed/{user_id}/{image_id}/medium.jpg
/processed/{user_id}/{image_id}/watermarked.jpg
```

---

### 6. Metadata Database (PostgreSQL)

The metadata database stores structured information about each image.

#### Example fields

- image_id
- user_id
- original_file_name
- raw_image_path
- processing_status
- created_at
- updated_at
- error_reason
- generated_variants

#### Why PostgreSQL

PostgreSQL is a strong fit because:

- metadata is relational and structured
- transactional consistency is useful
- querying by user, time, status, and image is common

---

### 7. Message Queue (RabbitMQ)

RabbitMQ is used to decouple upload handling from image processing.

#### Responsibilities

- queue background processing jobs
- absorb upload spikes
- support retries
- prevent API blocking

#### Why a queue is needed

Image processing can be expensive and slow. Without a queue, upload requests would become slow and fragile.

The queue makes the system more scalable and resilient.

---

### 8. Image Processing Workers

Workers consume jobs from the queue and perform transformations.

#### Responsibilities

- fetch raw image
- generate image variants
- compress files
- apply watermarks
- convert formats
- upload processed outputs
- update metadata

#### Why workers are separate

This separation allows:

- independent scaling
- fault isolation
- faster upload response times
- easier extension of processing logic

---

### 9. CDN

The CDN serves processed images to end users.

#### Benefits

- low-latency image delivery
- global caching
- reduced origin load
- better scalability under high read traffic

---

## Detailed Design Decisions

### Why Asynchronous Processing

Heavy transformations should not happen in the upload request path because:

- request latency becomes unpredictable
- upload throughput drops
- failures become harder to recover from

Asynchronous processing improves performance and reliability.

---

### Why Separate Raw and Processed Images

Keeping raw and processed images separate helps with:

- reprocessing images later
- generating new variants
- debugging failures
- preserving original quality

---

### Why Use Metadata Status

Each image should have processing states such as:

- uploaded
- queued
- processing
- completed
- failed

This improves system visibility and makes retries manageable.

---

## Scaling Strategy

### API Layer Scaling

Multiple Upload API instances can run behind the load balancer.

This layer scales based on:

- request volume
- upload concurrency
- CPU and memory usage

### Worker Layer Scaling

Workers can scale independently based on:

- queue depth
- average processing time
- CPU usage

This is one of the most important design choices because upload traffic and processing demand often scale differently.

### Storage Scaling

Object storage provides near-unlimited storage growth.

### Read Scaling

The CDN absorbs most image read traffic and protects the origin.

---

## Reliability and Fault Tolerance

### Retry Mechanism

Transient failures should be retried automatically.

Examples:

- network failure
- temporary object storage failure
- timeout in processing library

### Dead Letter Queue

Repeatedly failing jobs should move to a dead letter queue for investigation.

### Health Checks

API and worker services should expose health endpoints for orchestration and recovery.

### Multi-Instance Deployment

Run multiple instances of APIs and workers to avoid single points of failure.

---

## Security Considerations

Because the system accepts user-uploaded content, security is important.

### Recommended Controls

- authenticate upload requests
- authorize image access
- validate file type and size
- scan uploaded files for malware
- encrypt data in transit
- encrypt data at rest
- use signed URLs for private access
- restrict direct storage access

---

## Observability

A production-grade system should include:

- request metrics
- queue depth metrics
- worker success/failure metrics
- processing latency metrics
- API logs
- distributed tracing
- alerting for failed jobs and queue backlog

---

## Bottlenecks and Trade-Offs

### RabbitMQ vs Kafka

RabbitMQ is better suited for job-based processing and worker queues.

Kafka is stronger for event streaming and replay-heavy architectures.

For this system, RabbitMQ is a simpler and more natural fit.

### PostgreSQL vs NoSQL

PostgreSQL is appropriate because metadata is structured and consistency matters.

A NoSQL solution could be explored later if access patterns become extremely high scale and simpler.

### Pre-Generate vs On-Demand Variants

Pre-generating common variants improves read performance but increases storage cost.

Generating on demand reduces storage usage but increases first-request latency.

A hybrid model is often best.

---

## Future Improvements

Possible next steps include:

- direct upload using pre-signed URLs
- image deduplication using content hashing
- AI moderation and tagging
- multi-region replication
- lifecycle rules for cold storage
- on-demand transformation service
- autoscaling workers based on queue depth
- regional CDN optimization

---

## Final Summary

This system is designed around a simple principle:

**keep uploads fast, move heavy work to the background, store files durably, and serve them efficiently at scale.**

By separating upload handling, metadata storage, background processing, and content delivery, the architecture remains:

- scalable
- reliable
- maintainable
- production-friendly

It is a strong foundation for any platform that needs to handle large-scale image upload and delivery.