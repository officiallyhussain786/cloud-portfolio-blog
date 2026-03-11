---
title: "Go Concurrency Patterns — Pipeline, Fan Out, Fan In Explained"
description: "Learn how Go's Pipeline, Fan Out, and Fan In concurrency patterns solve the async problems that break Python at scale — with real code from a production document ingestion pipeline."
pubDate: 2026-03-11
category: "Systems"
---

I spent two years writing Python backends. Threading confused me. AsyncIO infected my entire codebase. And when I finally pushed real load — 1000+ documents through a GenAI ingestion pipeline — everything collapsed.

Then I found Go.

Not because Go is faster. Because Go makes concurrency **simple to reason about.** No async/await chains. No GIL. No infection spreading through your codebase. Just goroutines, channels, and a handful of patterns that compose cleanly together.

This post covers the three concurrency patterns I use most — **Pipeline**, **Fan Out**, and **Fan In** — shown together in one real example from my document ingestion pipeline.

---

## The Problem With Python Concurrency

Before Go, here is what concurrency looked like for me in Python:

```python
async def process_document(doc):
    text = await read_pdf(doc)       # had to rewrite
    chunks = await chunk_text(text)  # had to rewrite
    result = await embed(chunks)     # had to rewrite
    await store(result)              # had to rewrite
```

Every function needed `async def`. Every call needed `await`. One library that did not support async — the entire pipeline broke. I spent more time fighting the language than solving the actual problem.

Go's answer is different. Concurrency is not a feature you bolt on — it is built into the language from day one.

---

## Three Patterns, One Pipeline

Let me show you all three patterns working together in a single program. The scenario is a document ingestion pipeline — the same one that broke in Python.

Three stages:

- **Stage 1 — Read:** Load documents one by one into the pipeline
- **Stage 2 — Chunk:** Fan out across 3 workers to process documents concurrently, fan in results
- **Stage 3 — Embed:** Generate embeddings and store results

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

type Document struct {
    Name    string
    Content string
}
```

`Document` is a simple struct that travels through every stage. Think of it as a parcel on a conveyor belt — each stage reads it, does something to it, passes it forward.

---

## Stage 1 — Pipeline: Read Documents

```go
func readDocs(ctx context.Context, docs []string) <-chan Document {
    out := make(chan Document)

    go func() {
        defer close(out)
        for _, doc := range docs {
            select {
            case <-ctx.Done():
                return
            default:
                time.Sleep(10 * time.Millisecond)
                out <- Document{Name: doc, Content: "raw:" + doc}
                fmt.Printf("📄 read:  %s\n", doc)
            }
        }
    }()

    return out
}
```

**Line by line:**

`out := make(chan Document)` — creates the output pipe. Next stage will read from this.

`go func() { ... }()` — runs the reading in a goroutine so main is never blocked.

`defer close(out)` — when all documents are read, seal the pipe. This signals the next stage to stop waiting.

`select { case <-ctx.Done(): return }` — if the context times out or is cancelled, stop cleanly. No work is abandoned mid-flight.

`out <- Document{...}` — sends each document into the pipe.

`return out` — hands the pipe to the next stage. This is how pipeline stages connect.

---

## Stage 2 — Fan Out + Fan In: Chunk Documents

This is where the real concurrency happens. Three workers read from the same input channel simultaneously — that is **Fan Out**. All three send results into one shared output channel — that is **Fan In**.

```go
func chunkWorker(ctx context.Context, in <-chan Document, out chan<- Document, wg *sync.WaitGroup, id int) {
    defer wg.Done()
    for doc := range in {
        select {
        case <-ctx.Done():
            return
        default:
            time.Sleep(30 * time.Millisecond)
            doc.Content = "chunked:" + doc.Content
            fmt.Printf("✂️  worker%d chunked: %s\n", id, doc.Name)
            out <- doc
        }
    }
}

