+++
date = '2026-02-02T12:00:00-03:00'
draft = false
title = 'Running AI Models on Mobile: ONNX, Flutter, and the Quest for On-Device Intelligence'
+++

## The Mobile AI Challenge

Can you run real AI—embeddings, semantic search, text generation—on a phone? Not through API calls to OpenAI, but actually **on the device**?

The answer is yes, but it requires solving serious constraints:

- **Storage:** App stores reject 200MB apps
- **Memory:** Phones have 4-8GB RAM, shared with OS and other apps
- **Battery:** ML inference drains batteries fast
- **Performance:** CPUs are slower than desktop GPUs

We experimented with running transformer models on iOS and Android using ONNX Runtime and Flutter. Here's what we learned.

## Two Experiments: Embeddings and Generation

### Experiment 1: Embeddings for Semantic Similarity

**Goal:** Run all-MiniLM-L6-v2 for text embeddings on mobile

**Use case:** Check if two sentences are similar without internet

**Challenge:** The model is 86MB (FP32), too large for mobile apps

### Experiment 2: Text Generation

**Goal:** Run SmolLM2-135M for autoregressive text generation

**Use case:** Generate poems, summaries, responses—entirely on-device

**Challenge:** Text generation is computationally expensive and requires managing state (KV cache)

Both experiments succeeded, but required specific optimizations.

## The Quantization Solution

### The Problem: FP32 Models are Too Large

Standard transformer models use FP32 (32-bit floating point) weights:

- **all-MiniLM-L6-v2:** 86MB
- **SmolLM2-135M:** 540MB

These sizes are unacceptable for mobile apps.

### The Solution: INT8 Quantization

Quantization converts FP32 weights to INT8 (8-bit integers):

```
FP32: -3.14159... → INT8: -127 to 127 (mapped)
```

**Impact:**
- **4x size reduction:** 86MB → 22MB (embeddings), 540MB → 135MB (generation)
- **4x speed improvement:** INT8 math is faster on ARM CPUs
- **iOS compatibility:** ONNX Runtime 1.22.0 on iOS requires INT8 for many ops

**Quality loss:** Minimal for text tasks (unlike images where quantization causes visible artifacts)

## Experiment 1: Mobile Embeddings with ONNX

### Setup

- **Framework:** Flutter (Dart)
- **Runtime:** flutter_onnxruntime (ONNX Runtime 1.22.0)
- **Model:** all-MiniLM-L6-v2 (INT8 quantized, 22MB)
- **Tokenizer:** BERT WordPiece (Dart implementation)

### Implementation

**1. Load the Model**

```dart
final session = await OrtSession.fromAsset(
  'assets/models/model_quantized.onnx'
);
```

Model loading takes ~1.2s (one-time cost).

**2. Tokenize Input**

```dart
class BERTTokenizer {
  final Map<String, int> vocab; // 30k tokens

  List<int> encode(String text) {
    final tokens = _wordPiece(text.toLowerCase());
    return tokens.map((t) => vocab[t] ?? vocab['[UNK]']!).toList();
  }
}
```

**3. Run Inference**

```dart
Future<List<double>> getEmbedding(String text) async {
  // Tokenize
  final inputIds = tokenizer.encode(text);

  // Prepare input tensor
  final inputTensor = OrtValue.fromList(
    [inputIds],
    [1, inputIds.length]
  );

  // Run inference
  final outputs = await session.run({'input_ids': inputTensor});

  // Extract embedding (mean pooling)
  final embeddings = outputs['last_hidden_state']!.value;
  return _meanPooling(embeddings);
}
```

**4. Calculate Similarity**

```dart
double cosineSimilarity(List<double> a, List<double> b) {
  final dotProduct = _dotProduct(a, b);
  final normA = sqrt(_dotProduct(a, a));
  final normB = sqrt(_dotProduct(b, b));
  return dotProduct / (normA * normB);
}
```

Pure math, no ML library needed.

### Performance

Measured on physical devices:

| Device | Inference Time | Memory Usage |
|--------|---------------|--------------|
| iPhone 13 | ~180ms | ~45MB |
| Pixel 6 (Android) | ~210ms | ~50MB |

