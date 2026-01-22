---
title: "Trivy, Go's init(), and the Magic of Dynamic Analyzer Discovery"
date: 2025-10-07
description: "go, open-source, design-patterns"
tags: ["life"]
---

{{< lead >}}
How Trivy Dynamically Discovers New File Format Analyzers in Go
{{< /lead >}}

## What?
I want to discuss a particular design pattern in go through a [PR](https://github.com/aquasecurity/trivy/pull/8897) which I raised in the open source project [trivy](https://github.com/aquasecurity/trivy)

## What is trivy?
[Trivy](https://trivy.dev) is an open source vulnerability scanner. You could use Trivy to find vulnerabilities (CVE) & misconfigurations (IaC) across code repositories, binary artifacts, container images, Kubernetes clusters, and more.

Since it is open source it used in extensively in build and deploy pipelines across projects. I used trivy in my build pipelines to scan images, configuration, licenses and most importantly - **vulnerabilities** in the source code.

And when I got to know that trivy is written in go, I realized I could make a meaningful open source contribution in the [repository](https://github.com/aquasecurity/trivy). I want to talk about a design pattern used in trivy and how I contributed to add support for another file format in it.

## Issue
So I found this relevant issue while inspecting the repository on Github: [feat(nodejs): Bun support](https://github.com/aquasecurity/trivy/issues/8307). The issue talks about adding support for [bun lockfile](https://bun.sh/blog/bun-lock-text-lockfile).
For those who do not know, [bun](https://bun.sh/) is a new javascript runtime, much like node, but faster. It comes with its own lock file *bun.lock* much like *package.json* as in the case with node projects.

Now bun.lock file has a structure and the issue talked about adding support for scanning this particular file only. You see, trivy already had the support for major file formats related to npm and node, but users wanted support for **bun.lock** file as well.

## Solution
The maintainers helped a lot, gave me pointers and solved my doubts while implementing the solution for this issue. I want to point out a certain nuance that trivy uses to analyze different file formats and how it was so easy to add support for yet another file format to it.

I raised 2 pull requests for this change, first I implemented a prerequisite [parser](https://github.com/aquasecurity/trivy/pull/8851/files) and then an [analyzer for bun.lock](https://github.com/aquasecurity/trivy/pull/8897)
I want you to look at the second PR.

## Go and trivy
You see Trivy scans many file types:
- package-lock.json (npm)
- Pipfile.lock (Python)
- poetry.toml (Python)
And more…

Each format requires a different analyzer. You can look at the file [pkg/fanal/analyzer/language/nodejs/bun/bun.go](https://github.com/aquasecurity/trivy/pull/8897/files#diff-56193d94f68a465ac3eb687e19688481e7411394179e2e2d78fd68211b2b8d5c)
where I wrote an analyzer for `bun.lock` files. The package basically has the following important components:
- init function
- bunLibraryAnalyzer struct
- PostAnalyze method

```go
func init() {
	analyzer.RegisterPostAnalyzer(analyzer.TypeBun, newBunLibraryAnalyzer)
}

type bunLibraryAnalyzer struct {
	logger        *log.Logger
	lockParser    language.Parser // generic type definition for the parser
	packageParser *packagejson.Parser
}

func newBunLibraryAnalyzer(_ analyzer.AnalyzerOptions) (analyzer.PostAnalyzer, error) {
	return &bunLibraryAnalyzer{
		logger:        log.WithPrefix("bun"),
		lockParser:    bun.NewParser(), // specific type definition while constructing
		packageParser: packagejson.NewParser(),
	}, nil
}

func (a bunLibraryAnalyzer) PostAnalyze(_ context.Context, input analyzer.PostAnalysisInput) (*analyzer.AnalysisResult, error) {
	// ... a long implementation of the analyzer specific to bun.lock file
}
```

Such implementations are present for different file formats as well.

### But how does trivy call the right analyzer for the right file?

If Trivy had to manually list every analyzer in a central switch statement, adding a new analyzer would require touching the core code every time.

This violates the Open/Closed Principle: software should be open for extension, closed for modification

This is where Trivy employs the go's `init()` function.

> In go, every package can define one or more init() functions:

```go
package bun

import "fmt"

func init() {
    fmt.Println("Bun analyzer initialized")
}
```

Key points about `init()`:
- Called automatically once per package before main() starts
- Useful for initialization logic that should run without explicit calls
- Can be used to register types, plug-ins, or handlers dynamically

Think of it as Go's "automatic constructor for a package."

### Dynamic discovery via import side effect
Trivy has a file pkg/fanal/analyzer/all/import.go:
```go
import (
    _ "github.com/aquasecurity/trivy/pkg/fanal/analyzer/language/nodejs/npm"
    _ "github.com/aquasecurity/trivy/pkg/fanal/analyzer/language/nodejs/bun"
)
```
If you look at [pkg/fanal/analyzer/all/import.go](https://github.com/aquasecurity/trivy/pull/8897/files#diff-8d2e69ce6143d1b1e019dae4cc7952170076c578519082c130d801eec95a9aea) file, this is where we import ALL the analyzers which are implemented in the trivy code base.

- `_` means import for side effects only
- go will import the package, run its `init()`, and then discard the package name
- The side effect is that the analyzer registers itself in the global registry

So by just adding an import here, your new analyzer is discovered automatically.


## Go init() as a "Plugin Hook"
Imagine every analyzer has a self-registration hook. Go automatically calls these hooks when the program starts. The core logic just says:

"Hey analyzers, tell me which files you can handle."

This is essentially a plugin discovery system powered by `init()`.

### Why This is Cool

- Open/Closed Principle: add new analyzers without touching core code
- Decoupled architecture: core logic doesn't care about specific analyzers
- Plug-and-play: Only requirement is importing the package

## Summary
Trivy's plugin/analyzer system is conceptually closer to virtual functions and inheritance than to the classical Visitor pattern.
In Trivy, each analyzer (like package-lock.json, Pipfile.lock, bun.lock) is a module that implements a common interface, something like:
```go
type Analyzer interface {
    Analyze(ctx context.Context, input analyzer.Input) (*analyzer.Result, error)
    Required(filePath string, info os.FileInfo) bool
}
```

Each analyzer:
- Implements these methods (like overriding virtual functions).
- Calls `analyzer.RegisterAnalyzer()` inside its `init()` to register itself with the global registry.

So the spirit is identical:

> Define a common interface; allow many implementations; dispatch at runtime.

The only difference is how the compiler/runtime achieves the polymorphism:
- Go uses interface type assertions (duck typing).
- C++ uses vtables and virtual calls.
