---
title: "Behind the Scenes of Devoxx Ticket Sales"
date: 2023-08-17T12:00:00+0100
draft: false
github_link: "https://github.com/alfio-event/alf.io"
author: "Celestino"
tags:
- Java
- Kubernetes
- PostgreSQL
- Performances
image: /images/devoxxbe/cover-dvbe19.jpg
description: "How we managed the high load of registrations for Devoxx Belgium"
summary: "How we managed the high load of registrations for Devoxx Belgium."
images: ["/images/devoxxbe/cover-dvbe19.jpg"]
toc:
---

## Introduction

[Devoxx Belgium](https://devoxx.be) is a developer community conference held in Antwerp, Belgium. This five-day event is renowed for its content, with researchers, scientists, industry leaders, and fellow developers presenting sessions suitable for diverse audience levels.

It is considered a "must attend" event by the community, and seats are limited. No wonder people cannot wait to secure their ticket!

Tickets are usually available in two batches. Last year, the first batch of tickets was sold in less than 5 minutes, this year for the same batch the ticket sales lasted **30 seconds**.

### Alf.io, the open source ticket reservation system

[Alf.io](https://alf.io) is a free and open source event attendance management system, developed for event organizers who care about privacy, security and fair pricing policy for their customers.
It features an ecosystem of tools to cover the lifecycle of an event from ticket distribution, to event management, to reporting.

Several conferences in the Devoxx Network, including Devoxx Belgium, use [Alf.io](https://alf.io) as their registration / check-in tool.

### Swicket's involvement

[At Swicket](https://swicket.io), our job is to provide Alf.io (among other tools) "As a Service".
Swicket is managed by the same people who develop [Alf.io](https://alf.io) and we've been fortunate to serve as registration partner for various conferences of the Devoxx network (Belgium, United Kingdom, France, Morocco) for the last couple of years.


## The Swicket Platform

[Swicket](https://swicket.io) offers a comprehensive solution for event organizers to efficiently manage attendees for their events.

For each event organizer, we create a dedicated namespace within our multi-tenant Kubernetes cluster, plus a dedicated database (whose content is property of the organizer as stated in our privacy policy) hosted on one of our shared PostgreSQL instances.
Our Kubernetes cluster has strict network policies in place to prevent unauthorized "lateral movements" between namespaces.

Within the customer namespace, we deploy [Alf.io](https://alf.io) to handle registrations alongside other proprietary components that provide additional functionalities, including:

- Integration module with external invoicing systems
- Integration module with video conferencing software/services (Zoom, GoToWebinar, BigBlueButton)
- Swicket's Exhibitor Portal for lead collection during the event

For those curious, this is a deployment diagram that visualize our infrastructure on Google Cloud Platform:

<div class="mb-5">
    <a href="/images/devoxxbe/swicket-deployment.png" title="Open in a new Tab" target="_blank">
        <img alt="Swicket's infrastructure" src="/images/devoxxbe/swicket-deployment.png" class="img-fluid" >
    </a>
</div>

### Technical details

Our node pool is composed by `e2-standard-4` instances (4vCPUs, 16GB of RAM), and we use the smallest Cloud SQL PostgreSQL instances available (1vCPU, 3.75GB of RAM).

## How we prepared for the load

### Alf.io Deployment configuration

We have configured the Alf.io deployment to have 4 replicas, spread across a total of 3 nodes.
The Alf.io container is assigned a memory request/limit of 1024MB and no CPU requests/limit, ensuring sufficient CPU resources for fast container restarts in case of errors.

### Refactoring the Invoice Number Service

In compliance with Belgian law, companies are required to use sequential numbers when generating invoices. Cancelled invoice numbers need to be recycled to maintain the sequential integrity.

While this country-specific logic is not supported by alf.io, [Stephan Jannsen](https://twitter.com/stephan007), the organizer of Devoxx Belgium, developed an external microservice that implements this particular requirements.

This microservice is invoked by alf.io whenever there's a need to create a new invoice, leveraging the capabilities of our [Extension Engine](https://alf.io/docs/reference/extensions/).

During the past year, we identified a potential concurrency issue within this component. To address this, Stephan decided to improve this service with the goal of eliminating any potential race conditions. 
For further details, you can refer to his [initial blog post](https://www.linkedin.com/pulse/invoice-generation-logic-devoxx-stephan-janssen) and the subsequent [follow-up post](https://www.linkedin.com/pulse/devoxx-invoice-generator-v2-stephan-janssen), which incorporates suggestions contributed by members of the Devoxx community.


## Everything happened in 30 seconds

During the critical **30-second interval** following the start of ticket sales (9:00:00 to 9:00:30 CEST), we received a total of **6'129** user-generated requests (excluding polling requests originated by alf.io itself):

- 6'069 "successful" responses ("200, you're in!" or "412, sorry tickets cannot be reserved")
- 60 failures (HTTP 500)

you can see the peak in the following histogram:

<div class="mb-5">
    <a href="/images/devoxxbe/load-histogram.png" title="Open in a new Tab" target="_blank">
        <img alt="Number of requests / 30s" src="/images/devoxxbe/load-histogram.jpg" class="img-fluid" >
    </a>
</div>

The Database CPU usage also reflects the load:

<div class="mb-5">
    <a href="/images/devoxxbe/db-load.png" title="Open in a new Tab" target="_blank">
        <img alt="Swicket's infrastructure" src="/images/devoxxbe/db-load.png" class="img-fluid" >
    </a>
</div>

### The reasons behind this load 

In an effort to ensure more fairness compared to last year's edition, Devoxx BE organizers decided to reduce the maximum number of tickets per reservation from 50 to 10.

The decrease in tickets per reservation (scale down, vertically) triggered an increase in the number of clients trying to reserve (scale out, horizontally). As a result, we recorded an unprecedented load on our cluster.

> &lt;brag-mode&gt; 🍻 We are really proud of how Alf.io and Swicket's infrastructure handled this unexpected surge, even with a single-core database! &lt;/brag-mode&gt;

> &lt;brag-mode&gt; 🍻 The Invoice Number Service performed extremely well under load, and generated hundreds of invoice numbers without problems! &lt;/brag-mode&gt;

## (More) Technical details: ticket reservation flow

Alf.io's aim is to operate efficiently even on modest hardware. The application does not require lots of CPU or RAM, and delegates work to the database as much as possible to maximize throughput. This is true especially when users try to reserve tickets.

Our algorithm is straightforward: we treat the "ticket" table as a queue, and we ensure that it contains exactly one record for each seat defined by the organizers for a given event.
This check is performed on creation / update of the event configuration.

When a user submits a ticket reservation request, the application attempts to "select" the requested number of tickets:

```sql
select id from ticket where [...] for update skip locked
```
[original statement on GitHub](https://github.com/alfio-event/alf.io/blob/master/src/main/java/alfio/repository/TicketRepository.java#L76)

The use of `skip locked` ([reference](https://www.postgresql.org/docs/11/sql-select.html#SQL-FOR-UPDATE-SHARE)) is crucial, as it instructs Postgres to skip already locked rows, potentially returning an empty result.

If the query returns the expected number of results, users can proceed to the next step. Otherwise, an HTTP 412 error is triggered.


## Conclusions


### Technical Perspective

We have proven that we are on the right path for delivering a smooth experience even under big loads.
Since the very beginning we have chosen a simple design and delegated as much work as possible to the database, and this has proven to be a good choice for scalability and performance.

However, we will start a code review to optimize the reservation phase. The goal is to minimize errors under load and improve performances.
In addition to that, we have received a couple of technical feedback which are now marked as [issues](https://github.com/alfio-event/alf.io/issues) on our GitHub repo.

Keep them coming!


### UX Perspective

Some people expressed frustration when they missed out on buying their tickets just because they were a couple of seconds late. We are now evaluating different approaches to improve fairness in the process.

Our initial idea would be to allow people to pre-register to a waitlist and then allocate tickets through a random selection process.

However, we're eager to hear suggestions from you! Feel free to comment here or post a message on our [discussion board](https://github.com/alfio-event/alf.io/discussions/categories/ideas). Thank you!