**Result:** Sub-200ms is fast enough for real-time similarity checks.

### Quality

Tested various text pairs:

- "dog" vs "puppy": **0.89** (High similarity)
- "computer" vs "machine": **0.75** (Related concepts)
- "apple" vs "orange": **0.60** (Different but same category)

Embeddings match human intuition for English text.

### iOS Compatibility Issues

**Critical finding:** FP32 models fail on iOS ONNX Runtime.

- **Error:** "Protobuf parsing failed"
- **Cause:** iOS ONNX Runtime 1.22.0 has stricter compatibility requirements
- **Solution:** Use INT8 quantized models specifically

This made quantization **mandatory**, not optional, for iOS support.

## Experiment 2: Text Generation on Mobile

### Setup

- **Framework:** Flutter (Dart)
- **Runtime:** flutter_onnxruntime
- **Model:** SmolLM2-135M-Instruct (INT8, 130MB)
- **Tokenizer:** GPT-2 BPE (Byte-Pair Encoding)
- **Optimization:** KV Cache (critical for performance)

### The KV Cache Breakthrough

Text generation without KV cache is **unusable on mobile**:

**Without KV Cache:**
- Every generation step re-computes attention for entire sequence
- Token 1: Compute attention for 10 prompt tokens
- Token 2: Compute attention for 11 tokens (10 prompt + 1 generated)
- Token 3: Compute attention for 12 tokens
- ...
- Token 20: Compute attention for 30 tokens

**Cost:** ~500ms per token = 10s for 20 tokens

**With KV Cache:**
- First step: Compute attention for all prompt tokens, save Key-Value pairs
- Subsequent steps: Only compute attention for new token, reuse cached KV
- Token 1: Full computation (10 tokens)
- Token 2: Only new token + cached prompt
- Token 3: Only new token + cached prompt + cached token 2
- ...

**Cost:** ~150ms per token = 3s for 20 tokens

**Speedup:** 3.3x faster

### Implementation

```dart
class TextGenerator {
  final OrtSession session;
  final GPT2Tokenizer tokenizer;

  Future<String> generate(String prompt, GenerationConfig config) async {
    final inputIds = tokenizer.encode(prompt);
    final generatedIds = List.from(inputIds);

    // Past Key Values (Cache)
    Map<String, OrtValue>? pastKeyValues;

    for (int i = 0; i < config.maxNewTokens; i++) {
      // Build input (full sequence first time, last token only after)
      final inputTensor = _buildInputTensor(generatedIds, pastKeyValues);
      final attentionMask = _buildAttentionMask(generatedIds.length);

      // Run inference (pass KV cache if available)
      final inputs = {
        'input_ids': inputTensor,
        'attention_mask': attentionMask,
        if (pastKeyValues != null) ...pastKeyValues
      };

      final outputs = await session.run(inputs);

      // Update KV cache for next step
      pastKeyValues = _extractPastKeyValues(outputs);

      // Sample next token (temperature + TopK)
      final nextTokenId = _sampleNextToken(
        outputs['logits']!,
        config.temperature,
        config.topK
      );

      generatedIds.add(nextTokenId);

      if (nextTokenId == config.stopTokenId) break;
    }

    return tokenizer.decode(generatedIds);
  }
}
```

### Sampling: Temperature and TopK

Greedy decoding (always pick highest probability) produces repetitive text:

```
"I like cats. I like cats. I like cats."
```

**Temperature** adds randomness:
- **0.0:** Greedy (deterministic, robotic)
- **0.8:** Balanced (human-like)
- **1.5:** Creative (chaotic)

**TopK** limits sampling to top K tokens:
- Vocabulary size: ~50,000 tokens
- TopK=40: Only sample from top 40 most likely tokens
- Filters out nonsense punctuation and random symbols

```dart
int _sampleNextToken(OrtValue logits, double temp, int topK) {
  final scores = logits.value as List<double>;
  final probs = _softmax(scores);
  final topIndices = _topKIndices(probs, topK);
  final adjustedProbs = _adjustTemperature(probs, topIndices, temp);
  return _weightedRandom(adjustedProbs);
}
```

### Performance

