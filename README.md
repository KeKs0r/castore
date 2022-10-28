<p align="center">
    <img src="assets/logo.svg" height="128">
    <h1 style="border-bottom:none;font-size:60px;margin-bottom:0;" align="center" >Castore</h1>
</p>
<p align="center">
  <a aria-label="NPM version" href="https://www.npmjs.com/package/@castore/core">
    <img alt="" src="https://img.shields.io/npm/v/@castore/core?color=935e0e&style=for-the-badge">
  </a>
  <a aria-label="License" href="https://github.com/castore-dev/castore/blob/main/LICENSE">
    <img alt="" src="https://img.shields.io/github/license/castore-dev/castore?color=%23F8A11C&style=for-the-badge">
  </a>
    <img alt="" src=https://img.shields.io/npm/dt/@castore/core?color=%23B7612E&style=for-the-badge>
    <br/>
    <br/>
</p>

# Making Event Sourcing easy 😎

Event Sourcing is a data storage paradigm that saves **changes in your application state** rather than the state itself. It is powerful but tricky to implement.

<!-- TODO: SCHEMA OF EVENT SOURCING -->

After years of using it at [Kumo](https://dev.to/kumo), we have grown to love it, but also experienced first-hand the lack of consensus and tooling around it. That's where Castore comes from!

<p align="center">
Castore is a TypeScript library that <b>makes Event Sourcing easy</b> 😎
</p>

With Castore, you'll be able to:

- Define your [event stores](#eventstore)
- Fetch and push new [events](#events) seamlessly
- Implement and test your [commands](#command)
- ...and much more!

All that with first-class developer experience and minimal boilerplate ✨

<!-- > Castore is still under active development. A v1 has been released with a first set of features, but we have big plans for the future (read models, events replay, migrations...)! So 📣 STAY TUNED 📣 -->

## 🫀 Core Design

Some important decisions that we've made early on:

### 💭 **Abstractions first**

Castore has been designed with **flexibility** in mind. It gives you abstractions that are meant to be used **anywhere**: React apps, containers, Lambdas... you name it!

For instance, `EventStore` classes are **stack agnostic**: They need an `EventStorageAdapter` class to interact with actual data. You can code your own `EventStorageAdapter` (simply implement the interface), but it's much simpler to use an off-the-shelf adapter like [`DynamoDBEventStorageAdapter`](./packages/dynamodb-event-storage-adapter/README.md).

### 🙅‍♂️ **We do NOT deploy resources**

While some packages like `DynamoDBEventStorageAdapter` require compatible infrastructure, Castore is not responsible for deploying it.

Though that is not something we exclude in the future, we are a small team and decided to focus on DevX first.

### ⛑ **Full type safety**

Speaking of DevX, we absolutely love TypeScript! If you do too, you're in the right place: We push type-safety to the limit in everything we do!

If you don't, that's fine 👍 Castore is still available in Node/JS. And you can still profit from some nice JSDocs!

### 📖 **Best practices**

The Event Sourcing journey has many hidden pitfalls. We ran into them for you!

Castore is opiniated. It comes with a collection of best practices and documented anti-patterns that we hope will help you out!

## Table of content

- [Getting Started](#getting-started)
  - [📥 Installation](#-installation)
  - [📦 Packages structure](#-packages-structure)
- [The Basics](#the-basics)
  - [📚 Events](#-events)
  - [🏷 Event Types](#-eventtype)
  - [🏗 Aggregates](#-aggregate)
  - [⚙️ Reducers](#%EF%B8%8F-reducer)
  - [🎁 Event Store](#%EF%B8%8F-reducers)
  - [💾 Event Storage Adapter](#-eventstorageadapter)
  - [📨 Command](#-command)
- [Resources](#resources)
  - [🎯 Test Tools](#-test-tools)
  - [🔗 Packages List](#-packages-list)
  - [📖 Common Patterns](#-common-patterns)

## Getting Started

### 📥 Installation

```bash
# npm
npm install @castore/core

# yarn
yarn add @castore/core
```

### 📦 Packages structure

Castore is not a single package, but a **collection of packages** revolving around a `core` package. This is made so every line of code added to your project is _opt-in_, wether you use tree-shaking or not.

Castore packages are **released together**. Though different versions may be compatible, you are **guaranteed** to have working code as long as you use matching versions.

Here is an example of working `package.json`:

```js
{
  ...
  "dependencies": {
    "@castore/core": "1.3.1",
    "@castore/dynamodb-event-storage-adapter": "1.3.1"
    ...
  },
  "devDependencies": {
    "@castore/test-tools": "1.3.1"
    ...
  }
}
```

## The Basics

### 📚 `Events`

Event Sourcing is all about **saving changes in your application state**. Such changes are represented by **events**, and needless to say, they are quite important 🙃

Events that concern the same business entity (like a `User`) are aggregated through a common id called `aggregateId` (and vice versa, events that have the same `aggregateId` represent changes of the same business entity). The index of an event in such a serie of events is called its `version`.

In Castore, stored events (also called **event details**) always have exactly the following attributes:

- <code>aggregateId <i>(string)</i></code>
- <code>version <i>(integer ≥ 1)</i></code>
- <code>timestamp <i>(string)</i></code>: A date in ISO 8601 format
- <code>type <i>(string)</i></code>: A string identifying the business meaning of the event
- <code>payload <i>(?any = never)</i></code>: A payload of any type
- <code>metadata <i>(?any = never)</i></code>: Some metadata of any type

```ts
import type { EventDetail } from '@castore/core';

type UserCreatedEventDetail = EventDetail<
  'USER_CREATED',
  { name: string; age: number },
  { invitedBy?: string }
>;

// 👇 Equivalent to:
type UserCreatedEventDetail = {
  aggregateId: string;
  version: number;
  timestamp: string;
  type: 'USER_CREATED';
  payload: { name: string; age: number };
  metadata: { invitedBy?: string };
};
```

### 🏷 `EventType`

Events are generally classified in **events types** (not to confuse with TS types). Castore lets you declare them via the `EventType` class:

```ts
import { EventType } from '@castore/core';

export const userCreatedEventType = new EventType<
  'USER_CREATED',
  { name: string; age: number },
  { invitedBy?: string }
>({ type: 'USER_CREATED' });
```

Note that we only provide TS types for `payload` and `metadata` properties. That is because, as stated in the [core design](#🫀-core-design), **Castore is meant to be as flexible as possible**, and that includes the validation library you want to use: The `EventType` class is not meant to be used directly, but rather extended by other classes which will add run-time validation methods to it 👍

See the following packages for examples:

- [JSON Schema Event Type](./packages/json-schema-event/README.md)
- [Zod Event Type](./packages/zod-event/README.md)

**Constructor:**

- <code>type <i>(string)</i></code>: The event type

```ts
import { EventType } from '@castore/core';

export const userCreatedEventType = new EventType({ type: 'USER_CREATED' });
```

**Properties:**

- <code>type <i>(string)</i></code>: The event type

```ts
const eventType = userCreatedEventType.type;
// => 'USER_CREATED'
```

**Type Helpers:**

- <code>EventTypeDetail</code>: Returns the event detail TS type of an `EventType`

```ts
import type { EventTypeDetail } from '@castore/core';

type UserCreatedEventTypeDetail = EventTypeDetail<typeof userCreatedEventType>;

// 👇 Equivalent to:
type UserCreatedEventTypeDetail = {
  aggregateId: string;
  version: number;
  timestamp: string;
  type: 'USER_CREATED';
  payload: { name: string; age: number };
  metadata: { invitedBy?: string };
};
```

- <code>EventTypesDetails</code>: Return the events details of a list of `EventType`

```ts
import type { EventTypesDetails } from '@castore/core';

type UserEventTypesDetails = EventTypesDetails<
  [typeof userCreatedEventType, typeof userRemovedEventType]
>;
// => EventTypeDetail<typeof userCreatedEventType>
// | EventTypeDetail<typeof userRemovedEventType>
```

### 🏗 `Aggregate`

Eventhough entities are stored as series of events, we still want to use a **stable interface to represent their states at a point in time** rather than directly using events. In Castore, it is implemented by a TS type called `Aggregate`.

> ☝️ Think of aggregates as _"what the data would look like in CRUD"_

In Castore, aggregates necessarily contain an `aggregateId` and `version` attributes (the `version` of the latest `event`). But for the rest, it's up to you 🤷‍♂️

For instance, we can include a `name`, `age` and `status` properties to our `UserAggregate`:

```ts
import type { Aggregate } from '@castore/core';

// Represents a User at a point in time
interface UserAggregate extends Aggregate {
  name: string;
  age: number;
  status: 'CREATED' | 'REMOVED';
}

// 👇 Equivalent to:
interface UserAggregate {
  aggregateId: string;
  version: number;
  name: string;
  age: number;
  status: 'CREATED' | 'REMOVED';
}
```

### ⚙️ `Reducer`

Aggregates are derived from their events by [reducing them](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/Reduce) through a `reducer` function. It defines **how to update the aggregate when a new event is pushed**:

<!-- TODO: SCHEMA OF EVENTS AGGREGATE -->

```ts
import type { Reducer } from '@castore/core';

export const usersReducer: Reducer<UserAggregate, UserEventsDetails> = (
  userAggregate,
  newEvent,
) => {
  const { version, aggregateId } = newEvent;

  switch (newEvent.type) {
    case 'USER_CREATED': {
      const { name, age } = newEvent.payload;

      // 👇 Return the next version of the aggregate
      return {
        aggregateId,
        version,
        name,
        age,
        status: 'CREATED',
      };
    }
    case 'USER_REMOVED':
      return { ...userAggregate, version, status: 'REMOVED' };
  }
};

const johnDowAggregate: UserAggregate = johnDowEvents.reduce(usersReducer);
```

> ☝️ Note that aggregates are always **computed on the fly**, and NOT stored. Changing them does not require any data migration whatsoever.

### 🎁 `EventStore`

Once you've defined your [event types](#-eventtype) and how to [aggregate](#%EF%B8%8F-reducer) them, you can bundle them together in an `EventStore` class.

Each event store in your application represents a business entity. Think of event stores as _"what tables would be in CRUD"_, except that instead of directly updating data, you just append new events to it!

In Castore, `EventStore` classes are NOT responsible for actually storing data (this will come with [event storage adapters](#-eventstorageadapter)). But rather to provide a boilerplate-free and type-safe interface to perform many actions such as:

- Listing aggregate ids
- Accessing events of an aggregate
- Building an aggregate with the reducer
- Pushing new events etc...

```ts
import { EventStore } from '@castore/core';

const userEventStore = new EventStore({
  eventStoreId: 'USERS',
  eventTypes: [
    userCreatedEventType,
    userRemovedEventType,
    ...
  ],
  reducer: usersReducer,
});
// ...and that's it 🥳
```

> ☝️ The `EventStore` class is the heart of Castore, it even gave it its name!

**Constructor:**

- <code>eventStoreId <i>(string)</i></code>: A string identifying the event store
- <code>eventStoreEvents <i>(EventType[])</i></code>: The list of event types in the event store
- <code>reduce <i>(EventType[])</i></code>: A [reducer function](#⚙️-reducer) that can be applied to the store event types
- <code>storageAdapter <i>(?EventStorageAdapter)</i></code>: See [`EventStorageAdapter`](#💾-eventstorageadapter)

> ☝️ Note that it's the return type of the `reducer` that is used to infer the `Aggregate` type of the EventStore. It is important to type it explicitely.

**Properties:**

- <code>eventStoreId <i>(string)</i></code>

```ts
const userEventStoreId = userEventStore.eventStoreId;
// => 'USERS'
```

- <code>eventStoreEvents <i>(EventType[])</i></code>

```ts
const userEventStoreEvents = userEventStore.eventStoreEvents;
// => [userCreatedEventType, userRemovedEventType...]
```

- <code>reduce <i>((Aggregate, EventType) => Aggregate)</i></code>

```ts
const reducer = userEventStore.reduce;
// => usersReducer
```

- <code>storageAdapter <i>?EventStorageAdapter</i></code>: See [`EventStorageAdapter`](#💾-eventstorageadapter)

```ts
const storageAdapter = userEventStore.storageAdapter;
// => undefined (we did not provide one in this example)
```

> ☝️ The `storageAdapter` is not read-only so you do not have to provide it right away.

**Sync Methods:**

- <code>getStorageAdapter <i>(() => EventStorageAdapter)</i></code>: Returns the event store event storage adapter if it exists. Throws an `UndefinedStorageAdapterError` if it doesn't.

```ts
import { UndefinedStorageAdapterError } from '@castore/core';

expect(() => userEventStore.getStorageAdapter()).toThrow(
  new UndefinedStorageAdapterError({ eventStoreId: 'USERS' }),
);
// => true
```

- <code>buildAggregate <i>((eventDetails: EventDetail[], initialAggregate?: Aggregate) => Aggregate | undefined)</i></code>: Applies the event store reducer to a serie of events.

```ts
const johnDowAggregate = userEventStore.buildAggregate(johnDowEvents);
```

**Async Methods:**

The following methods interact with the data layer of your event store through its [`EventStorageAdapter`](#💾-eventstorageadapter). They will throw an `UndefinedStorageAdapterError` if you did not provide one.

- <code>getEvents <i>((aggregateId: string, opt?: OptionsObj = {}) => Promise\<ResponseObj\>)</i></code>: Retrieves the events of an aggregate, ordered by `version`. Returns an empty array if no event is found for this `aggregateId`.

  `OptionsObj` contains the following attributes:

  - <code>minVersion <i>(?number)</i></code>: To retrieve events above a certain version
  - <code>maxVersion <i>(?number)</i></code>: To retrieve events below a certain version
  - <code>limit <i>(?number)</i></code>: Maximum number of events to retrieve
  - <code>reverse <i>(?boolean = false)</i></code>: To retrieve events in reverse order (does not require to swap `minVersion` and `maxVersion`)

  `ResponseObj` contains the following attributes:

  - <code>events <i>(EventDetail[])</i></code>: The aggregate events (possibly empty)

```ts
const { events: allEvents } = await userEventStore.getEvents(aggregateId);
// => typed as UserEventDetail[] 🙌

// 👇 Retrieve a range of events
const { events: rangedEvents } = await userEventStore.getEvents(aggregateId, {
  minVersion: 2,
  maxVersion: 5,
});

// 👇 Retrieve the last event of the aggregate
const { events: onlyLastEvent } = await userEventStore.getEvents(aggregateId, {
  reverse: true,
  limit: 1,
});
```

- <code>getAggregate <i>((aggregateId: string, opt?: OptionsObj = {}) => Promise\<ResponseObj\>)</i></code>: Retrieves the events of an aggregate and build it.

  `OptionsObj` contains the following attributes:

  - <code>maxVersion <i>(?number)</i></code>: To retrieve aggregate below a certain version

  `ResponseObj` contains the following attributes:

  - <code>aggregate <i>(?Aggregate)</i></code>: The aggregate (possibly `undefined`)
  - <code>events <i>(EventDetail[])</i></code>: The aggregate events (possibly empty)
  - <code>lastEvent <i>(?EventDetail)</i></code>: The last event (possibly `undefined`)

```ts
const { aggregate: johnDow } = await userEventStore.getAggregate(aggregateId);
// => typed as UserAggregate | undefined 🙌

// 👇 Retrieve an aggregate below a certain version
const { aggregate: aggregateBelowVersion } = await userEventStore.getAggregate(
  aggregateId,
  { maxVersion: 5 },
);

// 👇 Returns the events if you need them
const { aggregate, events } = await userEventStore.getAggregate(aggregateId);
```

- <code>getExistingAggregate <i>((aggregateId: string, opt?: OptionsObj = {}) => Promise\<ResponseObj\>)</i></code>: Same as `getAggregate` method, but ensures that the aggregate exists. Throws an `AggregateNotFoundError` if no event is found for this `aggregateId`.

```ts
import { AggregateNotFoundError } from '@castore/core';

expect(async () =>
  userEventStore.getExistingAggregate(unexistingId),
).resolves.toThrow(
  new AggregateNotFoundError({
    eventStoreId: 'USERS',
    aggregateId: unexistingId,
  }),
);
// true

const { aggregate } = await userEventStore.getAggregate(aggregateId);
// => 'aggregate' and 'lastEvent' are always defined 🙌
```

- <code>pushEvent <i>((eventDetail: EventDetail) => Promise\<void\>)</i></code>: Pushes a new event to the event store. Throws an `EventAlreadyExistsError` if an event already exists for the corresponding `aggregateId` and `version`.

```ts
await userEventStore.pushEvent({
  aggregateId,
  version: lastVersion + 1,
  timestamp: new Date().toISOString(),
  type: 'USER_CREATED', // <= event type is correctly typed 🙌
  payload, // <= payload is typed according to the provided event type 🙌
  metadata, // <= same goes for metadata 🙌
});
```

- <code>listAggregateIds <i>((opt?: OptionsObj = {}) => Promise\<ResponseObj\>)</i></code>: Retrieves the list of `aggregateId` of an event store, ordered by `timestamp` of their first event. Returns an empty array if no aggregate is found.

  `OptionsObj` contains the following attributes:

  - <code>limit <i>(?number)</i></code>: Maximum number of aggregate ids to retrieve
  - <code>pageToken <i>(?string)</i></code>: To retrieve a paginated result of aggregate ids

  `ResponseObj` contains the following attributes:

  - <code>aggregateIds <i>(string[])</i></code>: The list of aggregate ids
  - <code>nextPageToken <i>(?string)</i></code>: A token for the next page of aggregate ids if one exists

```ts
const accAggregateIds: string = [];
const { aggregateIds: firstPage, nextPageToken } =
  await userEventStore.getAggregate({ limit: 20 });

accAggregateIds.push(...firstPage);

if (nextPageToken) {
  const { aggregateIds: secondPage } = await userEventStore.getAggregate({
    limit: 20,
    pageToken: nextPageToken,
  });
  accAggregateIds.push(...secondPage);
}
```

**Type Helpers:**

- <code>EventStoreId</code>: Returns the `EventStore` id

```ts
import type { EventStoreId } from '@castore/core';

type UserEventStoreId = EventStoreId<typeof userEventStore>;
// => 'USERS'
```

- <code>EventStoreEventsTypes</code>: Returns the `EventStore` list of events types

```ts
import type { EventStoreEventsTypes } from '@castore/core';

type UserEventsTypes = EventStoreEventsTypes<typeof userEventStore>;
// => [typeof userCreatedEventType, typeof userRemovedEventType...]
```

- <code>EventStoreEventsDetails</code>: Returns the union of all the `EventStore` possible events details

```ts
import type { EventStoreEventsDetails } from '@castore/core';

type UserEventsDetails = EventStoreEventsDetails<typeof userEventStore>;
// => EventTypeDetail<typeof userCreatedEventType>
// | EventTypeDetail<typeof userRemovedEventType>
// | ...
```

- <code>EventStoreReducer</code>: Returns the `EventStore` reducer

```ts
import type { EventStoreReducer } from '@castore/core';

type UserReducer = EventStoreReducer<typeof userEventStore>;
// => Reducer<UserAggregate, UserEventsDetails>
```

- <code>EventStoreAggregate</code>: Returns the `EventStore` aggregate

```ts
import type { EventStoreAggregate } from '@castore/core';

type UserReducer = EventStoreAggregate<typeof userEventStore>;
// => UserAggregate
```

### 💾 `EventStorageAdapter`

_...coming soon_

<!-- You can store your events in many different ways. To specify how to store them (in memory, DynamoDB...) Castore implements Storage Adapters.

Adapters offer an interface between the Event Store class and your storage method 💾.

To be able to use your EventStore, you will need to attach a Storage Adapter 🔗.

All the Storage Adapters have the same interface, and you can create your own if you want to implement new storage methods!

So far, castore supports 2 Storage Adapters ✨:

- in-memory
- DynamoDB -->

### 📨 `Command`

_...coming soon_

## Resources

### 🎯 Test Tools

_...coming soon_

### 🔗 Packages List

_...coming soon_

### 📖 Common Patterns

_...coming soon_
