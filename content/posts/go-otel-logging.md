+++
title = 'Go OTel Logging'
date = 2025-02-08T12:42:36+01:00
draft = true
+++

To start with, let's quickly review the conceptual flow of logs from the creation
(we'll be using `slog` package) to the moment they're sent to a collector or a backend.

```goat
+-------------+       +-------------+       +----------+       +-----------+       +----------+
|             |       |             |       |          |       |           |       |          |
| slog logger +------>| slog bridge +------>| provider +------>| processor +------>| exporter |
|             |       |             |       |          |       |           |       |          |
+-------------+       +-------------+       +----+-----+       +-----+-----+       +----------+
```

Thanks to the way `slog` is architected
it was possible to create a 'bridge' that implements `Handler` interface.
Looking at the architecture of the otel instrumentation library itself
one might think it originates from Java world ;).
The documentation does not explain everything in detail, so I'll focus on the gaps.
To get the baseline information visit OTel's page for 
[getting started with Go](https://opentelemetry.io/docs/languages/go/getting-started/).
The code presented there gives some idea of how the parts should work together,
but is missing a few important details that will make our lives a little easier.

### Provider

The provider is responsible for creating new OTel loggers based on the configuration provided
when instantiating the provider itself (feeling that Java vibes already?).
So the below code creates a logger provider configured to use `BatchProcessor` to manipulate
logs before they get passed to the exporter to send to the log collector
(which can be some service, backend or OTel collector).

```go
import "go.opentelemetry.io/otel/sdk/log"

...

	loggerProvider := log.NewLoggerProvider(
		log.WithProcessor(log.NewBatchProcessor(logExporter)),
	)
```

### Processor

In the example above we used the `BatchProcessor` which is the recommended way
of handling the logs in production environment.
It first queues all records in a queue implemented as a ring buffer.
There is a separate, polling goroutine spawned, responsible for batching the records
at regular time intervals.
To prevent that polling goroutine from backing up, yet another, export goroutine
is spawned.
It is responsible for passing the batched records to the exporter.

There's also `SimpleProcessor` available, which might be useful for development.
It simply passes all the logs to exporter synchronously.


