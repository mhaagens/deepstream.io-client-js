
## [5.0.1] - 2019.10.29

Added a missing transition state on resubscription.

## [5.0.0] - 2019.10.27

### Breaking Change

Importing and creating the client has changed in order to be less insane when using typescript.

When creating the client you now do:

```typescript
const { DeepstreamClient } = require('@deepstream/client')
const dsClient = new DeepstreamClient(url, options)
```

instead of:

```javascript
const deepstream = require('@deepstream/client')
const dsClient = deepstream(url, options)
```

## [4.1.3] - 2019.09.09

### Fix

Another socketOptions defaulting issue

## [4.1.2] - 2019.09.09

### Fix

Defaulting socketOptions in socket wrapper factory.

# [4.1.1] - 2019-09-08

### Feature

Adding a `jsonTransportMode` flag that can be passed to `socketOptions` that allows the client
to talk to the server in JSON. This is mainly done to help people to debug writing new SDKs and
should not be used in production.

# [4.1.0] - 2019-09-05

### Improvement

Including a default implementation of timeouts using the native setTimeout API. The interval based one was created in order to mitigate the insane amount
of ack registries we used to have get created, and also because setting timeouts
isn't cheap (you can verify this by creating a couple thousand records and noticing the cost within the default noop storage registry). However as correctly stated by @Krishna [here](https://github.com/deepstreamIO/deepstream.io-client-js/issues/500) the interval implementation is naive in terms of mobile (looping every 20milliseconds for idle connections is overkill) and the benefits now is no longer as apparent as during the RC release (since bulk messaging now only makes one timeout for N amount amount of subscriptions).

You can toggle them with the following:

```
# When true uses setTimeout
nativeTimerRegistry: true,
# When nativeTimerRegistry is false uses an interval with this timer resolution
intervalTimerResolution: 50,
```

Either ways both implementations are expensive in terms of garbage collection since it binds to the context and data as part of the API. This can probably be avoided by providing a null context going forward.

### Fix 

Lists were not propagating the events on record-core. Discard, delete and error is now properly passed, and documentation needs to be updated to indicate the ready event was dropped in V4

# [4.0.5] - 2019-08-30

### Improvement

The initial version change seems to have broken some things, so it is now configurable and defaults to 1

# [4.0.4] - 2019-08-19

### Feature

If record is in readonly mode, only subscribe and read, don't try to upsert / create

# [4.0.3] - 2019-08-09

### Feature

Adding notify on record handler to notify if the db was changed without using deepstream APIs

### Fix

Change error message on connection to pass through error 

### Breaking Change

This shouldn't be a breaking change for those with offline disabled, which noone is using yet
hence the patch release. This just changes the initial version 1 to 0

### Fix:

# [4.0.2] - 2019-08-03

### Fix:

- Adding a readOnlyFlag

This is an issue where when not doing offline first requesting a record that doesn't exist would result in a new empty record being created (locally) with no data. This would break bindings for some people. Instead what we do now is use a recordReadOnlyMode flag which indicates that we expected most records to be only ever read from server, and hence will only be ready on data load. You can explicitly mention the things you do want to write to via a recordPrefixWriteWhitelist.

- Also added a transition from unsubscribing to deleted, because so many transitions.

# [4.0.1] - 2019-08-03

### Fix:

- Do a deep compare when setting data to avoid processing unchanged values

# [4.0.0] - 2019-07-30

### Features:

- New binary protocol support (under the hood)
- Bulk actions support (under the hood)
- Full typescript declaration files
- Promises everywhere! Long live async/await!
- Offline record support

```JavaScript
{
    // Use indexdb to store data client side
    offlineEnabled: false,
    // Save each update as it comes in from the server
    saveUpdatesOffline: false,
    indexdb: {
        // The db version, incrementing this triggers a db upgrade
        dbVersion: 1,
        // This auto updates the indexdb version if the objectStore names change
        autoVersion: false,
        // The key to index records by
        primaryKey: 'id',
        // The indexdb databae name
        storageDatabaseName: 'deepstream',
        // The default store name if not using a '/' to indicate the object store (example person/uuid)
        defaultObjectStoreName: 'records',
        // The object store names, required in advance due to how indexdb works
        objectStoreNames: [],
        // Things to not save, such search results
        ignorePrefixes: [],
        // The amount of time to buffer together actions before making a request
        flushTimeout: 50
    }
}
```