func fanOut(ctx context.Context, in <-chan Document) <-chan Document {
    out := make(chan Document, 10)
    var wg sync.WaitGroup

    for i := 1; i <= 3; i++ {
        wg.Add(1)
        go chunkWorker(ctx, in, out, &wg, i)
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

**Line by line:**

`in <-chan Document` — receive-only channel. Worker can only pull from this, never send into it. Go enforces this at compile time.

`out chan<- Document` — send-only channel. Worker can only push results into this, never read from it.

`for doc := range in` — worker keeps pulling documents from the shared input channel until it is closed and empty.

`defer wg.Done()` — when this worker exits, it tells the WaitGroup it is finished.

`for i := 1; i <= 3; i++ { go chunkWorker(...) }` — this is **Fan Out**. Three workers launched, all reading from the same `in` channel. Go scheduler distributes work naturally — whoever is free picks the next document.

`go func() { wg.Wait(); close(out) }()` — this is **Fan In**. Waits for all three workers to finish, then closes the output channel. Runs in a separate goroutine so it does not block main.

`out := make(chan Document, 10)` — buffered channel with 10 slots. Workers can send results without waiting for the next stage to receive immediately. Smooths out the flow.

---

## Stage 3 — Pipeline: Embed Documents

```go
func embedDocs(ctx context.Context, in <-chan Document) <-chan Document {
    out := make(chan Document)

    go func() {
        defer close(out)
        for doc := range in {
            select {
            case <-ctx.Done():
                return
            default:
                time.Sleep(20 * time.Millisecond)
                doc.Content = "embedded:" + doc.Content
                fmt.Printf("🧠 embed: %s\n", doc.Name)
                out <- doc
            }
        }
    }()

    return out
}
```

Identical shape to Stage 1. Takes input channel, does work, sends to output channel. This is the beauty of the pipeline pattern — every stage follows the same contract. Composable by design.

---

## Main — Wiring Everything Together

```go
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    docs := []string{
        "doc1.pdf", "doc2.pdf", "doc3.pdf",
        "doc4.pdf", "doc5.pdf", "doc6.pdf",
    }

    fmt.Println("🚀 pipeline started")
    fmt.Println("──────────────────────────────")

    start := time.Now()

    stage1 := readDocs(ctx, docs)
    stage2 := fanOut(ctx, stage1)
    stage3 := embedDocs(ctx, stage2)

    count := 0
    for doc := range stage3 {
        fmt.Printf("✅ stored: %s → %s\n", doc.Name, doc.Content)
        count++
    }

    fmt.Println("──────────────────────────────")
    fmt.Printf("📦 total processed: %d docs\n", count)
    fmt.Printf("⏱️  time taken:      %s\n", time.Since(start))
}
```

**Line by line:**

`context.WithTimeout(..., 10*time.Second)` — if the entire pipeline takes more than 10 seconds, cancel everything automatically. Every stage respects this.

`stage1 := readDocs(ctx, docs)` — returns a channel.

`stage2 := fanOut(ctx, stage1)` — takes stage1's channel as input, returns a new channel. Fan Out + Fan In happens here.

`stage3 := embedDocs(ctx, stage2)` — takes stage2's channel as input, returns a new channel.

`for doc := range stage3` — collect results from the final stage. Blocks until stage3's channel is closed and empty — which happens automatically when all documents are processed.

Three lines to wire a concurrent pipeline. That is it.

---

## What the Output Looks Like

```
🚀 pipeline started
──────────────────────────────
📄 read:  doc1.pdf
📄 read:  doc2.pdf
✂️  worker1 chunked: doc1.pdf
✂️  worker2 chunked: doc2.pdf
📄 read:  doc3.pdf
🧠 embed: doc1.pdf
✂️  worker3 chunked: doc3.pdf
✅ stored: doc1.pdf → embedded:chunked:raw:doc1.pdf
...
──────────────────────────────
📦 total processed: 6 docs
⏱️  time taken:      140ms
```

Notice `embedded:chunked:raw:doc1.pdf` — you can see the document travelling through each stage. And notice the overlap — while doc1 is being embedded, doc2 is being chunked, doc3 is being read. All stages running simultaneously.

Sequential equivalent would take roughly 360ms. The pipeline finishes in 140ms.

---

## The Three Patterns Summarised

**Pipeline** — a chain of stages connected by channels. Output of one stage becomes input of the next. All stages run concurrently.

**Fan Out** — one input channel, many workers. Work distributes itself — whoever is free picks the next job. Go scheduler handles distribution automatically.

**Fan In** — many workers, one output channel. Results collect themselves into one stream. WaitGroup and `close()` signal when collection is done.

---

## Why This Beats Python AsyncIO

In Python, making this pipeline concurrent means:

- Rewriting every function as `async def`
- Every call needs `await`
- One non-async library breaks the whole chain
- Cancellation and timeout handling is bolted on awkwardly

In Go:

- Functions are normal functions — no async keyword anywhere
- Concurrency is expressed through goroutines and channels
- Context handles cancellation and timeout across every stage automatically
- One stage failing does not crash the pipeline

The difference is not just performance. It is how much of your mental energy goes toward the actual problem versus fighting the language.

---

## Final Thought

Go was the right tool for this problem. Not because it is trendy — because the concurrency model matches how distributed systems actually work.

If you are a Python backend engineer hitting walls with async at scale — Go is worth a serious look. The patterns in this post are not theoretical. They are what I reach for every time I need concurrent, reliable, production-grade code.

---

*More Go patterns coming. Follow along.*
