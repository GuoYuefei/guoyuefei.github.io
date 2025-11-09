---
title: "Golang Cross-Compilation"
date: 2020-05-24
draft: false
tags: ["golang"]
categories: ["Tech"]
description: "Methods for compiling Golang across different architectures and systems"
---

# Golang Cross-Compilation

## Cross-compiling for Other Platforms on Linux and Mac

```
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build main.go
```

```
CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build main.go
```

```
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build main.go
```

## Cross-compiling for Other Platforms on Windows

In cmd:
```cmd
SET CGO_ENABLED=0 
SET GOOS=linux 
SET GOARCH=amd64 
go build main.go
```

## Complete List
After Go 1.13, you can use the toolchain:
```
go env -w GOOS=linux GOARCH=amd64 CGO_ENABLED=0
```

## Summary
GOOS: darwin freebsd linux windows  
GOARCH: 386 amd64 arm  
Cross-compilation doesn't support CGO (on Windows)  
Basically, set temporary environment variables first, then compile