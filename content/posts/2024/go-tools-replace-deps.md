---
title: "Go Tools - Replacing dependencies in `go.mod`"
date: 2024-11-02
summary: "Wrangling edge cases around the `replace` directive in `go.mod`"
tags:
 - programming
 - go
---

*These tips and tricks are extracted and expanded from my work notes, and are meant to serve more as a reference for me, than as a true standalone blog post with a full narrative structure and whatnot.*

# Replacing dependencies in `go.mod`

 - Official Go docs:
   - Go Modules Reference, [`replace` directive section](https://go.dev/ref/mod#go-mod-file-replace)
   - `go.mod` File format Reference, [`replace` directive](https://go.dev/doc/modules/gomod-ref#replace)

Go projects are built around a top-level `go.mod` (and `go.sum`) file that tracks the master list of all the package dependencies needed to build the project.
Sometimes, you want to *replace* one of those packages with something else, maybe a homegrown alternative library, or a personal fork of some abandoned open source project.

In those cases, the `replace` directive is your friend.

Here's an example where we alter our go.mod to build with my (often very out-of-date!) fork of OPA, on the `v0.69.0` tag that released recently:

    go mod edit -replace github.com/open-policy-agent/opa=github.com/philipaconrad/opa@v0.69.0

We can then run `go mod tidy` to fetch this dependency replacement down into the local Go module cache, and it'll be ready for the next `go build`.

## Replacing weirdly-named tags/branches in `go.mod`

Go's `replace` support is fairly smart, but falls over if your branch or tag name has path separator characters present, like in `philip/feature-branch`, for example.

If you want to use a Git branch with a name like that, try feeding `replace` the commit hash, and then running `go mod tidy` to force the toolchain to mangle your `go.mod` to point at the appropriate git commit.

    # Branch name: philip/explicit-timer-cancel
    # Commit hash at branch tip is: 910f252
    go mod edit -replace github.com/open-policy-agent/opa=github.com/philipaconrad/opa@910f252
    go mod tidy

After the `go mod tidy`, your `go.mod` should point to the tip of the branch.
This is a fantastic trick when you're actively working on a remote fork of a dependency, and want to update your `go.mod` to point to the latest commit on the remote.

## Replace with local filesystem paths

Sometimes, you've got a local copy of a fork, and you want to point your `go.mod` to fetch from that local folder, rather than going out to the internet to find the dependency.

    go mod edit -replace github.com/open-policy-agent/opa=../other-opa-module

Run `go mod tidy`, and you're back in business!