---
title: "Go Tools - Running custom analysis passes with `go vet`"
date: 2024-11-02
summary: "Wrangling edge cases around the `replace` directive in `go.mod`"
tags:
 - programming
 - go
---

*These tips and tricks are extracted and expanded from my work notes, and are meant to serve more as a reference for me, than as a true standalone blog post with a full narrative structure and whatnot.*

Sometimes, you just want to run an analysis pass that isn't available by default in your local `go vet`.
In my case, the `nilness` pass wasn't enabled by default for my local Golang tooling, but *was* enabled in my editor's Go linter tooling, leading to warnings that I couldn't investigate without firing up an IDE.
I did some digging, and realized that I could manually invoke the analysis pass, with some careful CLI hackery:

    go install golang.org/x/tools/go/analysis/passes/nilness/cmd/nilness@latest 
    go vet -vettool $(which nilness) ./...
