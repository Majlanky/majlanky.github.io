---
layout: full_single
title:  "Debezium Connect with Postgres"
tagline: "whole picture capture"
header:
  overlay_image: /assets/images/magicstudio-art.jpg
  particles: true
date:   2024-10-31 15:00:00
words_per_minute: 70
categories: debezium kafka-connect cdc postgres
---

Oh, what a magical tool that CDC. The idea that you do not need to implement the whole outbox pattern by your own. And what 
more you can pick from more implementations of CDC idea. One of the implementations is Debezium. Solid documentation, varieties
of possible installation and setup, what we want more? Nothing and everything I say. In this post we will dive into real installation
of Debezium connector to Kafka Connect together with Postgres. During the process I find out that I do not fully understand 
concept of Postgres logical replication and Kafka Connect idea that makes difficult to maintain consequences of Postgres 
and Debezium implementation when used together. 

If you are here, because you are trying to find what is going on in your system, do not miss the [troubleshooting](#troubleshooting)

## Basics

I will not explain every technology itself, mainly I will just focus on setting baseline for understanding context of this post.
First of all we will be discussing the following setup:
* [Debezium Connect](https://hub.docker.com/r/debezium/connect) or something based on Kafka Connect containing Debezium connectors
* In the (Kafka) Connect we will use [Postgres connector](https://debezium.io/documentation/reference/stable/connectors/postgresql.html)
* [Postgres](https://www.postgresql.org/)
* [pgoutput](https://www.postgresql.org/docs/current/protocol-logical-replication.html)
* [Kafka](https://kafka.apache.org/)

The overall idea is Debezium Connector is tracking changes in Postgres table(s) and created messages in a topic which
can be used as source of data for others.

## Advanced basics

### Postgres

The overall idea is easy to say, kinda fast to get the idea but the devil is in the details. So, what is hidden. In our particular
setup, Debezium leverage from Postgres logical replication means we need to understand the following
* Publication
* Replication slot
* WAL

In upcoming section I will try to explain in simple form. If you want to understand it even more, you need to dive deeper in documentation.

#### WAL

WAL stands for write access log and contains changes done in DB. It is sequence of by LSN identified changes/transaction containing 
an information about the change itself.

#### Publication

Publication is something we would call producer in Kafka terminology. It manages definition what changes or more specifically what tables 
will be contained in replication, and we can say it produces the events. What publication is not responsible for is the transfer of the events.
Publication can be shared.

#### Replication slot

Because publication is not able to deliver, replications slots emerge. Replication slot is responsible for delivery of produced message to 
the destination. It holds the status of transmission. When we use Kafka technology replication slot would be topic but differently from topic
there can be only one consumer because offset is tracked not for a consumer group aside but directly in replication slot. For tracking 
transmission replication slot uses LSN which is identifier of change/transaction in WAL. Replication is aware what replication it is doing
(logical or different one) and manages translation from WAL language to understandable form for subscribers via plugin (`pgoutput` in our case).
Replication slot can not be reused by more subscribers/consumers.

### Kafka connect

What is Kafka connect is not in the scope of this post. Nevertheless, there is one aspect we need to point out for everybody. Connectors
of source type (Debezium Postgres connector in our case) still ensures strictly that no information is lost in any case. For us this idea
explains why Debezium Postgres connector commits LSN only when some message is produced to Kafka topic. Remember this one for later.

## Real life scenarios

When I was designing architecture of microservice system that leverages from CDC, I was thinking this way:

* I want microservices that is responsible for the data to be declaring the CDC
  * ideally in automated way synchronized with database migrations if present
* I want to have CDC decoupled based on data ownership 
  * this point is connected to the previous point
* I do not want to use dedicated Debezium server to have the solution as lightweight as possible

### Real solution for a real life #1

The previous desires lead be to the following design:
* The same way we split responsibility for data across the microservices, we will split responsibility for CDC between more connectors.
  * In my case I tend to applications maintain connectors, so I can bear connector definitions together with migration scripts in a release
* Every connector will track one (main) entity
* Every connector will use its own replication slot (no other way because of how it works)
* Every connector will define its one tightly scoped publication (filtered auto-creation by Debezium wording)

Even it is look like solidly decoupled idea, it is pure nightmare. The horror appears after some time, and it is caused by hidden aspects of 
technologies used.

#### Pitfall #1

The hidden and dangerous aspect here is the possibility close to certainty that you system will create huge WAL. It will because by the fact
some entities are replicated often and others very rarely. This inconsistency will cause that some publication does not publish anything, means
connector subscribed to changes there is not producing any messages to topic and based on the [behavior](#kafka-connect) described before, 
it does not commit LSN to replication slot. **Replication slot then holds WAL from cleanup, and it swells more and more.**

### Real solution for a real life #2

I have learned from my mistakes and changed the solution the following way (changes are highlighted by bold):
* The same way we split responsibility for data across the microservices, we will split responsibility for CDC between more connectors.
    * In my case I tend to applications maintain connectors, so I can bear connector definitions together with migration scripts in a release
* Every connector will track one (main) entity
* Every connector will use its own replication slot (no other way because of how it works)
* Every connector will define its one tightly scoped publication (filtered auto-creation by Debezium wording)
* **Every connector will use heartbeat table (together with heartbeat.action.query) that will be tracked.**

#### Pitfall #2

Differently from the [first pitfall](#pitfall-1) this solution does not produce huge WAL because periodic heartbeat triggers changes in
tracked heartbeat table, but it messes up the topic by heartbeat CDC messages or forces us to somehow clean the topic by SMT (single message
transformations). Aside to the mess definition of the connector start to be tricky because you are dependent on a table that is not part
of business solution.

### Real solution for a real life #3

I have learned from my mistakes and changed the solution the following way (changes are highlighted by bold):
* The same way we split responsibility for data across the microservices, we will split responsibility for CDC between more connectors.
  * In my case I tend to applications maintain connectors, so I can bear connector definitions together with migration scripts in a release
* Every connector will track one (main) entity
* Every connector will use its own replication slot (no other way because of how it works)
* **Connectors will share one publication** (all_tables by Debezium wording)
* **Every connector will filter message based on `table.include.list`**

#### Pitfall #3

There is nothing on top like heartbeat table which is good. Nevertheless, this solution tends to create huge WAL again because every 
connector that filters out all irrelevant messages from shared publication, does not produce any message to any topic means no LSN commit.

### Real solution for a real life #4

I have learned from my mistakes again and changed the solution the following way (changes are highlighted by bold):
* The same way we split responsibility for data across the microservices, we will split responsibility for CDC between more connectors.
  * In my case I tend to applications maintain connectors, so I can bear connector definitions together with migration scripts in a release
* Every connector will track one (main) entity
* Every connector will use its own replication slot (no other way because of how it works)
* Connectors will share one publication (all_tables by Debezium wording)
* Every connector will filter message based on `table.include.list`
* **Enable heartbeat for connector without action.**

#### Pitfall #4

Everything looks OK on the first sight (or maybe not, it depends). The issue here is that Debezium connector is not committing LSN when
there are no changes from publication. In our case the publication is created for all tables, and it is the aspect that matters. There are
hints (maybe for somebody it is obvious more than for me) that warns that using CDC in environment where one DB is fed a lot and second no,
it will lead to WAL swelling (sWALing?). It is exactly because publication is DB-related, WAL is not. So when one DB is reporting changes to
its publication(s) and publication(s) in the other DB reports nothing, replication slot related to the silent publication just blocks WAL 
from release even with heartbeats.

### Real solution for a real life #5

In this scenario we will use all weapons and tools provided, and I was avoiding at the beginning but by the gradual progress I end with using
them but in IMHO the meaningful and optimal way.

* The same way we split responsibility for data across the microservices, we will split responsibility for CDC between more connectors.
  * In my case I tend to applications maintain connectors, so I can bear connector definitions together with migration scripts in a release
* Every connector will track one (main) entity
* Every connector will use its own replication slot (no other way because of how it works)
* Connectors will share one publication (all_tables by Debezium wording)
* Every connector will filter message based on `table.include.list`
* Enable heartbeat for connector
* **Add heartbeat action**

#### Pitfall-less?

Following official documentation, you can add `heartbeat.action.query`. This query should change a state of table tracked by publication.
In our case we are using publication for all tables, which mean the query can change anything anywhere in the DB, and it will do the magic.
To make connector independent of environment avoiding any manual preparation I designed the query the way it will create a table with generic
name containing one column with timestamp and update this timestamp by the query. So the snippet looks like this:

```json
{
  ...
  "heartbeat.action.query": "CREATE TABLE IF NOT EXISTS trololo (last_heartbeat_ts TIMESTAMP); UPDATE public.debezium_heartbeat SET last_heartbeat_ts = NOW();"
}
```

With this setting you need just generic publication in every DB that will be captured, everything else is done by connector and WAL is not
swelling

## Conclusion

After reading this whole think I can share my simplification what CDC in our context is.
```
Our CDC is translator, plugin above plugin we can say, of Postgres logical replication hence it bears all bad and good of it.
```

## Troubleshooting

Here I will just throw on you some nifty resources.

```postgresql
SELECT
    slot_name,
    plugin,
    slot_type,
    database,
    wal_status,
    active,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS data_left_to_process,
    restart_lsn AS last_commit_lsn
FROM pg_replication_slots;
```
This query will show you all replication slots and how much WAL they are blocking. It can show you if your WAL is swelling and which one 
exactly.
The most important info here aside the size of retained WAL are those:
* wal_status - tells you if the WAL size is over the configured max WAL size or still under the threshold (reserved is the prior one)
* active - if it is not active, means you consumer/subscribed died, hence nobody is here to commit LSN which would free WAL

```postgresql
SELECT *
FROM pg_stat_replication;
```
Simple but very helpful. This query will show you detail how the replications goes. The important information in the result are:
* sent_lsn - LSN that very sent to consumer
* write_lsn - LSN that very written to/accepted by consumer
* flush_lsn - LSN was replicated to the consumer WAL
* replay_lsn - LSN that very committed to consumer

Especially when flush and replay LSN is missing, it means that your connector is not commiting LSN. It happens for example in the described
[case](#pitfall-4) 

## Some other tricky part I hit

* When you use org.apache.kafka.connect.transforms.RegexRouter to get static name of topic like this:
```json
    "transforms": "staticTopic",
    "transforms.staticTopic.type": "org.apache.kafka.connect.transforms.RegexRouter",
    "transforms.staticTopic.regex": ".*",
    "transforms.staticTopic.replacement": "staticTopic"
```
it will ruin heartbeat separation to different topic which is default behavior because everything will end in the same topic. It is logical 
but burned my fingers when trying to have source of truth with static name.





