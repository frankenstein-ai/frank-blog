+++
date = '2026-02-02T12:00:00-03:00'
draft = false
title = 'Running AI Models on Mobile: ONNX, Flutter, and the Quest for On-Device Intelligence'
+++

## The mobile AI challenge

Can you run real AI on a phone? Not through API calls to OpenAI, but actually on the device: embeddings, semantic search, text generation.

The answer is yes, but you have to deal with serious constraints. App stores reject 200MB apps. Phones have 4-8GB RAM shared with the OS and other apps. ML inference drains batteries fast. And mobile CPUs are slower than desktop GPUs.

I experimented with running transformer models on iOS and Android using ONNX Runtime and Flutter. Here's what I found.

## Two experiments: embeddings and generation

### Experiment 1: embeddings for semantic similarity

The goal was to run all-MiniLM-L6-v2 for text embeddings on mobile, so I could check whether two sentences are similar without internet. The challenge: the model is 86MB in FP32, too large for mobile apps.

### Experiment 2: text generation

The goal here was to run SmolLM2-135M for autoregressive text generation, generating poems, summaries, responses entirely on-device. Text generation is computationally expensive and requires managing state (KV cache), which made this the harder of the two.

Both experiments succeeded, but required specific optimizations.

## The quantization solution

### The problem: FP32 models are too large

Standard transformer models use FP32 (32-bit floating point) weights:

- all-MiniLM-L6-v2: 86MB
- SmolLM2-135M: 540MB

These sizes are unacceptable for mobile apps.

### The solution: INT8 quantization

Quantization converts FP32 weights to INT8 (8-bit integers):

```
FP32: -3.14159... → INT8: -127 to 127 (mapped)
```

The impact is significant. Size drops by 4x: 86MB becomes 22MB for embeddings, 540MB becomes 135MB for generation. Speed improves by roughly 4x too, since INT8 math is faster on ARM CPUs. And on iOS, ONNX Runtime 1.22.0 actually requires INT8 for many ops.

Quality loss is minimal for text tasks (unlike images where quantization causes visible artifacts).

## Experiment 1: mobile embeddings with ONNX

### Setup

- Framework: Flutter (Dart)
- Runtime: flutter_onnxruntime (ONNX Runtime 1.22.0)
- Model: all-MiniLM-L6-v2 (INT8 quantized, 22MB)
- Tokenizer: BERT WordPiece (Dart implementation)

### Implementation

First, load the model:

```dart
final session = await OrtSession.fromAsset(
  'assets/models/model_quantized.onnx'
);
```

Model loading takes about 1.2s (one-time cost).

Then tokenize the input:

```dart
class BERTTokenizer {
  final Map<String, int> vocab; // 30k tokens

  List<int> encode(String text) {
    final tokens = _wordPiece(text.toLowerCase());
    return tokens.map((t) => vocab[t] ?? vocab['[UNK]']!).toList();
  }
}
```

Run inference:

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

And calculate similarity:

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

Sub-200ms is fast enough for real-time similarity checks.

### Quality

I tested various text pairs:

- "dog" vs "puppy": **0.89** (High similarity)
- "computer" vs "machine": **0.75** (Related concepts)
- "apple" vs "orange": **0.60** (Different but same category)

Embeddings match human intuition for English text.

### iOS compatibility issues

One critical finding: FP32 models fail on iOS ONNX Runtime. The error is "Protobuf parsing failed," and it happens because iOS ONNX Runtime 1.22.0 has stricter compatibility requirements. The solution is to use INT8 quantized models specifically. This made quantization mandatory, not optional, for iOS support.

## Experiment 2: text generation on mobile

### Setup

- Framework: Flutter (Dart)
- Runtime: flutter_onnxruntime
- Model: SmolLM2-135M-Instruct (INT8, 130MB)
- Tokenizer: GPT-2 BPE (Byte-Pair Encoding)
- Optimization: KV Cache (critical for performance)

### The KV cache difference

Text generation without KV cache is unusable on mobile.

Without it, every generation step re-computes attention for the entire sequence. Token 1 computes attention for 10 prompt tokens, token 2 for 11, token 3 for 12, and so on up to token 20 computing attention for 30 tokens. That costs about 500ms per token, or 10 seconds for 20 tokens.

With KV cache, the first step computes attention for all prompt tokens and saves the Key-Value pairs. Subsequent steps only compute attention for the new token and reuse the cached KV pairs. That brings it down to about 150ms per token, or 3 seconds for 20 tokens. A 3.3x speedup.

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

### Sampling: temperature and TopK

Greedy decoding (always pick highest probability) produces repetitive text:

```
"I like cats. I like cats. I like cats."
```

Temperature adds randomness: 0.0 is greedy and deterministic, 0.8 is balanced and more human-like, 1.5 gets creative but chaotic.

