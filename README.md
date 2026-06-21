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

### Plugin output

Plugins always set `Output` with a typed `PipelineValue`. The host passes it to the next stage
when the plugin is intermediate, or renders it as a stats/log table when it is the last stage.
Plugins never need to know their position in the pipeline.

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

If your plugin annotates the result for a specific frontend rendering (e.g. a chart type), set the
relevant field on the typed value:

```go
Stats: &plugin.StatsResult{
    Columns:   columns,
    RowsJson:  rowsJSON,
    ChartType: "pie",   // frontend renders as a pie chart
},
```

If a plugin sets no `Output` and is used as a non-terminal stage, the host returns a clear error
to the user. The plugin author does not need to handle this case.

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

Called by the host at startup to discover the plugin's name, pipe commands, and any
configuration variables it exposes to the user.

**Returns** `*InfoResponse`:

| Field | Type | Description |
|---|---|---|
| `Name` | `string` | Human-readable plugin name |
| `Version` | `string` | Plugin version string |
| `Commands` | `[]string` | Pipe keywords this plugin handles (e.g. `["chart"]`) |
| `ConfigVars` | `[]*ConfigVar` | User-configurable settings (see [Plugin configuration](#plugin-configuration)) |

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
| `GetConfig()` | `map[string]string` | User-supplied config values, keyed by `ConfigVar.Name` |

**Returns** `*ExecuteResponse`:

| Field | Type | Description |
|---|---|---|
| `Output` | `*PipelineValue` | Typed result — passed forward or rendered by the host |
| `Error` | `string` | Non-empty aborts the pipeline and shows an error to the user |

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
| `GetChartType()` | `string` | Optional rendering hint, e.g. `"pie"`, `"bar"`, `"line"` |

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

## Plugin configuration

Plugins can declare configuration variables that the Sieve host exposes to the user through a
settings UI. Values are persisted in `plugins/<name>/config.json` alongside the plugin and are
passed to every `Execute` call via `req.GetConfig()`.

### Declaring config variables

Return `ConfigVars` from `GetInfo`:

```go
func (p myPlugin) GetInfo(_ context.Context, _ *plugin.InfoRequest) (*plugin.InfoResponse, error) {
    return &plugin.InfoResponse{
        Name:     "my-plugin",
        Version:  "1.0.0",
        Commands: []string{"mycommand"},
        ConfigVars: []*plugin.ConfigVar{
            {
                Name:        "api_endpoint",
                Type:        "string",
                Description: "Base URL of the external API",
                DefaultValue: "https://api.example.com",
            },
            {
                Name:        "max_results",
                Type:        "number",
                Description: "Maximum number of results to return",
                DefaultValue: "100",
            },
            {
                Name:        "verbose",
                Type:        "bool",
                Description: "Log detailed output",
                DefaultValue: "false",
            },
            {
                Name:        "api_key",
                Type:        "string",
                Description: "API key for authentication",
                Secret:      true,  // stored encrypted, masked in the UI
            },
        },
    }, nil
}
```

**`ConfigVar` fields:**

| Field | Type | Description |
|---|---|---|
| `Name` | `string` | Key used in `req.GetConfig()` |
| `Type` | `string` | `"string"`, `"number"`, or `"bool"` — controls the input widget in the UI |
| `Description` | `string` | Label shown next to the input |
| `DefaultValue` | `string` | Value used when the user hasn't saved anything |
| `Secret` | `bool` | When `true`, the value is stored encrypted and never shown in the UI |

### Reading config in Execute

```go
func (p myPlugin) Execute(ctx context.Context, req *plugin.ExecuteRequest) (*plugin.ExecuteResponse, error) {
    cfg := req.GetConfig()

    endpoint := cfg["api_endpoint"]
    if endpoint == "" {
        endpoint = "https://api.example.com"  // fall back to default
    }

    apiKey := cfg["api_key"]  // decrypted by the host before delivery
    if apiKey == "" {
        return &plugin.ExecuteResponse{Error: "api_key is required — configure it in Settings → Plugins"}, nil
    }

    maxResults := 100
    if v := cfg["max_results"]; v != "" {
        if n, err := strconv.Atoi(v); err == nil {
            maxResults = n
        }
    }

    // ... use endpoint, apiKey, maxResults
}
```

### Config storage and secrets

Config is stored in `plugins/<name>/config.json` next to the plugin directory. Non-secret values
are stored as plain text and can be copied between machines freely. Secret values are encrypted
with AES-256-GCM using a machine-specific key (`plugins/.config-key`).

Because the encryption key is machine-specific, secret values do not transfer when copying a
config file to another machine — the user must re-enter them. Non-secret values transfer without
any extra steps.

To use a shared key across machines (e.g. in a team deployment), set the `SIEVE_CONFIG_KEY`
environment variable to a hex-encoded 32-byte key:

```bash
export SIEVE_CONFIG_KEY=$(openssl rand -hex 32)
```

---

## Versioning

SDK releases follow the same `v*` tags as the Sieve server. Use the SDK version that matches
your target Sieve server version.
