# sieve-plugin-sdk

Go bindings for building [Sieve](https://requisitesoft.com) backend plugins.

Backend plugins are WASM modules that register new pipe command keywords (e.g. `| mycommand ...`).
They are compiled with `GOOS=wasip1 GOARCH=wasm` and loaded at runtime by the Sieve server.

## Installation

```bash
go get github.com/requisitesoft/sieve-plugin-sdk
```

## Quick start

```go
//go:build wasip1

package main

import (
    "context"

    "github.com/requisitesoft/sieve-plugin-sdk/plugin"
)

func main() {}

func init() { plugin.RegisterPipePlugin(myPlugin{}) }

type myPlugin struct{}

func (myPlugin) GetInfo(_ context.Context, _ *plugin.InfoRequest) (*plugin.InfoResponse, error) {
    return &plugin.InfoResponse{
        Name:     "my-command",
        Version:  "1.0.0",
        Commands: []string{"mycommand"},
    }, nil
}

func (myPlugin) Execute(ctx context.Context, req *plugin.ExecuteRequest) (*plugin.ExecuteResponse, error) {
    host := plugin.NewPipeHost()
    resp, err := host.ExecuteStats(ctx, &plugin.StatsExecuteRequest{
        StatsExpr:    req.GetArgs(),
        ParquetPaths: req.GetParquetPaths(),
        WhereClause:  req.GetWhereClause(),
        Page:         req.GetPage(),
        PageSize:     req.GetPageSize(),
    })
    if err != nil {
        return &plugin.ExecuteResponse{Error: err.Error()}, nil
    }
    return &plugin.ExecuteResponse{ResultJson: resp.GetRawJson()}, nil
}
```

## Building

```bash
GOOS=wasip1 GOARCH=wasm go build -buildmode=c-shared -o plugin.wasm .
```

Place `plugin.wasm` in `plugins/<your-plugin-name>/backend/` relative to the Sieve server binary.
The server discovers and loads it automatically on startup.

---

## API Reference

### Entry point

#### `RegisterPipePlugin(p PipePlugin)`

Call this in your `init()` function to register your plugin with the host. Must be called exactly once.

---

### `PipePlugin` interface

Your plugin struct must implement this interface.

```go
type PipePlugin interface {
    GetInfo(context.Context, *InfoRequest) (*InfoResponse, error)
    Execute(context.Context, *ExecuteRequest) (*ExecuteResponse, error)
}
```

#### `GetInfo`

Called by the host at startup to discover the plugin's name and the pipe commands it handles.

**Returns** `*InfoResponse`:

| Field | Type | Description |
|---|---|---|
| `Name` | `string` | Human-readable plugin name |
| `Version` | `string` | Plugin version string |
| `Commands` | `[]string` | Pipe keywords this plugin handles (e.g. `["chart"]`) |

#### `Execute`

Called when a query contains one of your registered pipe keywords.

**Receives** `*ExecuteRequest`:

| Getter | Type | Description |
|---|---|---|
| `GetCommand()` | `string` | The matched keyword (useful if you registered multiple) |
| `GetArgs()` | `string` | Everything after the keyword on the same pipe segment |
| `GetRest()` | `string` | Remaining pipe segments after yours (rarely needed) |
| `GetParquetPaths()` | `[]string` | Data files in scope for the current query |
| `GetWhereClause()` | `string` | Lucene filter from the search bar (before the first pipe) |
| `GetStartTime()` | `string` | ISO 8601 start of the selected time range |
| `GetEndTime()` | `string` | ISO 8601 end of the selected time range |
| `GetPage()` | `int32` | Requested page number (1-based) |
| `GetPageSize()` | `int32` | Requested page size |

**Returns** `*ExecuteResponse`:

| Field | Type | Description |
|---|---|---|
| `ResultJson` | `[]byte` | JSON payload passed to the frontend component for rendering |
| `Error` | `string` | Non-empty string aborts execution and shows an error to the user |

---

### `PipeHost` interface

Obtained by calling `NewPipeHost()`. Lets your plugin call back into the Sieve host to run queries.

```go
type PipeHost interface {
    ExecuteStats(context.Context, *StatsExecuteRequest) (*StatsExecuteResponse, error)
    ExecuteSearch(context.Context, *SearchExecuteRequest) (*SearchExecuteResponse, error)
}
```

#### `NewPipeHost() PipeHost`

Returns a handle to the host. Call this inside `Execute` when you need to run a query.

#### `ExecuteStats`

Run a stats aggregation (equivalent to a `| <expr>` query in the search bar).

**Request** `*StatsExecuteRequest`:

| Field | Type | Description |
|---|---|---|
| `StatsExpr` | `string` | Aggregation expression, e.g. `"count() BY host"` or `"sum(bytes)"` |
| `ParquetPaths` | `[]string` | Pass through from `ExecuteRequest.GetParquetPaths()` |
| `WhereClause` | `string` | Pass through from `ExecuteRequest.GetWhereClause()` |
| `Page` | `int32` | Page number (1-based) |
| `PageSize` | `int32` | Rows per page |

**Response** `*StatsExecuteResponse`:

| Getter | Type | Description |
|---|---|---|
| `GetColumns()` | `[]string` | Column names in result order |
| `GetGroupBy()` | `[]string` | Which columns are GROUP BY keys |
| `GetRowsJson()` | `[]string` | Each row serialized as a JSON array, e.g. `["host1", 42]` |
| `GetIsAggregate()` | `bool` | True when the expression produced aggregated results |
| `GetTotalCount()` | `int32` | Total matching rows across all pages |
| `GetError()` | `string` | Non-empty if the host query failed |

#### `ExecuteSearch`

Retrieve raw log entries matching a filter.

**Request** `*SearchExecuteRequest`:

| Field | Type | Description |
|---|---|---|
| `ParquetPaths` | `[]string` | Pass through from `ExecuteRequest.GetParquetPaths()` |
| `WhereClause` | `string` | Pass through from `ExecuteRequest.GetWhereClause()` |
| `Page` | `int32` | Page number (1-based) |
| `PageSize` | `int32` | Rows per page |

**Response** `*SearchExecuteResponse`:

| Getter | Type | Description |
|---|---|---|
| `GetEntriesJson()` | `[]string` | Each log entry serialized as a JSON object |
| `GetTotalCount()` | `int32` | Total matching entries across all pages |
| `GetError()` | `string` | Non-empty if the host query failed |

---

## Versioning

SDK releases follow the same `v*` tags as the Sieve server. Use the SDK version that matches
your target Sieve server version.
