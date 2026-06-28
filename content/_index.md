---
title: go-output
---

`go-output` is the gomatic ecosystem's **CLI-agnostic result encoder** for Go. It encodes any value to an [`io.Writer`](https://pkg.go.dev/io#Writer) as JSON or YAML through a small format registry, selected by a string-backed [`Format`](https://pkg.go.dev/github.com/gomatic/go-output#Format) type. A single entry point — [`Write`](https://pkg.go.dev/github.com/gomatic/go-output#Write) — takes a writer, a format, and a value, so a daemon, a CLI, or a test can all reuse the same encoding. It defaults to JSON when the format is empty and returns the [`ErrUnsupportedFormat`](https://pkg.go.dev/github.com/gomatic/go-output#ErrUnsupportedFormat) sentinel for any format it cannot produce.

- **Source:** [`gomatic/go-output`](https://github.com/gomatic/go-output)
- **API reference:** [pkg.go.dev/github.com/gomatic/go-output](https://pkg.go.dev/github.com/gomatic/go-output)

## Install

```sh
go get github.com/gomatic/go-output
```

## Why a format registry

CLIs and services repeatedly face the same chore: take some result and render it in whichever serialization the caller asked for. `go-output` owns that chore once. The encoder is keyed by a typed [`Format`](https://pkg.go.dev/github.com/gomatic/go-output#Format) value, not a free-form string sprinkled through `switch` statements, so the supported set lives in one place and an unknown format fails with a matchable sentinel rather than a silently wrong output.

The package is deliberately small — two encoders, one entry point, one error. It depends only on the standard library's [`encoding/json`](https://pkg.go.dev/encoding/json), [`gopkg.in/yaml.v3`](https://pkg.go.dev/gopkg.in/yaml.v3), and the gomatic [`go-error`](https://github.com/gomatic/go-error) sentinel mechanism.

## Usage

### Encode to JSON

```go
package main

import (
	"os"

	output "github.com/gomatic/go-output"
)

func main() {
	data := map[string]any{"hello": "world"}
	if err := output.Write(os.Stdout, output.FormatJSON, data); err != nil {
		panic(err)
	}
}
```

JSON is encoded with two-space indentation and HTML escaping turned off, so characters like `&`, `<`, and `>` appear verbatim instead of as `&`-style escapes.

### Encode to YAML

```go
data := map[string]any{"name": "bucket", "count": 3}
if err := output.Write(os.Stdout, output.FormatYAML, data); err != nil {
	panic(err)
}
```

### Default to JSON

An empty [`Format`](https://pkg.go.dev/github.com/gomatic/go-output#Format) selects JSON, so a CLI can pass an unset `--format` flag straight through:

```go
err := output.Write(os.Stdout, "", data) // empty format → JSON
```

### Handle an unsupported format

`Write` returns [`ErrUnsupportedFormat`](https://pkg.go.dev/github.com/gomatic/go-output#ErrUnsupportedFormat) for any format outside the registry, matched with [`errors.Is`](https://pkg.go.dev/errors#Is). The offending format name is included in the error message:

```go
import "errors"

err := output.Write(os.Stdout, output.Format("xml"), data)
if errors.Is(err, output.ErrUnsupportedFormat) {
	// err.Error() contains "xml"
}
```

## Design

- **One entry point.** [`Write(w io.Writer, f Format, data any) error`](https://pkg.go.dev/github.com/gomatic/go-output#Write) is the whole public surface alongside the [`Format`](https://pkg.go.dev/github.com/gomatic/go-output#Format) type, its [`FormatJSON`](https://pkg.go.dev/github.com/gomatic/go-output#pkg-constants)/[`FormatYAML`](https://pkg.go.dev/github.com/gomatic/go-output#pkg-constants) constants, and the [`ErrUnsupportedFormat`](https://pkg.go.dev/github.com/gomatic/go-output#ErrUnsupportedFormat) sentinel.
- **Format is a `string` newtype.** Encodings are selected by a typed value, and the encoder registry is a private `map[Format]encoder` — adding a format is a one-line registration, and unknown formats can never reach the encoders.
- **CLI-agnostic.** `Write` takes only a writer, a format, and a value; it knows nothing about flags, commands, or `os.Stdout`. The caller injects the writer, which keeps it trivially testable against a [`bytes.Buffer`](https://pkg.go.dev/bytes#Buffer).
- **Sentinel error, not a string.** [`ErrUnsupportedFormat`](https://pkg.go.dev/github.com/gomatic/go-output#ErrUnsupportedFormat) is declared as a [`go-error`](https://github.com/gomatic/go-error) `errs.Const`, so callers match its identity with [`errors.Is`](https://pkg.go.dev/errors#Is) and the offending format is carried as context rather than baked into the matched string.

## Who uses it

`go-output` is the shared encoder for gomatic Go CLIs and services — any program that needs to render a result as JSON or YAML from a single `--format` switch. It pairs with [`go-error`](https://github.com/gomatic/go-error) for its sentinel and follows the [`gomatic/go-*`](https://github.com/orgs/gomatic/repositories?q=go-) library conventions.
