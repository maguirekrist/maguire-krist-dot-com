---
layout: ../../layouts/MarkdownPostLayout.astro
title: "Filling the Gap: Building a High-Performance, Mutable AcroForm PDF Processor in .NET"
description: "I built a C#/.NET PDF processor with performance in the same band as PDFPig - and added what I couldn’t find for free in .NET: **true in-memory, mutable AcroForms** you can edit and save. It powers my upcoming forms-over-data SaaS, **SubmitPath** (coming soon at **submitpath.com**)."
author: 'Maguire Krist'
pubDate: 2025-08-21
tags: ["blog", "pdf"]
---


When every free .NET PDF library said *read-only* or *non-commercial only* for forms, I stopped hunting and built my own. The result: a lean, production-minded parser with a practical API, competitive throughput, and the one feature I needed most — **edit AcroForm fields in memory and write the file back**.

---

## Why I built it

* **Real need:** For **SubmitPath**, I needed reliable **read + mutate** of AcroForm fields in .NET without license traps.
* **Design goals:**

  * Performance close to **PDFPig** on real documents
  * Clean API focused on forms work (no acrobatics just to set a field)
  * Minimal dependencies and debuggable code paths
* **Scope:** Implement the PDF 1.7 pieces required for high-volume form handling, then round out the essentials for general parsing/inspection.

> I referenced PDFPig’s **API and capabilities** while writing my own parser (no code copied).

---

## What it does (today)

* **AcroForms you can actually change**

  * Load a PDF, **edit fields in memory**, save back to disk/stream
  * Preserve structure; avoid surprise reflows
* **Solid performance on common gov/enterprise PDFs**

  * Competitive with PDFPig across a variety of real forms
* **Practical API surface**

  * Document catalog, pages, objects, streams
  * Read text + metadata
  * Allocation-aware hot paths

### One-minute API taste: edit a form and save

```csharp
var pdfData = File.ReadAllBytes("g-1145.pdf");
var parser = new PdfParser(pdfData);
var document = parser.Parse();
var form = document.GetAcroForm();
var fields = document.GetFields();

// Read
var field = fields[0];
var value = field.GetValueAsString();


// Mutate
field.SetValue("maguirekrist.com");

// Save
document.Save("g-1145.filled.pdf");
```

---

## Benchmarks (quick look)

> Hardware/runtime/build flags affect absolute values; the target is **same-order performance** as PDFPig on real forms.

**Aggregate across 20 PDFs**

| Metric           |        Value |
| ---------------- | -----------: |
| Documents        |           20 |
| p50 elapsed      |  **9.46 ms** |
| p95 elapsed      | **68.15 ms** |
| Peak working set |  **66.2 MB** |

**Selected files — elapsed (ms), working set (MB)**

| File               |   Size | Ours (ms) | PDFPig (ms) | Working Set (MB) | Notes              |
| ------------------ | -----: | --------: | ----------: | ---------------: | ------------------ |
| `i-9.pdf`          | 524 KB |     46.37 |       46.47 |             \~58 | parity             |
| `GSA_1241.pdf`     | 574 KB |     13.97 |       15.43 |             \~61 | ours faster        |
| `GSA_1535.pdf`     | 370 KB |     21.78 |       22.98 |           \~60.5 | parity             |
| `td1-fill-25e.pdf` | 235 KB |      8.76 |        9.78 |           \~61.7 | ours faster        |
| `GSA_1241E.pdf`    | 582 KB |      4.75 |        4.13 |           \~60.5 | pig faster         |
| `i-130.pdf`        | 733 KB |     59.82 |       65.68 |           \~62.2 | ours faster        |
| `fw9.pdf`          | 141 KB |     13.48 |        9.79 |           \~74.1 | pig faster         |
| `g-1145.pdf`       | 251 KB |     52.04 |        2.71 |           \~48.8 | outlier to improve |

**High-level takeaways**

* Many docs land in the **same performance band** as PDFPig; some favor us, some favor PDFPig.
* Small, trivial PDFs expose fixed overhead.
* A few outliers (e.g., `g-1145.pdf`) point to branches I can optimize.
* **Memory** stays modest across runs (peak WS \~**66 MB** in these tests).

<details>
<summary>Raw samples (verbatim)</summary>

```
file	size_bytes	elapsed_ms	alloc_bytes	working_set_mb
g-1145.pdf	250968	224.653	2247392	48.4
GSA1364A-1-16d.pdf	764626	23.312	4023096	53.1
multi_text.pdf	938	0.410	19432	53.1
ssa-89.pdf	52702	4.077	1489160	54.6
A4.pdf	752	0.131	17072	54.7
two_pager.pdf	1109	0.174	26352	54.7
test_me.pdf	954	0.335	25944	54.8
i-9.pdf	524095	48.708	14263608	58.0
Resume.pdf	121157	1.409	533864	58.5
GSA_1241E.pdf	581988	8.885	2279296	58.7
GSA_1535.pdf	370402	23.625	5221776	61.0
GSA_1241.pdf	574277	13.290	1534672	62.0
td1-fill-25e.pdf	235182	9.025	2938592	62.0
GSA_14.pdf	520126	16.194	2729576	62.0
GSA_1226.pdf	307366	7.955	976040	62.1
i-130.pdf	732985	59.914	23776408	62.6
fw9.pdf	140815	8.199	3415616	62.6
GSA_1380.pdf	330983	9.889	1576328	63.8
GSA1364WH-16d.pdf	824543	33.188	6865936	66.0
GSA_1181.pdf	289178	17.711	4023424	66.2

Count=20  p50=9.46 ms  p95=68.15 ms  PeakRSS=66.2 MB
```

