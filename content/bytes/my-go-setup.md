---
date: '2025-01-03T21:43:56+03:00'
draft: false
title: My Go setup
tags: 
    - code
    - go
    - testing
    - docker
---

In this post I'll describe what I use for writing, linting, formatting, shipping, ... **Go code**. This is the first in a series of posts where I'll describe my setup in the three languages that I code in: Go (this post), Python and C++. 

This one will be the most boring one in the series - Go is the most "batteries included" language out of the bunch, so there's not much to say beyond "I use the default thing that Go/standard library provides".

I code in VSCode, so my setup is geared towards it.

## Environment managament

I'm lucky I don't have to think about setting up an "environment" for Go at all - I just [install](https://go.dev/doc/install) the latest Go binary, navigate to the directory where I want to set up a project, and do

```
go mod init <project_name>
```

That's it. The only potential downside is that this way I am bound to the latest Go version across all my projects - but given Go's strong dedication to [backward compatibility](https://go.dev/blog/compat), this is not an issue. 

This "guarantee" also contributes to the established practice in Go community of only supporting the two most recent minor versions of the language - as it is always safe to upgrade, there is no reason anyone would need support for an older version of the language.

## Package management

Here I can also relax and just use Go's default package manager ([modules](https://go.dev/blog/using-go-modules)). I don't think anything else exists.

Updating all packages is easy:

```
go get -u
```

I like to follow that up with `go mod tidy` to clean up `go.sum`.

## IDE integration (LSP, formatting)

Language server features of Go are provided by `gopls` (ships with Go binary), and managed by the official Go VSCode extension. Everything works as it should - you can see object types, function signatures when hovering over code.

`go fmt` is used for formatting code—although I mostly call it with a keyboard shortcut in the IDE for the file I’m working on.

## Debugging

I haven't yet needed to use a debugger in Go - if I ever will, I will use [delve](https://github.com/go-delve/delve).

## Linting

I use [golangci-lint](https://github.com/golangci/golangci-lint) - it's a lint runner that runs a bunch of different linters, all specified in a single config file. Due to a large number of linters run, it takes a while to run (~20s). In VSCode I use it with the [`--fast`](https://golangci-lint.run/welcome/integrations/#go-for-visual-studio-code) flag, I only run the full version before committing and in CI.

## Testing

Go standard library [testing](https://pkg.go.dev/testing) package is great! Amazing support for all kinds of tests and benchmarking. Although to be able to fully exploit its capabilities you need to [learn a few tricks](https://www.youtube.com/watch?v=8hQG7QlcLBk). For example, using `t.Cleanup` to clean up resources after a test. 

If doing tests on a database, you might want to isolate tests from one another either by injecting a transaction that gets rolled back at the end of the test, or by re-creating database from a template for each test. Describing these techniques in details would require it own post - here's a nice [repo with examples](https://github.com/xorcare/testing-go-code-with-postgres) and [a video](https://www.youtube.com/watch?v=NMYBSyvoUzE) (in Russian) that go into much more detail.

Oh and, I use the [testify](https://github.com/stretchr/testify) library for `assert` statements. I really think it should be part of the standard library.

I used to heavily rely on `TestSuite` from this library as well (I came to Go from Python, and using a test suite was as close to writing tests a-la `pytest` as I could get), but have since torn it out and replaced it with [`TestMain`](https://pkg.go.dev/testing#hdr-Main) and a few judiciously placed global variables.

Here's an example test file that uses some of the things mentioned above:

```go
package db_test

import (
    "os"
    "testing"

    "github.com/stretchr/testify/assert"
    "myrepo/fakedb"
)

// Global DB connection.
var dbConn *fakedb.Connection

// setupUsersTable is a helper function that creates the 'users' table
// and schedules it to be dropped after the test completes.
func setupUsersTable(t *testing.T) {
    t.Helper()
    err := dbConn.Exec("CREATE TABLE users (id INT, name TEXT)")
    if err != nil {
        t.Fatalf("failed to create table: %v", err)
    }

    t.Cleanup(func() {
        _ = dbConn.Exec("DROP TABLE users")
    })
}

func TestMain(m *testing.M) {
    // Initialize and open the database connection.
    dbConn = fakedb.NewConnection("fakedb://localhost:5432")
    if err := dbConn.Open(); err != nil {
        os.Exit(1) // If we can't open the DB, exit immediately.
    }

    // Run all tests.
    m.Run()

    // Close the connection, clean up resources.
    dbConn.Close()
}

func TestInsertUser(t *testing.T) {
    setupUsersTable(t)

    err := dbConn.Insert("users", map[string]interface{}{
        "id":   1,
        "name": "Alice",
    })
    assert.NoError(t, err, "Insert should succeed")

    // Query the user row.
    rows, err := dbConn.Query("SELECT name FROM users WHERE id = 1")
    assert.NoError(t, err, "Query should succeed")
    
    // Check that exactly one row is returned, and the data is correct.
    assert.Len(t, rows, 1)
    assert.Equal(t, "Alice", rows[0]["name"])
}
```

## Shipping

Building code is as easy as runing

```
go build ./...
```

I ship go binaries in docker containers. Since the binary is pretty much the only thing required for the application to run, I use the minimal [distroless](https://github.com/GoogleContainerTools/distroless) images to ship the binary.

Another thing when shipping is that the binary can be used on a different architecture than it was built on (for example, you build it on an `amd64` PC, but will run it on a `arm64` server). There are two things you can do here:
* use emulation (e.g. QEMU) to build the image
* use cross-compilation

Emulation usually results in really slow build times, so I avoid it if I can. Luckily, Go has pretty great support for cross-compilation out of the box - you just need to set `GOARCH` flag when compiling.

Here's how a minimal `Dockerfile` for building and shipping a Go binary might look like

```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.23-alpine AS build

ARG TARGETPLATFORM

COPY go.sum go.mod main.go ./

RUN CGO_ENABLED=0 GOARCH=$(echo $TARGETPLATFORM | cut -d'/' -f2) GOOS=linux go build -o server .

FROM gcr.io/distroless/static-debian11 AS base

USER nonroot

COPY --from=build --chown=nonroot:nonroot /app/server /server

ENTRYPOINT ["/server"]
```

Docker requires you to specify the target platform as `linux/arm64` or `linux/amd64`, while Go omits the `linux/` prefix, which is why the use of `cut` is needed.