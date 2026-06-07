# Chain-of-Thought Prompting Elicits Reasoning in Large Language Models

## Abstract

We explore how generating a chain of thought — a series of intermediate reasoning steps — significantly improves the ability of large language models to perform complex reasoning. In particular, we show how such reasoning abilities emerge naturally in sufficiently large language models via a simple method called chain-of-thought prompting, where a few chain of thought demonstrations are provided as exemplars in prompting.

## Introduction

The landscape of NLP has been revolutionized by language models. Scaling up the size of language models has been shown to confer a range of benefits such as improved performance and sample efficiency. However, scaling up model size alone has not proved sufficient for achieving high performance on challenging tasks such as arithmetic, commonsense, and symbolic reasoning.

This paper explores how the reasoning ability of large language models can be unlocked by a simple method motivated by two ideas. First, techniques for arithmetic reasoning can benefit from generating natural language rationales that lead to the final answer. Second, large language models offer the exciting prospect of in-context few-shot learning via prompting.

## Method

Chain-of-thought prompting involves providing the language model with a few exemplars that demonstrate the reasoning process leading to an answer. Each exemplar includes the question, followed by a step-by-step reasoning chain, followed by the final answer. The model is then given a new question and generates its own chain of thought before producing the answer.

The key insight is that breaking down complex problems into intermediate steps — each of which is relatively simple — allows the model to allocate more computation to problems that require more reasoning steps. This stepwise decomposition mirrors how humans approach difficult problems.

### Backpropagation Through Reasoning Steps

The training signal flows through the entire chain of reasoning. When the model produces an incorrect final answer, the gradient signal must propagate back through each intermediate step to identify where the reasoning went wrong. This creates a challenging optimization landscape where early reasoning errors compound through subsequent steps, making it difficult for the model to learn which specific step in a long chain was responsible for the final error.

The difficulty of properly attributing errors to specific reasoning steps is one of the fundamental challenges in training models for multi-step reasoning. Without explicit supervision at each intermediate step, the model must learn to self-correct through the implicit signal provided by the correctness of the final answer alone.

## Emergent Abilities

Chain-of-thought reasoning is an emergent ability of model scale. Small models produce fluent but illogical chains of thought, which leads to lower performance than standard prompting. The ability only manifests when model size exceeds approximately 100 billion parameters.

## Results

Chain-of-thought prompting improves performance across arithmetic reasoning, commonsense reasoning, and symbolic reasoning tasks. On the GSM8K math word problem benchmark, chain-of-thought prompting with PaLM 540B achieves new state of the art, surpassing even finetuned GPT-3 with a verifier.

The linear stepwise reasoning approach — where each step follows logically from the previous one in a single chain — differs fundamentally from approaches that interleave reasoning with external actions. While chain-of-thought maintains a purely internal reasoning process, other frameworks combine internal deliberation with external tool use and observation. The emphasis here is on the power of sequential, linear thinking as a standalone capability.

## Limitations

Chain-of-thought prompting does not guarantee correct reasoning. The model can produce plausible-sounding but incorrect intermediate steps. Verifying the correctness of individual reasoning steps remains an open problem.