| Device | Time/Token (No Cache) | Time/Token (With Cache) | Total (20 tokens) |
|--------|----------------------|------------------------|-------------------|
| iPhone 13 | ~450ms | ~150ms | ~3s |
| Pixel 6 | ~520ms | ~180ms | ~3.6s |

**Result:** 3 seconds for a short poem is acceptable for mobile UX.

### Output Quality

Tested various prompts:

**Prompt:** "Write a haiku about coding"

**Output:**
```
Code flows through my mind
Algorithms dance and play
Logic becomes art
```

✅ Valid 5-7-5 structure

**Prompt:** "What is machine learning?"

**Output:**
```
Machine learning is a type of artificial intelligence
that allows computers to learn from data without being
explicitly programmed...
```

✅ Coherent explanation

**Observation:** SmolLM2-135M is smart enough for general tasks. INT8 quantization doesn't degrade quality noticeably.

## Technical Challenges Solved

### 1. BERT WordPiece Tokenization in Dart

No pre-built Dart tokenizer existed. We implemented:

```dart
List<String> _wordPiece(String text) {
  List<String> tokens = [];
  for (final word in text.split(' ')) {
    // Start with full word
    String remaining = word;
    while (remaining.isNotEmpty) {
      // Find longest subword in vocab
      String? match = _findLongestMatch(remaining);
      if (match != null) {
        tokens.add(match);
        remaining = remaining.substring(match.length);
      } else {
        tokens.add('[UNK]');
        break;
      }
    }
  }
  return tokens;
}
```

### 2. GPT-2 BPE in Dart

BPE (Byte-Pair Encoding) is more complex than WordPiece:

```dart
List<int> encode(String text) {
  // 1. Split into characters
  List<String> chars = text.split('');

  // 2. Apply learned merges
  while (true) {
    String? bestMerge = _findBestMerge(chars);
    if (bestMerge == null) break;
    chars = _applyMerge(chars, bestMerge);
  }

  // 3. Map to token IDs
  return chars.map((c) => vocab[c]!).toList();
}
```

### 3. Mean Pooling for Embeddings

BERT outputs embeddings for each token. We need one vector:

```dart
List<double> _meanPooling(List<List<List<double>>> output) {
  // output shape: [batch=1, sequence_length, hidden_size=384]
  final sequence = output[0];
  final hiddenSize = sequence[0].length;

  List<double> pooled = List.filled(hiddenSize, 0.0);

  for (final token in sequence) {
    for (int i = 0; i < hiddenSize; i++) {
      pooled[i] += token[i];
    }
  }

  // Average across sequence
  for (int i = 0; i < hiddenSize; i++) {
    pooled[i] /= sequence.length;
  }

  return pooled;
}
```

## Memory Management

Mobile apps have strict memory limits:

**Embeddings:**
- Model: ~22MB
- Vocab: ~200KB
- Activations: ~20MB
- **Total:** ~45MB

**Text Generation:**
- Model: ~130MB
- KV Cache: ~80MB
- Activations: ~10MB
- **Total:** ~220MB

Both fit comfortably within typical app budgets (< 300MB).

## Battery Considerations

ML inference drains battery. Optimizations:

**1. Run inference on background thread**

```dart
final embedding = await compute(getEmbeddingIsolate, text);
```

Prevents UI thread blocking and distributes CPU load.

**2. Cache results**

```dart
final cache = <String, List<double>>{};

Future<List<double>> getEmbedding(String text) async {
  if (cache.containsKey(text)) {
    return cache[text]!;
  }
  final embedding = await _runInference(text);
  cache[text] = embedding;
  return embedding;
}
```

**3. Batch operations**

Process multiple texts in one session load:

```dart
final embeddings = await Future.wait([
  getEmbedding(text1),
  getEmbedding(text2),
  getEmbedding(text3),
]);
```

Amortizes model loading cost.

## Use Cases Unlocked

### On-Device Semantic Search

Build a notes app with semantic search:

```dart
// User searches: "machine learning concepts"
final queryEmbedding = await getEmbedding(query);

// Find similar notes
final results = notes.map((note) {
  final similarity = cosineSimilarity(
    queryEmbedding,
    note.embedding
  );
  return (note, similarity);
}).toList()..sort((a, b) => b.$2.compareTo(a.$2));

// Return top 10
return results.take(10);
```

