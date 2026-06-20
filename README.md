# sieve-plugin-sdk

Go bindings for building [Sieve](https://requisitesoft.com) backend plugins.

Backend plugins are WASM modules that register new pipe command keywords (e.g. `| mycommand ...`).
They are compiled with `GOOS=wasip1 GOARCH=wasm` and loaded at runtime by the Sieve server.

## Installation

```bash
go get github.com/requisitesoft/sieve-plugin-sdk
```

## How pipelining works

Sieve's pipe syntax chains stages left to right. Each stage is either a built-in stats expression
or a plugin command:

```
<lucene filter> | <stage 1> | <stage 2> | ...
```

The host runs stages in order. Built-in stats expressions (e.g. `count() BY host`) execute a
DuckDB query and pass the result table to the next stage as a typed `PipelineValue`. Plugin stages
receive that value via `req.GetInput()` and can use it directly — no redundant query needed.

### Example: visualization plugin

```
errors | count() BY host | chart pie
```

1. `errors` — Lucene filter passed to all stages as context
2. `count() BY host` — built-in stats stage; produces a `StatsResult` pipeline value
3. `chart pie` — plugin stage; receives the stats table via `req.GetInput()`, renders a chart

The `chart` plugin reads `req.GetInput().GetStats()` instead of running its own query:

```go
func (p myPlugin) Execute(ctx context.Context, req *plugin.ExecuteRequest) (*plugin.ExecuteResponse, error) {
    if input := req.GetInput(); input != nil {
        s := input.GetStats()
        if s == nil {
            return &plugin.ExecuteResponse{Error: "expected a stats result from the previous stage"}, nil
        }
        // s.GetColumns(), s.GetRowsJson(), s.GetGroupBy(), etc.
        return renderChart(s), nil
    }

    // Fallback: plugin is stage 1, no pipeline input — run our own query.
    host := plugin.NewPipeHost()
    resp, err := host.ExecuteStats(ctx, &plugin.StatsExecuteRequest{
        StatsExpr:    req.GetArgs(),
        ParquetPaths: req.GetParquetPaths(),
        WhereClause:  req.GetWhereClause(),
        Page:         1,
        PageSize:     1000,
    })
    if err != nil {
        return &plugin.ExecuteResponse{Error: err.Error()}, nil
    }
    return renderChartFromResponse(resp), nil
}
```

### Non-terminal plugins (passing output to the next stage)

If your plugin is not the last stage, set `Output` instead of `ResultJson` so the host forwards
the value to the next stage:

```go
return &plugin.ExecuteResponse{
    Output: &plugin.PipelineValue{
        Value: &plugin.PipelineValue_Stats{
            Stats: &plugin.StatsResult{
                Columns:  columns,
                RowsJson: rowsJSON,
            },
        },
    },
}, nil
```

A plugin that sets `Output` and is the last stage will have its output rendered by the host as a
stats table. A plugin that sets `ResultJson` and is a non-terminal stage will cause a pipeline
error — the host requires `Output` from all non-terminal plugin stages.

---

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
| `GetParquetPaths()` | `[]string` | Data files in scope for the current query |
| `GetWhereClause()` | `string` | Lucene filter from the search bar (before the first pipe) |
| `GetStartTime()` | `string` | ISO 8601 start of the selected time range |
| `GetEndTime()` | `string` | ISO 8601 end of the selected time range |
| `GetPage()` | `int32` | Requested page number (1-based) |
| `GetPageSize()` | `int32` | Requested page size |
| `GetInput()` | `*PipelineValue` | Output of the previous stage, or `nil` if this is stage 1 |

**Returns** `*ExecuteResponse`:

| Field | Type | Description |
|---|---|---|
| `ResultJson` | `[]byte` | JSON rendered by the frontend component (terminal stage) |
| `Output` | `*PipelineValue` | Typed value passed to the next stage (non-terminal stage) |
| `Error` | `string` | Non-empty aborts the pipeline and shows an error to the user |

Set either `ResultJson` or `Output`, not both. Non-terminal stages must set `Output`.

---

### `PipelineValue`

The typed value that flows between pipe stages.

```go
// Wrap a stats result:
&plugin.PipelineValue{Value: &plugin.PipelineValue_Stats{Stats: statsResult}}

// Wrap a log result:
&plugin.PipelineValue{Value: &plugin.PipelineValue_Logs{Logs: logResult}}
```

**`StatsResult`** fields (via getters):

| Getter | Type | Description |
|---|---|---|
| `GetColumns()` | `[]string` | Column names in result order |
| `GetGroupBy()` | `[]string` | Which columns are GROUP BY keys |
| `GetRowsJson()` | `[]string` | Each row as a JSON array, e.g. `["host1", 42]` |
| `GetIsAggregate()` | `bool` | True when the expression produced aggregated results |
| `GetTotalCount()` | `int32` | Total rows across all pages |

**`LogResult`** fields (via getters):

| Getter | Type | Description |
|---|---|---|
| `GetEntriesJson()` | `[]string` | Each log entry as a JSON object |
| `GetTotalCount()` | `int32` | Total matching entries across all pages |

---

### `PipeHost` interface

Obtained by calling `NewPipeHost()`. Use this when your plugin is stage 1 and needs to run its
own query, or when it needs data from a source other than the pipeline input.

```go
type PipeHost interface {
    ExecuteStats(context.Context, *StatsExecuteRequest) (*StatsExecuteResponse, error)
    ExecuteSearch(context.Context, *SearchExecuteRequest) (*SearchExecuteResponse, error)
}
```

#### `ExecuteStats`

**Request** `*StatsExecuteRequest`:

| Field | Type | Description |
|---|---|---|
| `StatsExpr` | `string` | Aggregation expression, e.g. `"count() BY host"` |
| `ParquetPaths` | `[]string` | Pass through from `req.GetParquetPaths()` |
| `WhereClause` | `string` | Pass through from `req.GetWhereClause()` |
| `Page` | `int32` | Page number (1-based) |
| `PageSize` | `int32` | Rows per page |

**Response** `*StatsExecuteResponse`:

| Getter | Type | Description |
|---|---|---|
| `GetColumns()` | `[]string` | Column names |
| `GetGroupBy()` | `[]string` | GROUP BY column names |
| `GetRowsJson()` | `[]string` | Each row as a JSON array |
| `GetIsAggregate()` | `bool` | True for aggregated results |
| `GetTotalCount()` | `int32` | Total rows across all pages |
| `GetError()` | `string` | Non-empty if the query failed |

#### `ExecuteSearch`

**Request** `*SearchExecuteRequest`:

| Field | Type | Description |
|---|---|---|
| `ParquetPaths` | `[]string` | Pass through from `req.GetParquetPaths()` |
| `WhereClause` | `string` | Pass through from `req.GetWhereClause()` |
| `Page` | `int32` | Page number (1-based) |
| `PageSize` | `int32` | Rows per page |

**Response** `*SearchExecuteResponse`:

| Getter | Type | Description |
|---|---|---|
| `GetEntriesJson()` | `[]string` | Each log entry as a JSON object |
| `GetTotalCount()` | `int32` | Total matching entries |
| `GetError()` | `string` | Non-empty if the query failed |

---

## Versioning

SDK releases follow the same `v*` tags as the Sieve server. Use the SDK version that matches
your target Sieve server version.
