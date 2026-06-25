# LinkedIn Profile Extraction Benchmark

How well do web-search / answer APIs extract a **specific person's** LinkedIn profile when you hand them the exact profile URL?

We gave four APIs — **Linkup, Exa, Perplexity, Parallel** — the same task for **500 real LinkedIn profiles** and scored every response on two axes:

1. **Completeness** — did it fill the fields we asked for?
2. **Correctness** — is it the *right person*, with data matching ground truth?

---

## The task

For each of 500 people, every API gets the identical prompt + JSON schema:

> Return the full LinkedIn profile of `{url}` (`{name}`). Extract: full name, location, headline,
> current company and title, complete work experience (company/title/start/end), education,
> and most recent posts (text, date, like/comment counts).

Each API is called through its **own native structured-output surface** (no Claude/LLM layer in the extraction step), so we're measuring the API itself:

| API | Endpoint | Structured output | Fetches the given URL? |
|---|---|---|---|
| **Linkup** | `POST /v1/search` | `outputType:"structured"` + `structuredOutputSchema` | **Yes** — dereferences the URL |
| **Exa** | `POST /search` | `outputSchema` | No — semantic search (similar pages) |
| **Perplexity** | `POST /chat/completions` (`sonar-pro`) | `response_format.json_schema` | No — agentic web search |
| **Parallel** | Task API `POST /v1/tasks/runs` (`lite`) | `task_spec.output_schema` | No — agentic web research |

That last column is the whole story: only Linkup **fetches the URL you gave it**. The others *search* for the name and tend to land on a **namesake**.

---

## The two evals

### 1. Completeness (deterministic, 10-field rubric)
Each requested field that comes back non-empty scores 1 point → `score = filled / 10 × 100`.
Fields: `full_name, location, headline, current_company, current_title, experience, experience-with-dates, education, recent_posts, post-engagement`.

This measures *how much* came back — but **not whether it's the right person**. An API can score high here by confidently filling boxes for the wrong human.

### 2. Correctness (vs. a real ground-truth DB)
We join each response to `data/ground_truth_500.csv` (the authoritative profile for that URL) and check:

- **right-person** — does the returned `current_company` match the real person's company? *(the namesake detector)*
- **title ✓** / **name ✓** — fuzzy match vs ground truth
- **wrong (namesake)** — confidently returned a company that **isn't** the real person's
- **empty** — abstained / no company returned

Matching is token-set with company-suffix stripping (e.g. `Capital Bancorp Plc` ≈ `Capital Bancorp`). Ground truth is an **independent** DB — not derived from any of the four APIs — so there's no self-reference bias.

---

## Results (n = 500)

### Completeness
| Provider | Mean | Median | Fully empty |
|---|---|---|---|
| **Linkup** | **96.5** | 100 | 13 |
| Perplexity | 73.7 | 80 | 38 |
| Exa | 71.9 | 80 | 8 |
| Parallel | 60.5 | 80 | 74 |

### Correctness
| Provider | Right person | Wrong (namesake) | Empty | Title ✓ | Name ✓ |
|---|---|---|---|---|---|
| **Linkup** | **94%** | **12** | 20 | 93% | 97% |
| Perplexity | 64% | 122 | 57 | 61% | 86% |
| Parallel | 63% | 69 | 114 | 58% | 85% |
| Exa | 56% | **191** | 27 | 54% | 92% |

### What it means
- **Name match is high for everyone (85–97%) — but company match collapses to ~56–64%** for the three that search instead of fetch. They return *a person with the right name* at the **wrong company**.
- **Exa returns the wrong human 191/500 times (38%).** Perplexity 122, Parallel 69 (+114 abstentions).
- **Linkup gets the wrong company only 12/500 times (94% correct)** because it dereferences the URL instead of searching for the name.
- On completeness, the gap is in the **deep, verifiable fields** — dated work history (Exa 0%), education, and posts + engagement (only Linkup returns posts at scale).

**Takeaway:** completeness alone flatters search-based APIs because they fill the name box; correctness exposes that ~36–44% of their filled profiles are the *wrong person* or empty. For URL-anchored profile extraction, fetching the URL (Linkup) beats searching for the name.

---

## Run it yourself

```bash
pip install openpyxl
python3 eval.py
```

Outputs `results/benchmark_500.xlsx`:
- **Summary** — both evals + per-field fill rates
- **Completeness** — per-person score for each API (color-scaled)
- **Correctness detail** — per-person: real company/title vs. what each API returned, with ✓ / ✗ / — flags

---

## Files

```
data/
  api_outputs_500.csv    # each API's structured response per profile (provider,endpoint,query,response,params)
  ground_truth_500.csv   # authoritative profile fields per linkedin_url (the source of truth)
eval.py                  # runs both evals, writes the Excel
results/
  benchmark_500.xlsx     # generated report
```

## Method notes & caveats
- **Completeness ≠ correctness.** A filled field isn't necessarily the right person — that's exactly why both evals exist.
- All APIs were called at their **base/standard tier** (Linkup `standard`, Exa `auto`, Perplexity `sonar-pro`, Parallel `lite`) with no result/content caps that would handicap any one of them.
- Matching is fuzzy by design (companies are written many ways); thresholds live at the top of `eval.py` if you want to tighten them.
