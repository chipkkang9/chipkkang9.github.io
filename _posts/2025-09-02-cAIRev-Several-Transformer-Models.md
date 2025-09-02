---
layout: post
title: '[cAIRev] Several Transformer Models'
description: Several transformer models descriptions focus on its architecture
date: 2025-09-02 10:50:00 +09:00
categories: [chipkkang9's AI Reversing]
tags: [AI, Model, Transformer]
---

# Transformer Architecture
Transformer brought a remarkable improvement to AI studies.

It consists of Encoder and Decoder, known as its very first paper "Attention Is All You Need", but researchers could develop AI models by using only Encoder or only Decoder, and even both of them.

In this article, we'll gonna discuss the pros and cons of each Transformer-based model and the tasks each model fits. Let's look into these several Transformer Architectures.

## Encoder-Only Transformer
![](/assets/img/posts/Encoder-only%20Transformer.png)
_Encoder Part of Transformer_

Encoder-Only Transformer is also known as "auto-encoding" model. 

Encoder-Only Transformer models use only Encoder part of the Transformer architecture. It `bi-directionally` conducts attention about all the input contexts, and is specialized to generate advanced semantic representation information of each word.

### Pretraining
To pretrain Encoder-Only Transformer, in the pretraining process of the model, learning proceeds through the process of damagin a given inital sentence using various methods(e.g. word masking) and restoring the damaged sentence to the original sentence.

### Tasks
Encoder model fits for the tasks that require understand about whole sentence, like sentence classification, named-entity recognition, and more generally word classification or extractive question answering.

### Example Models

- BERT
- DistilBERT
- RoBERTa

<br>

## Decoder-Only Transformer
![](/assets/img/posts/Decoder-only%20Transformer.png)
_Decoder Part of Transformer_

Decoder-Only Transformer is also known as "auto-regressive" model.

Decoder-Only Transformer models use only Decoder part of the Transformer architecture. For each step, attention layer could only access former positioned words about currently processing word.

### Pretraining
To pretrain Decoder-Only Transformer, in the pretraining process of the model, learning proceeds through predicting next word of the sentence generally.

### Tasks
Decoder model fits for the tasks like text generation, summarization, translation, question answering, code generation, reasoning, few-shot learning

### Example Models

- Hugging Face SmolLM Series
- DeepSeek's V3
- Meta's Llama Series

<br>

## Encoder-Decoder Transformer
![](/assets/img/posts/Transformer%20Architecture.png)
_Encoder-Decoder Transformer_

Encoder-Decoder Transformer is also known as "sequence-to-sequence" model. It might be named the architecture is similar to seq2seq model.

Encoder-Decoder Transformer models use both parts of Transformer architecture. For each step, attention layer could access all words to inital sentence. However, the attention layer of Decoder part could only access former positioned words about currently processing word.

### Pretraining
To pretrain Encoder-Decoder Transformer, it might uses Encoder or Decoder model's objectives, but it requires more complex processing steps.

For example, `T5` model pretrains by replacing random spans of text to a mask special word, training objective is predicting correct special word that masked before. Especially, Encoder-Decoder Transformer based model pretraining method is not determined as only one, it is subject to be changed model by model.

### Example Models

| Application            |      Description      |  Example Model |
|:-----------------------|-----------------------|----------------|
|Machine Translation     |Converting text between languages | Marian, T5
|Text Summarization      |Creating concise summaries of longer texts | BART, T5
|Data-to-Text Generation |Converting structured data into natural language | T5 
|Grammar Correction      |Fixing grammartical errors in text | T5
|Question Answering      |Generating answers based on  context | BART, T5

<br>

## References.
> Huggingface. Transformer Architectures
: https://huggingface.co/learn/llm-course/en/chapter1/6