TopK limits sampling to the top K tokens. With a vocabulary of about 50,000 tokens, setting TopK=40 means only sampling from the 40 most likely, which filters out nonsense punctuation and random symbols.

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

Three seconds for a short poem is acceptable for mobile UX.

### Output quality

I tested various prompts.

Prompt: "Write a haiku about coding"

Output:
```
Code flows through my mind
Algorithms dance and play
Logic becomes art
```

Valid 5-7-5 structure.

Prompt: "What is machine learning?"

Output:
```
Machine learning is a type of artificial intelligence
that allows computers to learn from data without being
explicitly programmed...
```

Coherent explanation.

SmolLM2-135M handles general tasks well enough. INT8 quantization doesn't degrade quality noticeably.

## Technical challenges solved

### BERT WordPiece tokenization in Dart

No pre-built Dart tokenizer existed, so I implemented one:

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

### GPT-2 BPE in Dart

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

### Mean pooling for embeddings

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

## Memory management

Mobile apps have strict memory limits.

For embeddings, the breakdown is roughly 22MB for the model, 200KB for the vocab, and 20MB for activations, totaling about 45MB. For text generation, it's about 130MB for the model, 80MB for the KV cache, and 10MB for activations, totaling about 220MB. Both fit within typical app budgets (under 300MB).

## Battery considerations

ML inference drains battery. A few optimizations help.

Running inference on a background thread prevents UI blocking and distributes CPU load:

```dart
final embedding = await compute(getEmbeddingIsolate, text);
```

Caching results avoids redundant computation:

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

And batching operations amortizes model loading cost:

```dart
final embeddings = await Future.wait([
  getEmbedding(text1),
  getEmbedding(text2),
  getEmbedding(text3),
]);
```

## Use cases unlocked

### On-device semantic search

You could build a notes app with semantic search:

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

### On-device text generation

Or a writing assistant:

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

### Privacy-first applications

Medical apps, financial tools, personal journals, anything where data privacy matters. No data sent to servers, no internet connection needed, processing happens on-device, and user data never leaves the phone.

## Limitations and trade-offs

### Model size vs. quality

SmolLM2-135M is fast but limited in reasoning. Llama-2-1B offers better quality but is 8x larger and too slow for mobile. GPT-3 is impossible to run on-device. The reality of mobile AI is smaller, specialized models.

### Context windows

all-MiniLM-L6-v2 maxes out at 512 tokens. SmolLM2 handles up to 2048. Long documents need chunking.

### Language support

These models were trained primarily on English. English works well. Spanish and French are decent. Chinese and Japanese perform poorly. Multilingual models (larger) or language-specific models would be needed for better coverage.

### Battery drain

Continuous ML inference drains battery. It's not suitable for real-time video processing, continuous background monitoring, or always-on assistants. It works best for user-initiated actions like button clicks and explicit requests.

## What I learned

INT8 quantization turned out to be mandatory, not just for reducing size but because iOS compatibility requires it. Always use quantized models for mobile.

KV cache is essential for text generation. It provides a 3x speedup and should be implemented from the start.

I also had to implement both tokenizers (WordPiece and BPE) in Dart from scratch since no ready-made options existed. Budget time for this if you're going down the same path.

Testing on real devices matters. Emulators don't reveal the performance issues you'll hit on actual hardware. And always run inference in isolates to keep the UI responsive.

## The path forward

Mobile AI is practical for specific use cases right now. Text similarity checks, semantic search over local data, short-form text generation, and classification all work well. Long-form generation, complex reasoning, real-time video processing, and multilingual understanding are still out of reach.

As models get more efficient through better quantization, distillation, and pruning, and as phones get faster, the boundary will keep shifting.

## Try it yourself

For embeddings: download an INT8 all-MiniLM-L6-v2 ONNX model, add the flutter_onnxruntime dependency, implement a BERT tokenizer in Dart, run inference, extract embeddings, and calculate cosine similarity.

For text generation: download an INT8 SmolLM2-135M ONNX model, implement a GPT-2 BPE tokenizer, set up KV cache handling, implement sampling (temperature, TopK), and generate text token by token.

**Resources:**
- Models: Hugging Face ONNX exports
- ONNX Runtime: flutter_onnxruntime package
- Tokenizers: Implement based on transformers library logic

## Wrapping up

I got real AI running on mobile devices with the right optimizations: INT8 quantization for 4x size reduction and 4x speed improvement, KV cache for 3x generation speedup, and native ONNX Runtime for inference.

The results: semantic similarity in under 200ms on mobile, text generation in about 3 seconds for short outputs, complete privacy with zero data leaving the device, and full offline support.

**Read more:**
- [Browser-based AI feasibility](/posts/browser-based-ai-feasibility)
- [Privacy-first solutions](/posts/frank-bookmark-evolution)

This is the beginning of Frank Lab AI's mobile research.
