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

## Swicket's involvement

[At Swicket](https://swicket.io), we've been fortunate to serve as the chosen registration partner for Devoxx BE for the last couple of years.

## The Swicket Platform

Swicket offers a comprehensive solution for event organizers to efficiently manage attendees for their events.

For each event organizer, we create a dedicated namespace within our multi-tenant Kubernetes cluster, plus a dedicated database (whose content is property of the organizer as stated in our privacy policy) hosted on one of our shared PostgreSQL instances.
Our Kubernetes cluster has strict network policies in place to prevent unauthorized "lateral movements" between namespaces.

Within the customer namespace, we deploy [Alf.io](https://alf.io) to handle registrations alongside other proprietary components that provide additional functionalities, including:
- Integration with external invoicing systems
- Integration with video conferencing software/services (Zoom, GoToWebinar, BigBlueButton)
- Exhibitor Portal for lead collection during the event

For those curious, this is a deployment diagram that visualize our infrastructure on Google Cloud Platform:

<div class="mb-5">
    <a href="/images/devoxxbe/swicket-deployment.png" title="Open in a new Tab" target="_blank">
        <img alt="Swicket's infrastructure" src="/images/devoxxbe/swicket-deployment.png" class="img-fluid" >
    </a>
</div>

### Technical details

Our node pool is composed by `e2-standard-4` instances (4vCPUs, 16GB of RAM), and we use the smallest Cloud SQL PostgreSQL instances available (1vCPU, 3.75GB of RAM).

## Configuration for Devoxx Belgium

To manage the high load, we have configured the Alf.io deployment to have 4 replicas, spread across a total of 3 nodes.
The Alf.io container is assigned a memory request/limit of 1024MB and no CPU requests/limit, ensuring sufficient CPU resources for fast container restarts in case of errors.

## Managing the Load

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

In an effort to ensure more fairness compared to last year's edition, Devoxx BE organizers decided to reduce the maximum number of tickets per reservation from 40 to 10.

The decrease in tickets per reservation (scale down, vertically) triggered an increase in the number of clients trying to reserve (scale out, horizontally). As a result, we recorded an unprecedented load on our cluster.

> &lt;brag-mode&gt; 🍻 We are really proud of how Alf.io and Swicket's infrastructure handled this unexpected surge, even with a single-core database! &lt;/brag-mode&gt;

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

However, we will start a code review to optimize the reservation phase. The goal is to minimize errors under load and improve performances.
In addition to that, we have received a couple of technical feedback which are now marked as [issues](https://github.com/alfio-event/alf.io/issues) on our GitHub repo.

Keep them coming!


### UX Perspective

Some people expressed frustration when they missed out on buying their tickets just because they were a couple of seconds late. We are now evaluating different approaches to improve fairness in the process.
While we have initial ideas, we're eager to hear suggestions from you! Feel free to comment here or post a message on our [discussion board](https://github.com/alfio-event/alf.io/discussions/categories/ideas). Thank you!