- Customizable offline storage support

```typescript
export type offlineStoreWriteResponse = ((error: string | null, recordName: string) => void)

export interface RecordOfflineStore {
  get: (recordName: string, callback: ((recordName: string, version: number, data: RecordData) => void)) => void
  set: (recordName: string, version: number, data: RecordData, callback: offlineStoreWriteResponse) => void
  delete: (recordName: string, callback: offlineStoreWriteResponse) => void
}
```

### Improvements

- Separation of errors and warnings for clarity. Non critical failures (such as an ack timeout) can now be treated separated or fully muted.
- Enhanced services to reduce timeout overhead

### Backwards compatibility

- Only works with V4 server
- All single response APIs now return promises when not providing a callback. This means most APIs that could have been chained would now break.

```JavaScript
const client = deepstream()
try {
    await client.login()

    const record = client.record.getRecord(name)
    await record.whenReady()

    const data = await client.record.snapshot(name)
    const version = await client.record.head(name)
    const exists = await client.record.has(name)
    const result = await client.rpc.make(name, data)
    const users = await client.presence.getAll()
} catch (e) {
    console.log('Error occurred', e)
}
```

- Listening

The listening API has been ever so slightly tweaked in order to simplify removing an active subscription.

Before when an active provider was started you would usually need to store it in a higher scope, for example:

```typescript
const listeners = new Map()

client.record.listen('users/.*', (name, isSubscribed, ({ accept, reject }) => {
    if (isSubscribed) {
        const updateInterval = setInterval(updateRecord.bind(this, name), 1000)
        listeners.set(name, updateInterval)
        accept()
    } else {
        clearTimeout(listeners.get(name))
        listeners.delete(name)
    }
})
```

Where now we instead do:

```typescript
const listeners = new Map()

client.record.listen('users/.*', (name, ({ accept, reject, onStop }) => {
    const updateInterval = setInterval(updateRecord.bind(this, name), 1000)
    accept()

    onStop(() => clearTimeout(updateInterval))
})
```

### TLDR;

