# Processing Microsoft Outlook Exchange Events at Scale Using Kafka

## Background

Our product is a digital workspace and office management SaaS platform used by multiple organizations. One of the core integrations of the platform is with Microsoft Outlook / Exchange, which allows users to create meetings, book rooms, and manage participants directly from our application.

On paper, this sounded straightforward. In reality, handling calendar events coming from Exchange turned out to be far more complex than we initially expected.

## What Went Wrong Initially

Microsoft Exchange emits events whenever something changes in a meeting. This includes obvious actions like creating or cancelling a meeting, but also smaller changes such as updating attendees, modifying time slots, or even internal metadata updates.

What caught us off guard was that a single user action often generated multiple events. For example, modifying a meeting could result in several “update” notifications, sometimes arriving out of order and sometimes duplicated.

At the beginning, we processed these events using traditional background jobs. Events were stored and picked up by scheduled workers that updated the database. This worked fine when traffic was low, but as adoption grew, the cracks started to show.

Problems We Started Seeing

As the number of organizations and users increased, we ran into several issues:

-Background jobs started lagging during peak office hours
-Duplicate and out-of-order events caused older updates to overwrite newer ones
-It became difficult to reason about the final state of a meeting
-Scaling the jobs meant adding more complexity rather than solving the root problem
-At one point, we realized that the issue wasn’t just performance — it was that our mental model of the problem was wrong.

## Rethinking the Problem

We were treating Exchange notifications almost like commands:
“Here is an update, apply it and move on.”

But Exchange wasn’t giving us instructions. It was giving us signals about what had already happened. Those signals formed a continuous stream of events, not a clean, one-event-per-action flow.

Once we looked at the problem this way, it became clear that we needed an event-driven approach that could handle:

Continuous incoming events
-
-Bursts during peak usage
-Duplicate and retry notifications
-Ordering guarantees for events related to the same meeting
-This is what led us to Kafka.
-Introducing Kafka

We introduced Apache Kafka as the core event streaming layer between Microsoft Exchange and our internal processing logic.
-
Exchange events were received using Exchange subscriptions via the Microsoft Graph SDK. A background service handled these notifications and published them directly to Kafka. There was no heavy processing at this stage — Kafka was used to decouple event ingestion from event handling.

Kafka quickly became the buffer that absorbed Exchange’s noisy behavior and gave us control over how and when events were processed.

Partitioning and Ordering Strategy

One of the most important design decisions we made was around partitioning.

Kafka guarantees ordering only within a partition, so we needed a way to ensure that all events related to the same meeting were processed in order. We achieved this by using meetingId as the Kafka message key.

This meant:

-All events for the same meeting always went to the same partition
-Ordering was preserved per meeting
-Different meetings could still be processed in parallel
-This single change eliminated most of the sequencing issues we had seen earlier.

## Event Processing with Background Services

Kafka consumers were implemented using .NET BackgroundServices. Each running instance of the service acted as a Kafka consumer within the same consumer group.

Kafka handled partition assignment automatically. If we needed more throughput, we simply scaled out by running more instances of the service. No redesign was required.

Processing logic focused on correctness rather than speed:

Each event carried a lastModifiedDateTime

Incoming events were compared against the latest processed version

Stale or duplicate events were ignored

Processing was idempotent by design

Data Storage and State Management

Kafka was not treated as the source of truth. Microsoft Exchange remained authoritative.

Our database stored only the latest known state of a meeting, derived from processing the event stream. Kafka acted as a durable buffer and sequencing mechanism, allowing us to replay events when needed and recover cleanly from failures.

## Results and Impact

After moving to this architecture, we saw clear improvements:

Event processing became stable even during peak usage

Ordering issues for meeting updates were eliminated

Duplicate Exchange events no longer caused data corruption

The system scaled horizontally without additional complexity

Debugging and recovery became significantly easier

Most importantly, the system behaved predictably — which had not been the case before.

## Key Learnings

This project reinforced several important lessons:

External systems often produce eventually consistent and noisy events

Ordering guarantees are critical for stateful event processing

Kafka is best used as a streaming log, not a simple queue

Idempotency and version checks are non-negotiable in distributed systems

Scalability is easier when ingestion and processing are decoupled

## Technologies Used

Apache Kafka

.NET BackgroundService

Microsoft Outlook / Exchange (Microsoft Graph SDK)

SQL Server

Event-driven architecture

## Final Reflection

Looking back, the biggest shift wasn’t technical — it was conceptual.
Once we stopped treating Exchange notifications as commands and started treating them as a stream of events, the rest of the architecture fell into place.

Kafka didn’t just solve a scaling problem; it helped us model the problem correctly.


