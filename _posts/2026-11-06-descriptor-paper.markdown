---
layout: post
title:  "New paper: Task-Agnostic Web Document Annotation with LLM-Generated Descriptors"
date:   2026-11-06 15:00:00 +0200
categories: general
---

EMNLP2026 and the Web-as-Corpus Workshop are now over. What a great conference and workshop! It was a pleasure to meet so many colleagues from around the world and talk research. The WaC workshop was especially interesting as it brought together researchers with a shared interest in web corpora and their applications. I also got to present my paper *Task-Agnostic Web Document Annotation with LLM-Generated Descriptors* at the poster session and received much positive feedback. The paper is now published in the conference proceedings and is available [here](https://aclanthology.org/2026.wac-1.24/).

## TL;DR

We use an LLM to tag web documents with free-text descriptors that cover topic, style, format, quality and more, without targeting any particular research task. This lets researchers filter huge corpora for their own needs without training task-specific classifiers. Raw descriptors are judged accurate 88% of the time, and classifiers built on them perform on par with TF-IDF while being interpretable. Mapping descriptors to our shared 800k-descriptor schema lowers accuracy to 62%, which remains the main open problem. A 10B-token annotated FineWeb sample and the code are publicly available (links at the end of the post).

## Overview of the paper

The premise of the paper is simple: Modern web corpora are huge, but a typical research project only needs a small fraction of the documents. Filtering out irrelevant documents is a time-consuming task and can require annotating and training task-specific classifiers. This is not necessarily feasible for research projects that lack the computational resources, time or expertise to do so. To address this, we introduce *document descriptors*. A descriptor can look like this:

```text
car enthusiast; The document is written by a car enthusiast, indicating a strong interest in vehicles and their modification. The author's passion for cars is evident in their language and tone.
```

It has a *tag* (car enthusiast) and a *description* (The document is written by a car enthusiast...). The tag is a short, human-readable label while the description provides more context and specifies the tag.

In this paper, we propose a method for annotating web documents in a *task-agnostic* way with the help of an LLM. This means that the model generates descriptors for the documents that are not specific to any particular research task. The descriptors can then be used to filter the documents for any given research task, making it easier and faster for researchers to find the relevant documents. The (abbreviated) labelling pipeline looks like this:

1. A large language model (LLM) is prompted to generate descriptors for a given document. The model is prompted to "describe the document in aspects including, but not limited to, style, tone, genre, topic, domain, language, quality, sentiment, etc. Describe anything that you believe is essential for understanding, summarizing, or rewriting the document." No limit on the number of descriptors is set, but experimentally we find that the average document receives 15 descriptors.
2. After the initial descriptors are generated, the model is prompted to use the descriptors to try and rewrite the original document. The logic is simple: if the descriptors are accurate and adequate, the model should be able to recreate a text that is similar to the original document. Bad descriptors will lead to a poor rewrite.
3. The model is then prompted to compare the original document, its rewrite and the descriptors and to revise the descriptors so that the next rewrite is more similar to the original document. The rewrite and revision steps are repeated a total of three times. The final descriptors are chosen based on which of the rewrites is most similar to the original document. The similarity is measured using cosine similarity.
4. Labelling 500,000 FineWeb documents results in almost 7.5 million unique descriptors, many of which are semantically very similar. Therefore, we merge synonymous and near-synonymous descriptors, first by combining and modifying descriptors based on character-level identicality and then by using a semantic similarity model to merge descriptors that are semantically similar. The final set of descriptors contains 800k unique descriptors, which we call the descriptor schema.
5. To apply the schema to a new corpus, we first generate descriptors freely for the new documents, as in step 1. Then, for each new descriptor, we find the 10 most similar descriptors in the schema, prompt the LLM to choose the best match, and replace the new descriptor with the best match. This results in a set of harmonized descriptors for the new documents that are consistent with the schema.

## Results and findings

Evaluating the quality of the descriptors is not straightforward, as there is no ground truth or benchmark for this task. Therefore, we evaluate them using an LLM-as-judge approach and by comparing them to topic and format labels from [WebOrganizer](https://allenai.github.io/weborganizer-website/). We conduct all evaluations on a new, 100k-document sample of FineWeb that was not used in creating the descriptor schema. We evaluate both raw (unharmonized) descriptors, and harmonized (schema-mapped) descriptors.

Our first analysis measures accuracy: how well the descriptors describe the original document. The LLM-as-judge approach shows that raw descriptors are deemed to be accurate 88% of the time. However, the harmonization step reduces accuracy down to 62%. Still, 79% of harmonized descriptors are at least partially accurate, according to the judge model. I'll discuss this unfortunate drop in accuracy in the lessons learned section below.

Next, we evaluate what aspects of the document the descriptors capture. A truly universal schema cannot focus only on topic, but has to capture many other aspects as well. We find that while topical aspects are the most common, with 58% of descriptors, the descriptors also capture style and tone (20%), structure and format (10%), quality (5%) and other aspects, too. This shows that the descriptors can be used to filter documents based on many different aspects, not just topic.

We compare the descriptors to the topic and format labels from WebOrganizer in two ways. First, we label the 100k-document sample with the WebOrganizer classifiers and train two types of linear classifiers to predict the labels: one classifier is trained on TF-IDF features of the documents, while the other uses the descriptors as features. We find that descriptor-based classifiers peform on par with the TF-IDF-based classifiers with the added benefit of being much more interpretable. While TF-IDF features are just a bag of words, the descriptors are human readable and can thus be used to understand why a document was classified in a certain way.

Finally, we evaluate the descriptors in a retrieval setting. We use the descriptors to retrieve documents from the 100k-document sample. We use both topic and format labels as queries and a "sarcasm" query as an example of a document type that is hard to identify from surface features alone. Our results show that while both precision and recall are relatively low, the descriptors are able to retrieve relevant documents for queries. Raw descriptors are better in terms of precision, while harmonized descriptors achieve higher recall. This is likely because harmonization merges semantically similar descriptors, which increases the number of documents that can be retrieved for a given query. We propose that the descriptors can be used as a first step in filtering documents for a research task, after which more task-specific methods can be applied to further refine the results.

## Lessons learned

When we started this project, we did not know exactly what to expect and it took a lot longer than initially anticipated. We tried many different approaches and changed directions several times. Starting over from scratch turned into a joke in the office.

As an example of an unexpected challenge, we initially believed that an LLM would be able to keep track of the descriptors it had generated previously. However, we soon found out that as the number of new descriptors increased, the model would begin generating duplicates and semantically similar descriptors. This is why we introduced merging and harmonization as post-processing steps. Ideally, we could generate schema-consistent descriptors directly, but this is a problem that we could not solve within the scope of this project.

The harmonization step is especially problematic, as it reduces the accuracy of the descriptors quite drastically. One solution could be to make the harmonization more robust by including the full document context in the harmonization decision. This would allow the model to make more informed decisions but would increase the computational cost significantly. Another solution would be to train a classifier on our 800k-descriptors schema to directly predict schema-constrained descriptors. This is a research direction called *extreme classification*, which is a research direction that I have not yet explored, but one that could be a promising.

As for the next steps in this project, I already have a lot of ideas for Descriptors 2.0, such as

- sub-document level descriptors
- descriptor hierarchies
- adjustable granularity
- numerical salience scores for descriptors

The comments I received at the Web-as-Corpus poster session were very encouraging and I hope to continue working on this project in the future. For now, I will focus on a different project that I hope to publish soon. If you are interested in collaborating or have ideas regarding the use or development of the descriptors, please reach out!

## Artifacts and code
Research artifacts and a 10-billion-token descriptor-annotated sample of FineWeb are available on [HuggingFace](https://huggingface.co/datasets/TurkuNLP/WebDocumentDescriptors). The code for generating the descriptors is available on [GitHub](https://github.com/TurkuNLP/LLM_document_descriptors).