You can see the in depth side explanation of the changes [here](https://deepstream.io/releases/client-js/v4-0-0/)

## [2.3.0] - 2017-09-25

### Features

- the presence feature can now be used on a per user basis. The online status of individual users can be queried for as well as subscribed to. Check out the tutorial on our website [here](https://deepstreamhub.com/tutorials/guides/presence/)

### Improvements

- error messages are now stringified to better display information [#386](https://github.com/deepstreamIO/deepstream.io-client-js/pull/386) courtesy of [@SejH](@SejH)
- improved handling of parameters in calls to `client.record.setData`

### Miscellaneous

- moved e2e steps into [deepstream.io](https://github.com/deepstreamIO/deepstream.io) repository

## [2.2.1] - 2017-06-02

### Fixes
- Update `dist/` files correctly

## [2.2.0] - 2017-05-29

### Features
- Clients are now able to perform and upsert operation `CU` (create and update) via `RecordHandler.setData`. This allows writing to records while not subscribed to them locally

## Fixes
- Heartbeat timeout now emits the reconnect event

## [2.1.5] - 2017-05-03

### Bug Fixes
- Calling login() with a callback (but no auth data) now behaves as you would expect.
- Fix issue where client did not emit `MAX_RECONNECTION_ATTEMPTS_REACHED` event by [@rbarroetavena](@rbarroetavena).

## [2.1.4] - 2017-04-11

### Enhancements
- Tightened up typescript callback interfaces by [@EnigmaCurry](@EnigmaCurry)

### Small Fixes
- Using main file as `dist/deepstream.js` over `src/client.js`

## [2.1.3] - 2017-04-08

### Enhancements
- Write acks now called with a failure message if connection is down by [@Erik Karlsson](@Erik Karlsson)
- Write acks now called with null if value hasn't changed by [@Erik Karlsson](@Erik Karlsson)
- Linting / Babel support
- Improved a few typescript bindings by [@EnigmaCurry](@EnigmaCurry)
- Changed heartbeat missed message to include time
- Setting anonymous record with same name no longer discards and resubscribes the record

### Bug Fixes

- Invalid remote wins merge conflicts
- Prevent records from being set with scalar values by [@datasage](@datasage)
- Prevent bad login message from constantly attempting to reconnect by [@datasage](@datasage)
- RecordHandler invalid destroy state emitted an error instead of using client._$onError

## [2.1.2] - 2017-02-28

### Enhancements

- heartbeat missed should close connection [#324](https://github.com/deepstreamIO/deepstream.io-client-js/pull/324)
- optimized json-path patch [#329](https://github.com/deepstreamIO/deepstream.io-client-js/pull/329)
- TypeScript typings [#283](https://github.com/deepstreamIO/deepstream.io-client-js/pull/283) and [#338](https://github.com/deepstreamIO/deepstream.io-client-js/pull/338)
- Added support for non-NaNish base 16 numbers in jsonpath [#328](https://github.com/deepstreamIO/deepstream.io-client-js/pull/328)
- There is now a single ack timeout registry for the client, shared between all handlers. This means that ack timeouts are cleared when the connection is lost and don't occur when the connection is not open [#342](https://github.com/deepstreamIO/deepstream.io-client-js/pull/342)

### Bug Fixes

- reset queued methods in List once called [#315](https://github.com/deepstreamIO/deepstream.io-client-js/pull/315)
- fix queued methods by passing index to .bind() [#316](https://github.com/deepstreamIO/deepstream.io-client-js/pull/316)

## [2.1.1] - 2016-12-21

### Bug Fixes

- Fixed the generated dist release files

## [2.1.0] - 2016-12-20

### Features

- Record write acknowledgement. Records are now able to be set with an optional callback which will be called with any errors from storing the record in cache/storage [#290](https://github.com/deepstreamIO/deepstream.io-client-js/pull/290)

### Enhancements

- Additional tests around presence and records [#284](https://github.com/deepstreamIO/deepstream.io-client-js/pull/284) and [#285](https://github.com/deepstreamIO/deepstream.io-client-js/pull/285)
- Allow passing of node socket options into constructor [#289](https://github.com/deepstreamIO/deepstream.io-client-js/issues/289)

### Bug Fixes

- Fix bug in JSON path when updating nested null values [#281](https://github.com/deepstreamIO/deepstream.io-client-js/issues/281)
- Adding check for undefined entries in single notifier [#291](https://github.com/deepstreamIO/deepstream.io-client-js/pull/291)

## [2.0.0] - 2016-11-18

### Features
- Added support for the deepstream `presence` API, enabling querying and
  subscribing to who is online within a cluster. For example:
  ``` javascript
  ds.presence.getAll((users) => {
    users.forEach((username) => {
      console.log(`${username} is online`)
    })
  })
  ds.presence.subscribe((username, loggedIn) => {
    if (loggedIn) {
      console.log(`${username} has logged in`)
    } else {
      console.log(`${username} has logged out`)
    }
  })
  ```
### Enhancements

- Added heartbeats over WebSocket connection
- Presence has been added to query and subscribe to who is online with the cluster
- E2E tests refactored

### Breaking Changes

- Supports deepstream.io v2.0.0+ only
- Changed format of RPC request ACK messages to be more consistent with the rest of the specs
[#408](https://github.com/deepstreamIO/deepstream.io/issues/408)
- We now depend only on browser/node.js WebSockets, removing support for TCP and engine.io
- Support for webRTC has been removed

## [1.1.1] - 2016-09-30

### Bug Fixes

- Connection errors now occur on the CONNECTION topic
- Message denied clears down associated ACK timeout messages

### Enhancements

- Optimize and refactor records by [@ronag](@ronag)

### Misc

- Porting over remaining e2e tests
- Adding specs as part of the client project and build

## [1.1.0] - 2016-09-08

### Bug Fixes

- fix leaking internal reference to the record data  [#202](https://github.com/deepstreamIO/deepstream.io-client-js/issues/202) by [@ronag](@ronag)

### Enhancements

###### `listen(pattern, isSubscribed, response)`
add a third argument for the listen callback (`client.record.listen` and `client.event.listen`) which contains
an object which two functions (`accept` and `reject`). One of these functions needs to be called otherwise you
will get a deprecated message. [#203](https://github.com/deepstreamIO/deepstream.io-client-js/issues/203) [#212](https://github.com/deepstreamIO/deepstream.io-client-js/issues/212)

This enhancements fixes some issues like [#74](https://github.com/deepstreamIO/deepstream.io-client-js/issues/74) [#155](https://github.com/deepstreamIO/deepstream.io-client-js/issues/155) [#170](https://github.com/deepstreamIO/deepstream.io-client-js/issues/170)

###### provider flag and event
Records supports now a boolean flag (`record.hasProvider`) which indicates whether a listener has accepted providing data. You can also subscribe to event which is triggered when the flag changes:

```javascript
record.on('hasProviderChanged', hasProvider => {
  /* do something */
})
```

###### API checks are now in place that throw an error if you provide the incorrect argument amount or types [#207](https://github.com/deepstreamIO/deepstream.io-client-js/pull/227) by [@ronag](@ronag)

### Misc

###### Gherkin tests are now used for E2E testing, allowing e2e tests to be run against any language rather than just node, and allows writing more scenarios much easier

## [1.0.2] - 2016-07-28

### Bug Fixes

- allow do create and discard the same record in a synchronous loop [#167](https://github.com/deepstreamIO/deepstream.io-client-js/issues/167)
- record snapshots are not waiting for `isReady` [#140](https://github.com/deepstreamIO/deepstream.io-client-js/issues/140)
- record subscriptions are not waiting for `isReady` in combination with `triggerNow` [#138](https://github.com/deepstreamIO/deepstream.io-client-js/issues/138)
- broken unsubscribe due to wrong internal argument delegation [#190](https://github.com/deepstreamIO/deepstream.io-client-js/issues/190)


### Enhancements

- provide a `MAX_RECONNECTION_ATTEMPTS_REACHED` event [#175](https://github.com/deepstreamIO/deepstream.io-client-js/issues/175)
- provide a `maxReconnectInterval` option [#176](https://github.com/deepstreamIO/deepstream.io-client-js/issues/176)

###  Features

- Terminate unauthenticated connections after a timeout [#226](https://github.com/deepstreamIO/deepstream.io/issues/226)

## [1.0.1] - 2016-07-19

### Bug Fixes

- Fixed issue where deleted record was not getting removed

## [1.0.0] - 2016-07-10

### Features

###### Merge Conflict Resolution #4
Users can now set a global and per record merge strategy. This allows the application to respond to `VERSION_EXISTS` and use a `REMOTE_WINS`, `LOCAL_WINS` or a custom merge strategy

Global:
```javascript
const client = deepstream( 'localhost:6020', {
    mergeStrategy: deepstream.MERGE_STRATEGIES.REMOTE_WINS
});
```

Local:
```javascript
const record = client.record.getRecord( 'user/1' )
record.setMergeStrategy( ( record, remoteValue, remoteVersion, callback ) => {
  callback( null, remoteValue )
} )
```


#### Enhancements

###### Connection Redirects
deepstream protocol now has a connection establishment handshake that allows the client to be redirected to another deepstream before requesting/publishing data

###### Record deletes #111
Users can now delete content within records by setting it to undefined

```javascript
record.set( path, undefined )
```

###### Reduced client size #75
Client size bundle has been reduced by not including mocks for the tcp connection

###### Discard/Delete only on ready #94
Record discards and deletes now get called after when ready, which makes the API cleaner

Before:
```javascript
record = client.record.getRecord( 'user1' )
record.set( 'name', 'bob' )
record.onReady( () => {
  record.discard()
})
```

Now:
```javascript
record = client.record.getRecord( 'user1' )
record
  .set( 'name', 'bob' )
  .discard()
```

###### Exposed Constants
You can now access constants on deepstream

```javascript
// on the constructor
const C = deepstream.CONSTANTS;
// and instance
const client = deepstream( 'localhost:6020' )
CONST C = client.CONSTANTS;
```

### Breaking Changes

###### Login callback

The login callback now only takes two arguments instead of three. This is to make it easier for the user to send their own custom data from an authentication hander or when using the http authentication handler

```javascript
client.login( {}, ( success, data ) => {
  if( success ) {
    // data is meta data associated with user session
    // or null
  } else {
    // data is error message or custom error object
    // with reason why or null
  }
} )
```

###### EngineIO Default Path
We now use `deepstream` instead of `engine.io` as the default engineio path

### Bug Fixes

- Login after logout doesn't overide auth parameters #88

- Deepstream not updating object properties #96 ( @drsirmrpresidentfathercharles )
