# Financial data SDK for Go

A Go SDK demonstration for financial research: retrieve the data available at
a chosen cutoff, resume an interrupted download, and keep the versions an
earlier analysis used.

I'm [Nathan Slaughter](https://nathanslaughter.com/), a software engineer with a
background in investment research and portfolio management. My approach to
client libraries comes partly from Go library development. I help data teams
design the client interfaces, examples, and release processes their customers
depend on.

**Status:** Project brief. This repository currently contains this README.
The SDK, tests, and runnable examples are planned; nothing described here has
been implemented or tested yet. The demo API and the shared data contract live
in [financial-data-api](https://github.com/nslaughter/financial-data-api). The
dataset is synthetic, and this is a demonstration project, not client work.

## What this project demonstrates

This is a supporting example for my SDK development and maintenance work. It
covers the same customer workflow as the
[Python SDK](https://github.com/nslaughter/financial-data-sdk-python) and the
[TypeScript SDK](https://github.com/nslaughter/financial-data-sdk-ts), written
the way a Go developer expects. It is meant to show:

- **One contract, idiomatic in each language.** The three SDKs return the same
  records for the same queries. Each follows its own language's conventions
  for errors, iteration, cancellation, and packaging.
- **The same checks for every SDK.** CI runs the shared data contract's
  expected results against the same pinned demo API image the other SDKs use.
- **Data whose meaning survives the client.** Records keep observation and
  revision identities, observation periods, availability times, decimal
  values, units, and missing values.
- **Downloads that can resume.** Pages and continuation tokens are available
  alongside a record iterator, so a scheduled job can checkpoint its work.
- **Failures a customer can act on.** Denied access, invalid credentials,
  invalid input, throttling, and transient errors are distinguishable from one
  another and from an empty result.

This SDK follows the Python SDK in the first of four stages of a demonstration
for financial data providers. The [financial-data-api](https://github.com/nslaughter/financial-data-api)
supplies the demo API and expands it in the second stage, and the
[financial-data-api-monitor](https://github.com/nslaughter/financial-data-api-monitor)
checks what customers receive from it.

## A successful download should leave the researcher able to explain the result

The example follows a fictional monthly activity index. Its August 2026 value
is released as 102.4 on September 3, then revised to 102.1 on September 10.
A researcher reproducing an analysis made on September 4 still needs 102.4.
A scheduled job that loses its connection on the fourth page needs to know
which records it saved and where to resume.

## How the first demonstration will work

1. Add the tagged module to a clean Go module outside this repository and start
   the demo API from the container image that financial-data-api publishes, at
   a pinned version. The demonstration needs no external data credentials.
2. Retrieve the observations available at a chosen cutoff, retaining their
   observation and revision identities.
3. Interrupt a paginated download, resume from its saved checkpoint, and
   compare the local records with the expected results in the shared data
   contract.
4. Follow releases, revisions, and withdrawals through an update cursor,
   keeping the version with the highest revision number current.
5. Revoke the credential's access to the series, force throttling beyond the
   retry budget, and introduce a revision during a download.

The intended interface looks like this. It is a sketch; no module has been
published.

```go
client := financialdata.NewClient(os.Getenv("FINANCIAL_DATA_API_KEY"))

query := financialdata.ObservationQuery{
	SeriesID:      "activity-index",
	PeriodStart:   financialdata.Date{Year: 2026, Month: time.August, Day: 1},
	PeriodEnd:     financialdata.Date{Year: 2026, Month: time.September, Day: 1},
	AvailableAsOf: time.Date(2026, time.September, 4, 0, 0, 0, 0, time.UTC),
}
for obs, err := range client.Observations(ctx, query) {
	if err != nil {
		return err
	}
	fmt.Println(obs.PeriodStart, obs.Value, obs.RevisionID)
}
```

## Design choices the examples will make visible

- **Synchronous calls that take a context.** Every method accepts a
  `context.Context`, and its deadline covers requests and retry waits. No
  goroutine outlives the call that started it; callers add concurrency where
  they want it.
- **Iteration without hidden work.** `Observations` returns an
  `iter.Seq2[Observation, error]` that fetches a page only when the loop needs
  one, and breaking out of the loop stops further requests. A page-level
  method returns each page with its continuation token for jobs that
  checkpoint.
- **Errors that work with `errors.Is` and `errors.As`.** Sentinel errors
  identify denied access, invalid credentials, throttling, and an expired
  snapshot. An `*APIError` carries the status, the provider's request ID, and
  any `Retry-After`, never the credential. An empty result is no records and a
  nil error.
- **Types that keep the data's meaning.** A decimal string type preserves each
  value exactly instead of converting it to `float64`. Observation periods use
  a date type without a time zone, and a missing value stays distinct from zero.
- **The client fits the application around it.** Customers pass their own
  `*http.Client` and, optionally, a `*slog.Logger`. The package has no global
  state and depends only on the standard library.
- **Releases follow Go's module rules.** Versions are semantic tags. A breaking
  change gets a new major version with its own module path, so old and new
  versions can coexist in one build. CI covers the Go releases the Go project
  currently supports.

## Scope of the first release

The first release covers the same synthetic dataset and research workflow as
the Python SDK: authentication, typed records, pagination, errors, bounded
retries, resumable downloads, and update following. Whether the later migration
stage covers this SDK will be decided when that stage is planned.

## The demonstration is complete when

- The tagged module installs in a clean module, and the documented workflow
  runs against the pinned demo API, passing the shared contract's checks for
  this stage.
- Authentication and rate-limit errors are reported distinctly, and retries
  stop within their configured bounds.
- An interrupted paginated download resumes with no missing or duplicate
  records.
- The September 4 query still returns 102.4 after the revision is loaded.

## What the repository will contain

- A customer-focused README and quickstart.
- A runnable research example, and a scheduled-job example that checkpoints
  and resumes.
- Documentation for every exported identifier, mapped to the shared data
  contract.
- CI that runs `go vet`, the tests, the examples, and the contract's checks
  against the pinned demo API image on each supported Go release.
- A tagged module release.
- Documented limitations and a clear demonstration label.

## Related projects and writing

- [financial-data-sdk-python](https://github.com/nslaughter/financial-data-sdk-python)
  and [financial-data-sdk-ts](https://github.com/nslaughter/financial-data-sdk-ts):
  the same client in Python and TypeScript.
- [financial-data-api](https://github.com/nslaughter/financial-data-api): the
  demo API, the shared data contract, and the full API.
- *Building an SDK your customers love*: an article on the design behind these
  SDKs, in preparation. I'll link it here when it is published.

## Work with me on an SDK your customers can use

I take on SDK projects scoped around the workflows your customers need to
complete. The work can include interface design, implementation, documentation,
release packaging, compatibility checks, and ongoing maintenance.

[Discuss an SDK project](https://www.linkedin.com/in/nathan-slaughter) with the
API, target language, and customer workflow you need to support.
