## Substreams Sink

This is a Substreams Sink library. You can use to build any sink application that consumes Substreams in Golang

### Features
What you get by using this library:

- Handles connection and reconnections
- Throughput Logging (block rates, etc)
- Best Practices error handling

### Usage

The library provides a `Sinker` class that can be used to connect to the Substreams API. The `Sinker` class is a wrapper around the `substreams` library, which is a low-level library that provides a convenient way to connect to the Substreams API.

The user's primary responsibility when creating a custom sink is to pass a `BlockScopedDataHandler` and a `BlockUndoSignalHandler` implementation(s) which has the following interface:

```go
impport (
	pbsubstreamsrpc "github.com/streamingfast/substreams/pb/sf/substreams/rpc/v2"
)

type BlockScopedDataHandler = func(ctx context.Context, cursor *Cursor, data *pbsubstreamsrpc.BlockScopedData) error
type BlockUndoSignalHandler = func(ctx context.Context, cursor *Cursor, undoSignal *pbsubstreamsrpc.BlockUndoSignal) error
```

##### `BlockScopedDataHandler`

* `ctx context.Context` is the `sink.Sinker` actual `context.Context`.
* `cursor *Cursor` is the cursor at the given block, this cursor should be saved regularly as a checkpoint in case the process is interrupted.
* `data *pbsubstreamsrpc.BlockScopedData` contains the data that was received from the Substreams API, refer to it's definition for proper usage.

##### `BlockUndoSignalHandler`

* `ctx context.Context` is the `sink.Sinker` actual `context.Context`.
* `cursor *Cursor` is the cursor to use after the undo, this cursor should be saved regularly as a checkpoint in case the process is interrupted.
* `data *pbsubstreamsrpc.BlockUndoSignal` contains the last valid block that is still valid, any data saved after this last saved block should be discarded.

#### Handlers Flow

The basic pattern for using the `Sinker` is as follows:

* Create your data layer which is responsible for decoding the Substreams' data and saving it to the desired storage.
* Have this object implement handling of `*pbsubstreamsrpc.BlockScopedData` message.
* Have this object implement handling of `*pbsubstreamsrpc.BlockUndoSignal` message.
* Create a `Sinker` object using `sink.New` and pass in the two handlers that calls your implementations.

The `BlockScopedDataHandler` is called for each block scoped data message that is received from the Substreams API and contains all the data output for the given Substreams module. In your handler, you are responsible for decoding and processing the data, and returning an error if there is a problem. It's there also that you should persist the cursor according to your persistence logic.

The `BlockUndoSignalHandler` is called when a message `*pbsubstreamsrpc.BlockUndoSignal` is received from the stream. Those message contains a `LastValidBlock` which points to the last block should be assumed to be part of the canonical chain as well as `LastValidCursor` which should be used as the current active cursor.

How is the `*pbsubstreamsrpc.BlockUndoSignal` is actually is implementation dependent. The correct behavior is to treat every piece of data that is contained in a `BlockScopedData` whose `BlockScopedData.Clock.Number` is `> LastValidBlock.Number` as now invalid. For example, if all written entities have a block number, one handling possibility is to delete every entities where `blockNumber > LastValidBlock.Number`.

### Launching

The sinker can be launched by calling the `Start` method on the `Sinker` object. The `Start` method will block until the sinker is stopped.

The sinker implements the [shutter](https://github.com/streamingfast/shutter/blob/develop/shutter.go) interface which can be used to handle all shutdown logic (eg: flushing any remaining data to storage, stopping the sink in case of database disconnection, etc.)

### Example uses

The following repositories are examples of how the sink library can be used:

* [substreams-sink-mongodb](https://github.com/streamingfast/substreams-sink-mongodb)
* [substreams-sink-postgres](https://github.com/streamingfast/substreams-sink-postgres)
* [substreams-sink-files](https://github.com/streamingfast/substreams-sink-files)