No internet required. Complete privacy.

### On-Device Text Generation

Build a writing assistant:

```dart
// User prompt: "Write a summary of this article"
final summary = await textGenerator.generate(
  prompt,
  GenerationConfig(
    maxNewTokens: 50,
    temperature: 0.7,
    topK: 40
  )
);

// Display to user
```

Zero API costs. Works offline.

### Privacy-First Applications

Medical apps, financial tools, personal journals—anything where data privacy matters:

- No data sent to servers
- No internet connection needed
- Processing happens on-device
- User data never leaves phone

## Limitations and Trade-offs

### Model Size vs. Quality

- **SmolLM2-135M:** Fast but limited reasoning
- **Llama-2-1B:** Better quality, 8x larger, too slow for mobile
- **GPT-3:** Impossible to run on-device

**Reality:** Mobile AI means smaller, specialized models.

### Context Windows

- **all-MiniLM-L6-v2:** 512 tokens max
- **SmolLM2:** 2048 tokens max

Long documents need chunking.

### Language Support

Models trained primarily on English:

- English: ✅ Excellent
- Spanish/French: ⚠️ Decent
- Chinese/Japanese: ❌ Poor

**Solution:** Use multilingual models (larger) or language-specific models.

### Battery Drain

Continuous ML inference drains battery. Not suitable for:
- Real-time video processing
- Continuous background monitoring
- Always-on assistants

**Best for:** User-initiated actions (button clicks, explicit requests).

## Lessons Learned

### 1. INT8 Quantization is Mandatory

Not just for size—iOS compatibility requires it. Always use quantized models for mobile.

### 2. KV Cache is Critical

For text generation, KV cache provides 3x speedup. Implement it from day one.

### 3. Tokenizers Must Be Implemented

No ready-made Dart tokenizers exist. Budget time for implementing WordPiece or BPE.

### 4. Test on Real Devices

Emulators don't reveal performance issues. Always test on physical iOS and Android devices.

### 5. Background Threads Prevent Freezing

Run inference in isolates to keep UI responsive.

## The Path Forward

Mobile AI is now practical for specific use cases:

**Works well:**
- Text similarity checks
- Semantic search over local data
- Short-form text generation
- Classification and categorization

**Doesn't work (yet):**
- Long-form generation (novels, reports)
- Complex reasoning (math, logic puzzles)
- Real-time video processing
- Multilingual understanding

As models get more efficient (better quantization, distillation, pruning) and phones get faster, the boundary shifts.

## Try It Yourself

**For embeddings:**
1. Download INT8 all-MiniLM-L6-v2 ONNX model
2. Add flutter_onnxruntime dependency
3. Implement BERT tokenizer in Dart
4. Run inference, extract embeddings
5. Calculate cosine similarity

**For text generation:**
1. Download INT8 SmolLM2-135M ONNX model
2. Implement GPT-2 BPE tokenizer
3. Set up KV cache handling
4. Implement sampling (temperature, TopK)
5. Generate text token by token

**Resources:**
- Models: Hugging Face ONNX exports
- ONNX Runtime: flutter_onnxruntime package
- Tokenizers: Implement based on transformers library logic

## Conclusion

Running real AI on mobile devices is no longer theoretical. With the right optimizations:

**INT8 quantization** → 4x size reduction, 4x speed improvement
**KV cache** → 3x generation speedup
**Native ONNX Runtime** → Production-ready inference

We built:
- **Semantic similarity** in < 200ms on mobile
- **Text generation** in ~3s for short outputs
- **Complete privacy** (zero data leaves device)
- **Works offline** (no internet required)

The technology is ready. The constraints are manageable. On-device AI is here.

**Read more:**
- [Browser-based AI feasibility](/posts/browser-based-ai-feasibility)
- [Privacy-first solutions](/posts/frank-bookmark-evolution)

This is the beginning of Frank Lab AI's mobile research. Expect more experiments, more optimizations, and more discoveries.

The future of AI is local.
