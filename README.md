# Ollama Models Comparison for Local AI

Comprehensive benchmark comparing **GLM-5 Cloud**, **Qwen 3.5 Cloud**, and **Gemma 4 Cloud** models on two major use cases:

1. **Coding Agent Benchmark** — Code generation, bug fixing, code review, refactoring, test writing, architecture design
2. **App Modernisation Benchmark** — PL/SQL rules extraction, Java JEE documentation, OpenAPI spec generation, .NET forward engineering, integration design, test specifications

## 🚀 Quick Results

### Coding Agent (6 tests)
| Model | Total Time | Status |
|-------|-----------|--------|
| **Qwen 3.5** | **256.4s** 🏆 | Best for agents |
| Gemma 4 | 439.9s | Good fallback |
| GLM-5 | 808.9s | Offline only |

**Winner: Qwen 3.5** — Fastest all-rounder with excellent code quality.

### App Modernisation (6 tests)
| Model | Total Time | Output |
|-------|-----------|--------|
| **Qwen 3.5** | **324.0s** 🏆 | 8,574 words |
| Gemma 4 | 535.3s | 4,813 words |
| GLM-5 | 1005.9s | 7,490 words |

**Winner: Qwen 3.5** — Fastest, most detailed, wins 4/6 tests.

## 📊 Reports

### Full Benchmark Reports
- [Coding Agent Comparison](docs/GLM-5-COMPARISON.md)
- [App Modernisation Comparison](docs/GLM-5-APPMOD-COMPARISON.md)
- [App Modernisation PDF](docs/GLM-5-APPMOD-COMPARISON.pdf)

### Individual Test Results
- [Coding benchmark results](docs/test-results/) — 18 result files (6 tests × 3 models)
- [App modernisation results](docs/test-results-appmod/) — 18 result files (6 tests × 3 models)

## 🎯 Key Findings

### Qwen 3.5:397b-cloud
- ✅ Fastest across both benchmarks
- ✅ Best output depth (8,574 words for appmod)
- ✅ Wins 8/12 test categories overall
- ✅ **Recommended as primary model for OpenClaw and modernisation workflows**

### Gemma 4:31b-cloud
- ⭐ Fastest at code generation (19.7s)
- ⭐ Good fallback/backup model
- ⭐ Decent quality on forward engineering
- ✅ Use when speed is critical

### GLM-5:cloud
- 🔬 3.1x slower than Qwen for coding tasks
- 📚 Most comprehensive OpenAPI specs
- 🐛 Best at catching subtle legacy code bugs
- ⚠️ **Only use for offline batch analysis**

## 🔧 Setup & Testing

To replicate these benchmarks:

```bash
# Required: Ollama with models installed
ollama pull qwen3.5:397b-cloud
ollama pull gemma4:31b-cloud
ollama pull glm-5:cloud

# Run coding benchmark
./coding-agent-test-glm.sh

# Run app modernisation benchmark
./appmod-agent-test-glm.sh
```

## 📖 GitHub Pages

This repository is published as **GitHub Pages** at:
- [Main benchmark dashboard](https://arshimkola.github.io/ollama-models-compare-for-local-ai/)

All reports, benchmarks, and test results are interactive and browsable directly from the site.

## 📋 Test Categories

### Coding Agent Tests
1. **Code Generation** — Implement `merge_sorted_streams()` with heap-based merge
2. **Bug Fixing** — Identify and fix threading bugs in a RateLimiter
3. **Code Review** — Security audit of vulnerable REST API
4. **Refactoring** — Clean up nested business logic
5. **Test Writing** — Comprehensive Jest tests for LRU Cache
6. **Architecture** — Design webhook delivery system with reliability patterns

### App Modernisation Tests
1. **PL/SQL Rules Extraction** — Extract 13+ business rules from insurance claims procedure
2. **Java JEE Documentation** — Document legacy EJB order fulfillment system
3. **OpenAPI Spec Generation** — Generate complete OpenAPI 3.1 spec from requirements
4. **Forward Engineering** — Convert .NET Framework WCF to .NET 8 Minimal API
5. **Integration Design** — Decouple monolith into microservices with async/sync patterns
6. **Test Specification** — Write comprehensive test spec from legacy business rules

## 📊 Methodology

- **Cloud-only testing** — All three models are Ollama cloud variants (no local GPU required)
- **Same prompts** — Identical test inputs across all three models
- **Production-ready code** — Tests designed to evaluate real-world agent and modernisation tasks
- **Timing** — Wall-clock time from curl request to response completion
- **Output quality** — Manual review of code, documentation, and design accuracy

## 👤 Author

Benchmark suite created by Alok Mishra | April 2026

## 📄 License

Test prompts, results, and reports are shared publicly for the developer community. Individual model outputs respect each vendor's terms.
