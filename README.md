# akka-persistence-inmemory #

[![Join the chat at https://gitter.im/dnvriend/akka-persistence-inmemory](https://badges.gitter.im/dnvriend/akka-persistence-inmemory.svg)](https://gitter.im/dnvriend/akka-persistence-inmemory?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
[![Build Status](https://travis-ci.org/dnvriend/akka-persistence-inmemory.svg?branch=master)](https://travis-ci.org/dnvriend/akka-persistence-inmemory)
[![Download](https://api.bintray.com/packages/dnvriend/maven/akka-persistence-inmemory/images/download.svg) ](https://bintray.com/dnvriend/maven/akka-persistence-inmemory/_latestVersion)
[![Codacy Badge](https://api.codacy.com/project/badge/Grade/2cedef156eaf441fbe867becfc5fcb24)](https://www.codacy.com/app/dnvriend/akka-persistence-inmemory?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=dnvriend/akka-persistence-inmemory&amp;utm_campaign=Badge_Grade)
[![License](http://img.shields.io/:license-Apache%202-red.svg)](http://www.apache.org/licenses/LICENSE-2.0.txt)

Akka-persistence-inmemory is a plugin for akka-persistence that stores journal and snapshot messages memory, which is very useful when testing persistent actors, persistent FSM and akka cluster.

## Installation ##
Add the following to your `build.sbt`:

```scala
// the library is available in Bintray's JCenter
resolvers += Resolver.jcenterRepo

libraryDependencies += "com.github.dnvriend" %% "akka-persistence-inmemory" % "1.3.6-RC1"
```

## Contribution policy ##

Contributions via GitHub pull requests are gladly accepted from their original author. Along with any pull requests, please state that the contribution is your original work and that you license the work to the project under the project's open source license. Whether or not you state this explicitly, by submitting any copyrighted material via pull request, email, or other means you agree to license the material under the project's open source license and warrant that you have the legal authority to do so.

## License ##

This code is open source software licensed under the [Apache 2.0 License](http://www.apache.org/licenses/LICENSE-2.0.html).

# Configuration
Add the following to the application.conf:

```
akka {
  persistence {
    journal.plugin = "inmemory-journal"
    snapshot-store.plugin = "inmemory-snapshot-store"
  }
}
```   

# Configuring the query API
The query API can be configured by overriding the defaults by placing the following in application.conf:
 
```
# Optional
inmemory-read-journal {
  # New events are retrieved (polled) with this interval.
  refresh-interval = "100ms"

  # How many events to fetch in one query (replay) and keep buffered until they
  # are delivered downstreams.
  max-buffer-size = "100"
}
```

## Clearing Journal and Snapshot messages
It is possible to manually clear the journal an snapshot storage, for example:

```scala
import akka.actor.ActorSystem
import akka.persistence.inmemory.extension.{ InMemoryJournalStorage, InMemorySnapshotStorage, StorageExtension }
import akka.testkit.TestProbe
import org.scalatest.{ BeforeAndAfterEach, Suite }

trait InMemoryCleanup extends BeforeAndAfterEach { _: Suite =>

  implicit def system: ActorSystem

  override protected def beforeEach(): Unit = {
    val tp = TestProbe()
    tp.send(StorageExtension(system).journalStorage, InMemoryJournalStorage.ClearJournal)
    tp.expectMsg(akka.actor.Status.Success(""))
    tp.send(StorageExtension(system).snapshotStorage, InMemorySnapshotStorage.ClearSnapshots)
    tp.expectMsg(akka.actor.Status.Success(""))
    super.beforeEach()
  }
}
```

## Refresh Interval
The async query API uses polling to query the journal for new events. The refresh interval can be configured
eg. "1s" so that the journal will be polled every 1 second. This setting is global for each async query, so
the _allPersistenceId_, _eventsByTag_ and _eventsByPersistenceId_ queries.

## Max Buffer Size
When an async query is started, a number of events will be buffered and will use memory when not consumed by 
a Sink. The default size is 100.

## How to get the ReadJournal using Scala
The `ReadJournal` is retrieved via the `akka.persistence.query.PersistenceQuery` extension:

```scala
import akka.persistence.query.scaladsl._

lazy val readJournal = PersistenceQuery(system).readJournalFor("inmemory-read-journal")
 .asInstanceOf[ReadJournal
    with CurrentPersistenceIdsQuery
    with AllPersistenceIdsQuery
    with CurrentEventsByPersistenceIdQuery
    with CurrentEventsByTagQuery
    with EventsByPersistenceIdQuery
    with EventsByTagQuery]
```

## How to get the ReadJournal using Java
The `ReadJournal` is retrieved via the `akka.persistence.query.PersistenceQuery` extension:

```java
import akka.persistence.query.PersistenceQuery
import akka.persistence.inmemory.query.journal.javadsl.InMemoryReadJournal

final InMemoryReadJournal readJournal = PersistenceQuery.get(system).getReadJournalFor(InMemoryReadJournal.class, InMemoryReadJournal.Identifier());
```

## Persistence Query
The plugin supports the following queries:

## AllPersistenceIdsQuery and CurrentPersistenceIdsQuery
`allPersistenceIds` and `currentPersistenceIds` are used for retrieving all persistenceIds of all persistent actors.

```scala
import akka.actor.ActorSystem
import akka.stream.{Materializer, ActorMaterializer}
import akka.stream.scaladsl.Source
import akka.persistence.query.PersistenceQuery
import akka.persistence.inmemory.query.journal.scaladsl.InMemoryReadJournal

implicit val system: ActorSystem = ActorSystem()
implicit val mat: Materializer = ActorMaterializer()(system)
val readJournal: InMemoryReadJournal = PersistenceQuery(system).readJournalFor[InMemoryReadJournal](InMemoryReadJournal.Identifier)

val willNotCompleteTheStream: Source[String, NotUsed] = readJournal.allPersistenceIds()

val willCompleteTheStream: Source[String, NotUsed] = readJournal.currentPersistenceIds()
```

The returned event stream is unordered and you can expect different order for multiple executions of the query.

When using the `allPersistenceIds` query, the stream is not completed when it reaches the end of the currently used persistenceIds, 
but it continues to push new persistenceIds when new persistent actors are created. 

When using the `currentPersistenceIds` query, the stream is completed when the end of the current list of persistenceIds is reached,
thus it is not a `live` query.

The stream is completed with failure if there is a failure in executing the query in the backend journal.

## EventsByPersistenceIdQuery and CurrentEventsByPersistenceIdQuery
`eventsByPersistenceId` and `currentEventsByPersistenceId` is used for retrieving events for 
a specific PersistentActor identified by persistenceId.

```scala
import akka.actor.ActorSystem
import akka.stream.{Materializer, ActorMaterializer}
import akka.stream.scaladsl.Source
import akka.persistence.query.scaladsl._

implicit val system: ActorSystem = ActorSystem()
implicit val mat: Materializer = ActorMaterializer()(system)

lazy val readJournal = PersistenceQuery(system).readJournalFor("inmemory-read-journal")
 .asInstanceOf[ReadJournal
    with CurrentPersistenceIdsQuery
    with AllPersistenceIdsQuery
    with CurrentEventsByPersistenceIdQuery
    with CurrentEventsByTagQuery
    with EventsByPersistenceIdQuery
    with EventsByTagQuery]

val willNotCompleteTheStream: Source[EventEnvelope, NotUsed] = readJournal.eventsByPersistenceId("some-persistence-id", 0L, Long.MaxValue)

val willCompleteTheStream: Source[EventEnvelope, NotUsed] = readJournal.currentEventsByPersistenceId("some-persistence-id", 0L, Long.MaxValue)
```

You can retrieve a subset of all events by specifying `fromSequenceNr` and `toSequenceNr` or use `0L` and `Long.MaxValue` respectively to retrieve all events. Note that the corresponding sequence number of each event is provided in the `EventEnvelope`, which makes it possible to resume the stream at a later point from a given sequence number.

The returned event stream is ordered by sequence number, i.e. the same order as the PersistentActor persisted the events. The same prefix of stream elements (in same order) are returned for multiple executions of the query, except for when events have been deleted.

The stream is completed with failure if there is a failure in executing the query in the backend journal.

## EventsByTag and CurrentEventsByTag
`eventsByTag` and `currentEventsByTag` are used for retrieving events that were marked with a given 
`tag`, e.g. all domain events of an Aggregate Root type.

```scala
import akka.actor.ActorSystem
import akka.stream.{Materializer, ActorMaterializer}
import akka.stream.scaladsl.Source
import akka.persistence.query.scaladsl._

implicit val system: ActorSystem = ActorSystem()
implicit val mat: Materializer = ActorMaterializer()(system)

lazy val readJournal = PersistenceQuery(system).readJournalFor("inmemory-read-journal")
 .asInstanceOf[ReadJournal
    with CurrentPersistenceIdsQuery
    with AllPersistenceIdsQuery
    with CurrentEventsByPersistenceIdQuery
    with CurrentEventsByTagQuery
    with EventsByPersistenceIdQuery
    with EventsByTagQuery]

val willNotCompleteTheStream: Source[EventEnvelope, NotUsed] = readJournal.eventsByTag("apple", 0L)

val willCompleteTheStream: Source[EventEnvelope, NotUsed] = readJournal.currentEventsByTag("apple", 0L)
```

To tag events you'll need to create an [Event Adapter](http://doc.akka.io/docs/akka/2.4.1/scala/persistence.html#event-adapters-scala) 
that will wrap the event in a [akka.persistence.journal.Tagged](http://doc.akka.io/api/akka/2.4.1/#akka.persistence.journal.Tagged) 
class with the given tags. The `Tagged` class will instruct the persistence plugin to tag the event with the given set of tags.
The persistence plugin will __not__ store the `Tagged` class in the journal. It will strip the `tags` and `payload` from the `Tagged` class,
and use the class only as an instruction to tag the event with the given tags and store the `payload` in the 
`message` field of the journal table. 

```scala
import akka.persistence.journal.{ Tagged, WriteEventAdapter }
import com.github.dnvriend.Person.{ LastNameChanged, FirstNameChanged, PersonCreated }

class TaggingEventAdapter extends WriteEventAdapter {
  override def manifest(event: Any): String = ""

  def withTag(event: Any, tag: String) = Tagged(event, Set(tag))

  override def toJournal(event: Any): Any = event match {
    case _: PersonCreated ⇒
      withTag(event, "person-created")
    case _: FirstNameChanged ⇒
      withTag(event, "first-name-changed")
    case _: LastNameChanged ⇒
      withTag(event, "last-name-changed")
    case _ ⇒ event
  }
}
```

The `EventAdapter` must be registered by adding the following to the root of `application.conf` Please see the 
[demo-akka-persistence-jdbc](https://github.com/dnvriend/demo-akka-persistence-jdbc) project for more information. The identifier of the persistence plugin must be used which for the inmemory plugin is `inmemory-journal`. 

```bash
inmemory-journal {
  event-adapters {
    tagging = "com.github.dnvriend.TaggingEventAdapter"
  }
  event-adapter-bindings {
    "com.github.dnvriend.Person$PersonCreated" = tagging
    "com.github.dnvriend.Person$FirstNameChanged" = tagging
    "com.github.dnvriend.Person$LastNameChanged" = tagging
  }
}
```

You can retrieve a subset of all events by specifying offset, or use 0L to retrieve all events with a given tag. 
The offset corresponds to an ordered sequence number for the specific tag. Note that the corresponding offset of each 
event is provided in the EventEnvelope, which makes it possible to resume the stream at a later point from a given offset.

In addition to the offset the EventEnvelope also provides persistenceId and sequenceNr for each event. The sequenceNr is 
the sequence number for the persistent actor with the persistenceId that persisted the event. The persistenceId + sequenceNr 
is an unique identifier for the event.

The returned event stream contains only events that correspond to the given tag, and is ordered by the creation time of the events, 
The same stream elements (in same order) are returned for multiple executions of the same query. Deleted events are not deleted 
from the tagged event stream. 

# Some videos about Akka Persistence, streams and Event Sourcing
Is Event Sourcing getting traction? I would say so:

- [Konrad Malawski - Akka Streams & Reactive Streams in action (2016)](https://www.youtube.com/watch?v=x62K4ObBtw4)
- [Björn Antonsson & Konrad Malawski - Resilient Applications with Akka Persistence (2016)](https://www.youtube.com/watch?v=qqNsGomfabc)
- [Aleksei Irbe - Akka persistence (2016)](https://www.youtube.com/watch?v=Oc9lHVVn1YQ)
- [Greg Young - Event Sourcing is actually just functional code (2016)](https://www.youtube.com/watch?v=kZL41SMXWdM)
- [Greg Young — A Decade of DDD, CQRS, Event Sourcing (2016)](https://www.youtube.com/watch?v=LDW0QWie21s)
- [Packt - Introduction to Akka Persistence (2016)](https://www.youtube.com/watch?v=QwsA8hkNGOA)
- [Paweł Szulc - Event Sourcing & Functional Programming - a pair made in heaven (2016)](https://www.youtube.com/watch?v=1rFY2SfdDoE)
- [Konrad Malawski - Akka and the Zen of Reactive System Design (2016)](https://www.youtube.com/watch?v=Mg5ZmoMddJI)
- [Renato Cavalcanti - Field guide to DDD/CQRS using the Scala Type System and Akka (2015)](https://www.youtube.com/watch?v=fQkKu4tTgCE)
- [Martin Zapletal: Data in Motion - Streaming Static Data Efficiently in Akka Persistence (2016)](https://www.youtube.com/watch?v=K4FY0XKediU)
- [Martin Krasser - Event Sourcing and CQRS with Akka Persistence and Eventuate (2015)](https://www.youtube.com/watch?v=vFVry457XLk)
- [Duncan DeVore - CQRS/ES with Scala and Akka Persistence (2015)](https://www.youtube.com/watch?v=uA2AsZW0I7A)
- [Sander Mak - Event-Sourced Architectures with Akka (2015)](https://www.youtube.com/watch?v=gvsRl6xZiiE)
- [Sidharth Khattri - Akka Persistence | Event Sourcing (2015)](https://www.youtube.com/watch?v=yAI71_smS34)
- [Michał Płachta - Building multiplayer game using Reactive Streams](https://www.youtube.com/watch?v=iKTFalVfoSU)
- [Patrik Nordwall - Intro to Akka persistence (2014)](https://www.youtube.com/watch?v=r5lecCBazvE)
- [Greg Young - Event Sourcing(2014)](https://www.youtube.com/watch?v=8JKjvY4etTY)

# What's new?
## 1.3.6-RC1 (2016-08-03)
  - Akka 2.4.8 -> 2.4.9-RC1

## 1.3.5 (2016-07-23)
  - Support for the __non-official__ bulk loading interface [akka.persistence.query.scaladsl.EventWriter](https://github.com/dnvriend/akka-persistence-query-writer/blob/master/src/main/scala/akka/persistence/query/scaladsl/EventWriter.scala)
    added. I need this interface to load massive amounts of data, that will be processed by many actors, but initially I just want to create and store one or
    more events belonging to an actor, that will handle the business rules eventually. Using actors or a shard region for that matter, just gives to much
    actor life cycle overhead ie. too many calls to the data store. The `akka.persistence.query.scaladsl.EventWriter` interface is non-official and puts all
    responsibility of ensuring the integrity of the journal on you. This means when some strange things are happening caused by wrong loading of the data,
    and therefor breaking the integrity and ruleset of akka-persistence, all the responsibility on fixing it is on you, and not on the Akka team.

## 1.3.4 (2016-07-17)
  - Codacy code cleanup release.

## 1.3.3 (2016-07-16)
  - No need for Query Publishers with the new akka-streams API.

## 1.3.2 (2016-07-09)
  - Journal entry 'deleted' fixed, must be set manually.

## 1.3.1 (2016-07-09)
  - Akka 2.4.7 -> 2.4.8,
  - Behavior of akka-persistence-query *byTag query should be up to spec,
  - Refactored the inmemory plugin code base, should be more clean now.

## 1.3.0 (2016-06-09)
  - Removed the queries `eventsByPersistenceIdAndTag` and `currentEventsByPersistenceIdAndTag` as they are not supported by Akka natively and can be configured by filtering the event stream.
  - Implemented true async queries using the polling strategy

## 1.2.15 (2016-06-05)
  - Akka 2.4.6 -> 2.4.7

## 1.2.14 (2016-05-25)
  - Fixed issue Unable to differentiate between persistence failures and serialization issues
  - Akka 2.4.4 -> 2.4.6

## 1.2.13 (2016-04-14)
  - Akka 2.4.3 -> 2.4.4
  
## 1.2.12 (2016-04-01)
  - Scala 2.11.7 -> 2.11.8
  - Akka 2.4.2 -> 2.4.3

## 1.2.11 (2016-03-18)
  - Fixed issue on the query api where the offset on eventsByTag and eventsByPersistenceIdAndTag queries were not sequential  
  
## 1.2.10 (2016-03-17)
  - Refactored the akka-persistence-query interfaces, integrated it back again in one jar, for jcenter deployment simplicity

## 1.2.9 (2016-03-16)
  - Added the appropriate Maven POM resources to be publishing to Bintray's JCenter

## 1.2.8 (2016-03-03)
  - Fix for propagating serialization errors to akka-persistence so that any error regarding the persistence of messages will be handled by the callback handler of the Persistent Actor; `onPersistFailure`.

## 1.2.7 (2016-02-18)
  - Better storage implementation for journal and snapshot

## 1.2.6 (2016-02-17)
  - Akka 2.4.2-RC3 -> 2.4.2

## 1.2.5 (2016-02-13)
  - akka-persistence-jdbc-query 1.0.0 -> 1.0.1

## 1.2.4 (2016-02-13)
  - Akka 2.4.2-RC2 -> 2.4.2-RC3

## 1.2.3 (2016-02-08)
  - Compatibility with Akka 2.4.2-RC2
  - Refactored the akka-persistence-query extension interfaces to its own jar: `"com.github.dnvriend" %% "akka-persistence-jdbc-query" % "1.0.0"`

## 1.2.2 (2016-01-30)
  - Code is based on [akka-persistence-jdbc](https://github.com/dnvriend/akka-persistence-jdbc)
  - Supports the following queries:
    - `allPersistenceIds` and `currentPersistenceIds`
    - `eventsByPersistenceId` and `currentEventsByPersistenceId`
    - `eventsByTag` and `currentEventsByTag`
    - `eventsByPersistenceIdAndTag` and `currentEventsByPersistenceIdAndTag`
  
## 1.2.1 (2016-01-28)
  - Supports for the javadsl query API

## 1.2.0 (2016-01-26)
  - Compatibility with Akka 2.4.2-RC1 

## 1.1.6 (2015-12-02)
 - Compatibility with Akka 2.4.1
 - Merged PR #17 [Evgeny Shepelyuk](https://github.com/eshepelyuk) Upgrade to AKKA 2.4.1, thanks!

## 1.1.5 (2015-10-24)
 - Compatibility with Akka 2.4.0
 - Merged PR #13 [Evgeny Shepelyuk](https://github.com/eshepelyuk) HighestSequenceNo should be kept on message deletion, thanks!
 - Should be a fix for [Issue #13 - HighestSequenceNo should be kept on message deletion](https://github.com/dnvriend/akka-persistence-inmemory/issues/13) as per [Akka issue #18559](https://github.com/akka/akka/issues/18559) 

## 1.1.4 (2015-10-17)
 - Compatibility with Akka 2.4.0
 - Merged PR #12 [Evgeny Shepelyuk](https://github.com/eshepelyuk) Live version of eventsByPersistenceId, thanks!
 
## 1.1.3 (2015-10-02)
 - Compatibility with Akka 2.4.0
 - Akka 2.4.0-RC3 -> 2.4.0
 
## 1.1.3-RC3 (2015-09-24)
 - Merged PR #10 [Evgeny Shepelyuk](https://github.com/eshepelyuk) Live version of allPersistenceIds, thanks!
 - Compatibility with Akka 2.4.0-RC3
 - Use the following library dependency: `"com.github.dnvriend" %% "akka-persistence-inmemory" % "1.1.3-RC3"` 

## 1.1.1-RC3 (2015-09-19)
 - Merged Issue #9 [Evgeny Shepelyuk](https://github.com/eshepelyuk) Initial implemenation of Persistence Query for In Memory journal, thanks!
 - Compatibility with Akka 2.4.0-RC3
 - Use the following library dependency: `"com.github.dnvriend" %% "akka-persistence-inmemory" % "1.1.1-RC3"` 

## 1.1.0-RC3 (2015-09-17)
 - Merged Issue #6 [Evgeny Shepelyuk](https://github.com/eshepelyuk) Conditional ability to perform full serialization while adding messages to journal, thanks!
 - Compatibility with Akka 2.4.0-RC3
 - Use the following library dependency: `"com.github.dnvriend" %% "akka-persistence-inmemory" % "1.1.0-RC3"` 

## 1.1.0-RC2 (2015-09-05)
 - Compatibility with Akka 2.4.0-RC2
 - Use the following library dependency: `"com.github.dnvriend" %% "akka-persistence-inmemory" % "1.1.0-RC2"` 

## 1.0.5 (2015-09-04)
 - Compatibilty with Akka 2.3.13
 - Akka 2.3.12 -> 2.3.13

## 1.1.0-RC1 (2015-09-02)
 - Compatibility with Akka 2.4.0-RC1
 - Use the following library dependency: `"com.github.dnvriend" %% "akka-persistence-inmemory" % "1.1.0-RC1"` 
   
## 1.0.4 (2015-08-16)
 - Scala 2.11.6 -> 2.11.7
 - Akka 2.3.11 -> 2.3.12
 - Apache-2.0 license
       
## 1.0.3 (2015-05-25)
 - Merged Issue #2 [Sebastián Ortega](https://github.com/sortega) Regression: Fix corner case when persisted events are deleted, thanks!
 - Added test for the corner case issue #1 and #2

## 1.0.2 (2015-05-20)
 - Refactored from the ConcurrentHashMap implementation to a pure Actor managed concurrency model

## 1.0.1 (2015-05-16)
 - Some refactoring, fixed some misconceptions about the behavior of Scala Futures one year ago :)
 - Akka 2.3.6 -> 2.3.11
 - Scala 2.11.1 -> 2.11.6
 - Scala 2.10.4 -> 2.10.5
 - Merged Issue #1 [Sebastián Ortega](https://github.com/sortega) Fix corner case when persisted events are deleted, thanks!

## 1.0.0 (2014-09-25)
 - Moved to bintray

## 0.0.2 (2014-09-05)
 - Akka 2.3.4 -> 2.3.6

## 0.0.1 (2014-08-19)
 - Initial Release

Have fun!
