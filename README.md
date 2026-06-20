# sieve-plugin-sdk

Go bindings for building [Sieve](https://requisitesoft.com) backend plugins.

Backend plugins are WASM modules that register new pipe command keywords (e.g. `| mycommand ...`).
They are compiled with `GOOS=wasip1 GOARCH=wasm` and loaded at runtime by the Sieve server.

## Installation

```bash
go get github.com/requisitesoft/sieve-plugin-sdk
```

## Usage

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
    // req.GetArgs()         — everything after the command keyword in the query
    // req.GetParquetPaths() — data files in scope
    // req.GetWhereClause()  — Lucene filter from the search bar
    // req.GetPage() / req.GetPageSize() — pagination

    // Call back into the host to run a stats query:
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

## Versioning

SDK releases follow the same `v*` tags as the Sieve server. Use the SDK version that matches
your target Sieve server version.
