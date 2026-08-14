# Embedding Similarity Calibrator

Paste your own embedding vectors and get the empirical cosine distribution of unrelated pairs, drawn against the exact closed-form null for random unit vectors in the same dimension.

**Live demo:** https://0xelitesystem.github.io/embedding-similarity-calibrator/

## Use

Your retrieval threshold is 0.8. Against what?

There are two answers and they disagree by a factor of four. At d=1536 the null distribution for two random unit vectors has standard deviation exactly 1 over the square root of 1536, which is 0.0255, so 0.8 sits 31.4 standard deviations out and the exact tail probability is about 10 to the power -342. Against a real embedder the same 0.8 is a different number entirely, because real unrelated text does not sit at cosine 0. The built-in example corpus at d=1536 puts unrelated pairs at 0.31 with a standard deviation of 0.060, so 0.8 sits 8.1 standard deviations out, not 31.4. Only one of those two numbers is about your index.

The page computes both from whatever you paste, and prints the cutoff at which 10 percent, 5 percent, 1 percent and 0.1 percent of unrelated pairs clear it.

1. Paste vectors. One per line, or a JSON array of arrays, or a whole embeddings API response body: the parser walks the JSON and picks up any array of numbers, so an `embedding` field anywhere in the structure is found.
2. Optionally paste one query vector and set your threshold.
3. Press Calibrate. Four example corpora are built in, generated in the page from fixed seeds so every reader gets the same numbers.

What comes back: where your threshold sits against both baselines, cutoffs with a stated false-positive rate, the mean-vector norm, a gated mean-removal rerank, an index-hygiene report, and a live seeded Monte Carlo that checks the null identity in front of you instead of asserting it.

## Why this exists

Threshold numbers get copied between projects. A 0.75 that worked for someone else's model on someone else's corpus becomes a constant in your retriever, and nothing in the stack ever states what it was measured against. Cosine similarity has no units and no absolute meaning: it is only interpretable against a distribution, and the distribution belongs to the model and the corpus, not to the metric.

Three decisions in this build are corrections to the obvious version of it.

**Mean removal is not the headline.** Subtracting the corpus mean vector before scoring is the best known trick in this area, and it is the one thing on the page that is allowed to do nothing. The size of the effect is set by the norm of the mean of your unit-normalized vectors, because that is exactly the shared term that mean removal deletes from every pairwise cosine. On an embedder with a small mean-vector norm the deletion changes almost nothing, and a tool whose marquee demo silently no-ops on a large fraction of modern retrieval models is worse than no tool. So the mean-vector norm is reported as the primary anisotropy number, the rerank sits behind it, and the expected outcome is printed before the rerank runs: at or above 0.5 rankings should move, below 0.5 they should not. A null result then reads as a correct measurement of your embedder rather than a broken feature. The two built-in corpora demonstrate both outcomes.

**Index hygiene is a gate, not a decoration.** Four conditions block calibration outright: a dimension split from a half-finished migration, zero vectors from empty chunks, norms outside tolerance, and a ranked list sorted as if a distance were a similarity. A calibration computed on top of any of those is a number with no meaning attached, and printing it next to a warning badge is how it gets believed anyway.

Two details there are load-bearing. The norm check is tolerance based, because a unit vector stored in float32 usually does not have norm 1, and a gate written as `norm == 1` rejects healthy vectors. The page measures that live rather than asserting a constant: it builds eight unit vectors at your dimension, rounds the components to float32, and reports how many came back with a norm that was not exactly 1 under float64 accumulation and under float32 accumulation, with the largest deviation in each. At d=1536 the first probe reads 1.0000000003 and 0.9999997616, and the second of those is a float32 neighbour of 1 rather than 1. The result also depends on the vector and on where the sum was accumulated, which is the actual argument for a tolerance: the equality test does not fail consistently, it fails unpredictably. And duplicates are reported in two classes, because bit-identical vectors and vectors at cosine 1 minus 1e-6 are different bugs: the first is the same text embedded twice, an ingest loop or a retry without an idempotency key, and the second is two different strings landing in nearly the same place, which is boilerplate or a shared footer. Merging the counts hides whichever is smaller.

**The random-pair baseline is an assumption, and it is labelled as one.** Random pairs are a fair proxy for unrelated pairs in a broad corpus. In a topically narrow index, one product's documentation or one legal domain, random pairs really are related, the measured distribution shifts right, and every cutoff derived from it comes out too high with a tight interval attached. The page cannot detect that, because detecting it needs labels it does not have, so it says so instead of hiding it.

The math is exact where it can be. The null density is proportional to (1-x^2) to the power (d-3)/2, which underflows to zero at d=1536 long before the interesting thresholds, so everything is evaluated in log space: a Lanczos log-gamma for the beta normalizer and a Lentz continued fraction for the regularized incomplete beta. The implementation is checked at run time against three cases with elementary closed forms, at d=2, d=3 and d=4, and the page prints the errors.

## Privacy

Your vectors never leave the browser. There is no network request of any kind: no fonts, no analytics, no API call, no image fetch. One HTML file, no dependencies. The only thing stored is the light or dark theme choice in localStorage, wrapped in try/catch so it degrades quietly in private mode.

A blob-URL Worker is used for the heavy passes, which makes no network request and falls back to running on the main thread if Worker construction is blocked.

## Run locally

```
git clone https://github.com/0xelitesystem/embedding-similarity-calibrator.git
cd embedding-similarity-calibrator
```

Open `index.html` in a browser. That is the whole procedure. It works from `file://` with no server.

## Build

There is no build step. One HTML file with inline CSS and inline JavaScript, no dependencies, no bundler, no package manifest. Edit the file, reload the page.

## Related

- [embedding-cost-estimator](https://0xelitesystem.github.io/embedding-cost-estimator/) - what a corpus costs to embed across providers, before you commit. That is the decision before this one. It prices dimensions; this page tells you what a similarity score in one of them means.
- [rag-chunk-visualizer](https://0xelitesystem.github.io/rag-chunk-visualizer/) - how four chunking strategies actually split your text. Chunking decides what gets embedded; this page measures the vectors that come out.
- [embeddings-and-vector-search-basics-reference](https://github.com/0xelitesystem/embeddings-and-vector-search-basics-reference) - the concepts, in plain language.
- [rag-basics-reference](https://github.com/0xelitesystem/rag-basics-reference) - retrieval-augmented generation end to end.

## License

MIT. See [LICENSE](LICENSE).

## More

Part of a catalog of single-file browser tools and plain-language references,
all MIT licensed and dependency-free: [0xelitesystem.github.io](https://0xelitesystem.github.io/).
Built by [elitesystem.ai](https://elitesystem.ai).
