# chans: Building blocks for idiomatic Go pipelines

The `chans` package provides generic channel operations to help you build concurrent pipelines in Go. It aims to be flexible, unopinionated, and composable, without over-abstracting or taking control away from the developer.

```go
// Given a channel of documents.
docs := make(chan []string, 10)
docs <- []string{"go", "is", "awesome"}
docs <- []string{"cats", "are", "cute"}
close(docs)

// Extract all words from the documents.
words := make(chan string, 10)
chans.Flatten(ctx, words, docs)
close(words)

// Calculate the total byte count of all words.
step := func(acc int, word string) int { return acc + len(word) }
count := chans.Reduce(ctx, words, 0, step)

fmt.Println("byte count =", count)
// byte count = 22
```

You can find function signatures and usage examples at the links below, or check out the full package [documentation](https://pkg.go.dev/github.com/nalgeon/chans).

## Features

The golden trio:

-   [Filter](https://pkg.go.dev/github.com/nalgeon/chans#Filter) sends values from the input channel to the output if a predicate returns true.
-   [Map](https://pkg.go.dev/github.com/nalgeon/chans#Map) reads values from the input channel, applies a function, and sends the result to the output.
-   [Reduce](https://pkg.go.dev/github.com/nalgeon/chans#Reduce) combines all values from the input channel into one using a function and returns the result.

Filtering and sampling:

-   [FilterOut](https://pkg.go.dev/github.com/nalgeon/chans#FilterOut) ignores values from the input channel if a predicate returns true, otherwise sends them to the output.
-   [Drop](https://pkg.go.dev/github.com/nalgeon/chans#Drop) skips the first N values from the input channel and sends the rest to the output.
-   [DropWhile](https://pkg.go.dev/github.com/nalgeon/chans#DropWhile) skips values from the input channel as long as a predicate returns true, then sends the rest to the output.
-   [Take](https://pkg.go.dev/github.com/nalgeon/chans#Take) sends up to N values from the input channel to the output.
-   [TakeNth](https://pkg.go.dev/github.com/nalgeon/chans#TakeNth) sends every Nth value from the input channel to the output.
-   [TakeWhile](https://pkg.go.dev/github.com/nalgeon/chans#TakeWhile) sends values from the input channel to the output while a predicate returns true.
-   [First](https://pkg.go.dev/github.com/nalgeon/chans#First) returns the first value from the input channel that matches a predicate.

Batching and windowing:

-   [Chunk](https://pkg.go.dev/github.com/nalgeon/chans#Chunk) groups values from the input channel into fixed-size slices and sends them to the output.
-   [ChunkBy](https://pkg.go.dev/github.com/nalgeon/chans#ChunkBy) groups consecutive values from the input channel into slices whenever the key function's result changes.
-   [Flatten](https://pkg.go.dev/github.com/nalgeon/chans#Flatten) reads slices from the input channel and sends their elements to the output in order.

De-duplication:

-   [Compact](https://pkg.go.dev/github.com/nalgeon/chans#Compact) sends values from the input channel to the output, skipping consecutive duplicates.
-   [CompactBy](https://pkg.go.dev/github.com/nalgeon/chans#CompactBy) sends values from the input channel to the output, skipping consecutive duplicates as determined by a custom equality function.
-   [Distinct](https://pkg.go.dev/github.com/nalgeon/chans#Distinct) sends values from the input channel to the output, skipping all duplicates.
-   [DistinctBy](https://pkg.go.dev/github.com/nalgeon/chans#DistinctBy) sends values from the input channel to the output, skipping duplicates as determined by a key function.

Routing:

-   [Broadcast](https://pkg.go.dev/github.com/nalgeon/chans#Broadcast) sends every value from the input channel to all output channels.
-   [Split](https://pkg.go.dev/github.com/nalgeon/chans#Split) sends values from the input channel to output channels in round-robin fashion.
-   [Partition](https://pkg.go.dev/github.com/nalgeon/chans#Partition) sends values from the input channel to one of two outputs based on a predicate.
-   [Merge](https://pkg.go.dev/github.com/nalgeon/chans#Merge) concurrently sends values from multiple input channels to the output, with no guaranteed order.
-   [Concat](https://pkg.go.dev/github.com/nalgeon/chans#Concat) sends values from multiple input channels to the output, processing each input channel in order.
-   [Drain](https://pkg.go.dev/github.com/nalgeon/chans#Drain) consumes and discards all values from the input channel.

## Motivation

I think third-party concurrency packages are often too opinionated and try to hide too much complexity. As a result, they end up being inflexible and don't fit a lot of use cases.

For example, here's how you use the `Map` function from the [rill](https://github.com/destel/rill) package:

```go
// Concurrency = 3
users := rill.Map(ids, 3, func(id int) (*User, error) {
    return db.GetUser(ctx, id)
})
```

The code looks simple, but it makes `Map` pretty opinionated and not very flexible:

-   The function is non-blocking and spawns a goroutine. There is no way to change this.
-   The function doesn't exit early on error. There is no way to change this.
-   The function creates the output channel. There is no way to control its buffering or lifecycle.
-   The function can't be canceled.
-   The function requires the developer to use a custom `Try[T]` type for both input and output channels.
-   The "N workers" logic is baked in, so you can't use a custom concurrent group implementation.

While this approach works for many developers, I personally don't like it. With `chans`, my goal was to offer a fairly low-level set of composable channel operations and let developers decide how to use them.

For comparison, here's how you use the `chans.Map` function:

```go
err := chans.Map(ctx, users, ids, func(id int) (*User, error) {
    return db.GetUser(ctx, id)
})
```

`chans.Map` only implements the core mapping logic:

-   Reads values from the input channel.
-   Calls the mapping function on each value.
-   Writes results to the output channel.
-   Stops if there's an error or if the context is canceled.
-   Does not start any additional goroutines.

You decide the rest:

-   Want Map to be non-blocking? Run it in a goroutine.
-   Don't want to exit early? Gather the errors instead of returning them.
-   Want to buffer the output channel or keep it opened? You have full control.
-   Need to process input in parallel? Use `errgroup.Group`, or `sync.WaitGroup`, or any other implementation.

The same applies to other channel operations.

## License

Created by [Anton Zhiyanov](https://antonz.org/). Released under the MIT License.