```
processor	file	size_bytes	elapsed_ms	alloc_bytes	working_set_mb
Ours	g-1145.pdf	250968	52.039	2247392	48.8
PdfPig	g-1145.pdf	250968	2.707	1880880	48.8
Ours	GSA1364A-1-16d.pdf	764626	23.835	4023096	52.8
PdfPig	GSA1364A-1-16d.pdf	764626	23.141	4023096	52.8
Ours	multi_text.pdf	938	0.808	19432	56.0
PdfPig	multi_text.pdf	938	0.099	19432	56.0
Ours	ssa-89.pdf	52702	6.424	1498256	56.3
PdfPig	ssa-89.pdf	52702	3.459	1453096	56.3
Ours	A4.pdf	752	0.129	17072	56.3
PdfPig	A4.pdf	752	0.080	17072	56.3
Ours	two_pager.pdf	1109	0.145	26352	56.3
PdfPig	two_pager.pdf	1109	0.113	26352	56.3
Ours	test_me.pdf	954	0.361	25944	56.3
PdfPig	test_me.pdf	954	0.111	25872	56.3
Ours	i-9.pdf	524095	46.368	14263608	57.9
PdfPig	i-9.pdf	524095	46.466	14394648	57.9
Ours	Resume.pdf	121157	1.136	533864	60.4
PdfPig	Resume.pdf	121157	1.042	533864	60.4
Ours	GSA_1241E.pdf	581988	4.751	2279296	60.5
PdfPig	GSA_1241E.pdf	581988	4.125	2279296	60.5
Ours	GSA_1535.pdf	370402	21.776	5221776	60.5
PdfPig	GSA_1535.pdf	370402	22.979	5221776	60.5
Ours	GSA_1241.pdf	574277	13.968	1534672	61.0
PdfPig	GSA_1241.pdf	574277	15.434	1535504	61.0
Ours	td1-fill-25e.pdf	235182	8.759	2938680	61.7
PdfPig	td1-fill-25e.pdf	235182	9.779	2456528	61.7
Ours	GSA_14.pdf	520126	15.585	2729576	61.7
PdfPig	GSA_14.pdf	520126	15.425	2729576	61.7
Ours	GSA_1226.pdf	307366	7.793	976040	61.7
PdfPig	GSA_1226.pdf	307366	8.271	976040	61.7
Ours	i-130.pdf	732985	59.821	23808936	62.2
PdfPig	i-130.pdf	732985	65.676	23114720	62.2
Ours	fw9.pdf	140815	13.484	3415616	74.1
PdfPig	fw9.pdf	140815	9.793	3411832	74.1
Ours	GSA_1380.pdf	330983	11.408	1576328	76.4
PdfPig	GSA_1380.pdf	330983	13.161	1576352	76.4
Ours	GSA1364WH-16d.pdf	824543	40.708	6997032	75.2
PdfPig	GSA1364WH-16d.pdf	824543	38.831	6865936	75.2
Ours	GSA_1181.pdf	289178	16.527	4023424	73.4
PdfPig	GSA_1181.pdf	289178	18.282	4023424	73.4
```

</details>

---

## How it works (high-level)

* **Parser pipeline:** xref/trailer → object table → stream decoding → content parsing
* **AcroForm model:** typed in-memory field objects (text, checkbox, radio, combos) with setters that update dictionaries, appearances (where present), and incremental write
* **Writer:** conservative PDF 1.7 writer; no linearization/compression yet (clarity first)
* **Performance posture:** eager object materialization for simplicity + fast random access; allocation-aware hot loops

---

## Real-world use: powering **SubmitPath**

This engine drives **SubmitPath**, a forms-over-data platform. Typical flow:

1. Load government/firm PDF forms (AcroForm).
2. Bind fields to data (from CRM/HRIS/user input).
3. Validate and **write a filled PDF** for signature or archive.

The **mutable AcroForm** capability—without paid licensing—is the difference between prototype and production for my use case.

---

## Roadmap

* **Encryption:** broader crypto support; user-supplied password API
* **Large files:** I/O buffering or memory-mapped reads for **>2 GB**
* **Lazy parsing modes:** avoid full object materialization when not needed
* **Writer:** compression, linearization, object streams
* **Layout/OCR:** document layout analysis + OCR hooks
* **Richer mutability:** add text/shapes/images; image extraction & modification
* **General polish:** more spec corners as they matter to SubmitPath

---

## Closing

This started as “I just need **mutable AcroForms** in .NET,” and became a **fast, practical PDF processor** that holds its own next to mature libraries—purpose-built for forms. I plan to open-source the core under **MIT** (web app code stays private) and continue growing features as **SubmitPath** needs them.

**Interested in the parser or the SaaS?**

* Ping me for a deep dive, profiles, or code review.
* Watch **submitpath.com** for launch updates.
