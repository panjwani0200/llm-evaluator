# LLM Evaluation Pipeline

**Production-ready** evaluation system for scoring AI responses across 6 critical metrics in real-time.

## Overview

This project implements an automated LLM evaluation pipeline that scores AI-generated responses using:

- **Response Relevance** (25%): Semantic similarity between response and query+context using embeddings
- **Completeness** (20%): Coverage of key query requirements via keyword matching
- **Hallucination Detection** (25%): Identification of unsupported claims using context alignment
- **Factual Accuracy** (30%): Verification of response facts against provided context
- **Latency** (ms): End-to-end pipeline execution time
- **Cost Estimation** (USD): Token-based pricing for 4 major LLM providers

## Architecture

```
Data Input (2 JSONs)
    |
    v
[DataLoader] --> Extract Query + Response + Context
    |
    +---> [TextMetrics] ------> Relevance, Completeness, Hallucination, Factual Accuracy
    |
    +---> [SystemMetrics] -----> Latency, Cost
    |
    v
[Evaluator] --> Aggregate Scores & Output Report
```

## Key Design Decisions

### 1. **Semantic Embeddings Over Rule-Based**
- Uses `SentenceTransformer` (all-MiniLM-L6-v2) for efficient similarity computation
- Lightweight model: ~22MB, fast inference on CPU (~50ms per sentence)
- Cost: $0 (open-source alternative to API-based embedding services)

### 2. **Chunked Context Processing**
- Breaks long contexts into sentences for granular hallucination detection
- Threshold-based scoring (0.45 cosine similarity) for context alignment

### 3. **Token Counting for Cost Estimation**
- Uses `tiktoken` (OpenAI's tokenizer) for accurate token counts
- Supports 4 providers: OpenAI, Claude, Gemini, Local
- Configurable pricing rates (updated to Dec 2024 rates)

### 4. **Scalability for Millions of Daily Evals**

**Low Latency (<500ms target)**:
- Embedding computation: ~100ms for avg response
- Keyword extraction: ~10ms
- Cost calculation: ~5ms
- Total overhead: <200ms beyond inference

**Low Cost (<$0.01 per eval)**:
- Local embedding model: $0 ops cost
- Gemini pricing: ~$0.0003 per eval (fastest)
- Caching strategy: Batch 100 evals, compute shared embeddings once
- Async processing with `asyncio` for parallel context fetches

**Optimization Techniques**:
1. **Embedding Cache**: Store computed embeddings to avoid recomputation
2. **Batch Processing**: Process multiple evaluations in parallel
3. **Lazy Loading**: Load models only when needed
4. **Provider Selection**: Use cheapest provider for non-critical metrics

## Installation

```bash
git clone https://github.com/panjwani0200/llm-evaluator.git
cd llm-evaluator

pip install -r requirements.txt
```

## Quick Start

```python
from src.runner import LLMEvaluator

evaluator = LLMEvaluator(provider="openai")

result = evaluator.evaluate(
    conversation_path="samples/conversation.json",
    context_path="samples/context.json"
)

print(f"Overall Score: {result['overall_score']}")
print(f"Cost: ${result['metrics']['estimated_cost_usd']:.6f}")
print(f"Latency: {result['metrics']['latency_ms']}ms")
```

## JSON Input Format

**conversation.json**:
```json
{
  "conversation_id": "conv_001",
  "messages": [
    {"role": "user", "content": "What is machine learning?"},
    {"role": "assistant", "content": "Machine learning is..."}
  ]
}
```

**context.json**:
```json
{
  "context_id": "ctx_001",
  "query": "What is machine learning?",
  "chunks": [
    {"text": "ML is a subset of AI...", "score": 0.95},
    {"text": "It learns from data...", "score": 0.87}
  ]
}
```

## Output Format

```json
{
  "conversation_id": "conv_001",
  "overall_score": 0.82,
  "metrics": {
    "relevance": {"score": 0.89, "explanation": "..."},
    "completeness": {"score": 0.75, "explanation": "..."},
    "hallucination": {"score": 0.95, "explanation": "..."},
    "factual_accuracy": {"score": 0.88, "explanation": "..."},
    "latency_ms": 187,
    "estimated_cost_usd": 0.000425
  }
}
```

## PEP-8 Compliance

- All code follows PEP-8 style guide
- Max line length: 100 characters
- Type hints throughout
- Docstrings for all functions

## Testing

```bash
python -m pytest tests/
```

## Production Deployment

**For 1M daily evaluations**:
1. Deploy on Kubernetes with auto-scaling
2. Use Redis for embedding cache (TTL: 7 days)
3. Batch requests: queue + process 100 at a time
4. Monitor: latency p95<300ms, cost <$0.008/eval
5. DB: Store results in PostgreSQL with indexing on conv_id

## Future Improvements

- [ ] Fine-tune embeddings on domain-specific data
- [ ] Add LLM-as-judge for harder metrics (hallucination confidence)
- [ ] Support multi-language evaluation
- [ ] Add A/B testing framework for response comparison

## License

MIT

## Author

Built for BeyondChats LLM Engineer Internship - Dec 2025
