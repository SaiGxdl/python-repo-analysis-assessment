# Part 3: Prompt Preparation

## Task 3.1: Comprehensive Prompt Documentation

**Chosen PR:** #1224 — Fix the potential duplicate embeddings in the RAG module  
**Repository:** FoundationAgents/MetaGPT  

---

### 3.1.1 Repository Context

MetaGPT is an entirely Python-based framework that essentially acts as a virtual software company in a box. Instead of just treating a Large Language Model as a chatbot, MetaGPT gives it structure. If you give the system a simple, one-line instruction like "build a basic 2048 web game", it doesn't just spit out a massive block of code. Instead, it spins up a whole team of specialized AI agents. You get a virtual product manager who writes the requirements, an architect who designs the system, an engineer who writes the actual code, and a QA tester who reviews it. They talk to each other using strict Standard Operating Procedures (SOPs). This assembly-line approach is huge because it stops the AI from wandering off-topic or hallucinating uncontrollably.

A really vital piece of this framework is its RAG module (Retrieval-Augmented Generation). Rather than forcing agents to rely purely on their training data, RAG lets them read specific, external documents right before they answer a question or write code. For instance, the "Data Interpreter" agent relies heavily on this to pull in domain-specific context when crunching numbers or analyzing datasets.

The people mostly using MetaGPT are AI developers, researchers, and automation engineers. These are folks looking to automate complex software engineering tasks or experiment with how multiple AI agents collaborate to solve problems. They generally know their way around Python, understand how to hook up different LLM providers like OpenAI or AWS Bedrock, and are comfortable tinkering with vector databases to get the most out of the system.

---

### 3.1.2 Pull Request Description

This pull request addresses a rather frustrating bug inside the RAG module that was causing documents to be embedded twice during the setup phase. When a developer called `SimpleEngine.from_docs()` and passed in specific `retriever_configs`, the old code would fire off two separate index builds. First, it would take all the documents and build a `VectorStoreIndex` from scratch, which meant making a costly API call to embed every single file. Then, it would pass the configuration over to the `get_retriever()` method. Unfortunately, `get_retriever()` wasn't smart enough to realize the work was already done, so it just rebuilt the whole index again, triggering a second round of embedding API calls. If you were feeding 1,000 documents into the system, you ended up paying for 2,000 embeddings and waiting twice as long.

The fix completely changes how the system handles the document index. Instead of forcing `SimpleEngine` to hold onto a rigid `VectorStoreIndex` object right away, the PR refactors it to use a "transformation pipeline." Basically, the engine now processes the raw documents into intermediate nodes first. On top of that, the developer added a clever decorator to the `RetrieverFactory`. Now, when the factory is asked to build a retriever, it stops and checks the engine's current state. If it sees that a valid index has already been built, it just reuses it. If not, it builds a new one. As a nice little bonus, they also fixed a minor annoyance in the base factory configuration where missing keys used to crash the program with a `KeyError`; now they just safely return `None`.

---

### 3.1.3 Acceptance Criteria

✓ When `SimpleEngine.from_docs()` is executed with both `input_files` and `retriever_configs` provided, the system should trigger the embedding API exactly once per document, avoiding duplicate calls.  
✓ When a valid vector index already exists in the engine's state, the system should reuse that existing index directly upon calling `get_retriever()` instead of triggering a new index build.  
✓ When users call `SimpleEngine.from_docs()` without any `retriever_configs` (the default use case), the system should behave identically to the pre-PR state, ensuring absolutely no regression in existing functionality.  
✓ The implementation should handle scenarios where `_val_from_config_or_kwargs` encounters a missing configuration key by returning `None` instead of throwing a `KeyError` exception.  
✓ When the engine is constructed via the newly introduced `_from_nodes` method, it should produce a fully functional RAG engine that retrieves context and answers user queries with the exact same accuracy as the old approach.  
✓ The implementation should handle multiple different `retriever_configs` (such as FAISS alongside BM25) by correctly routing the index creation and reuse logic for each specific retriever type without mixing them up.  

---

### 3.1.4 Edge Cases

1. **Empty Document Lists:** If the provided `input_files` or `input_dir` resolves to absolutely zero documents, the transformation pipeline will have nothing to process. The system must not crash or blindly call the embedding API with empty payloads. It needs to handle this gracefully, either returning an empty engine or raising a clear error before making any network calls.
2. **Repeated Factory Calls on the Same Instance:** A user might call `get_retriever()` multiple times on the exact same engine instance if they are experimenting with different retriever configurations sequentially. The index reuse logic needs to be persistent enough to catch subsequent calls, not just the very first one, to ensure the bug doesn't reappear later in the session.
3. **Explicit None vs. Missing Key:** Since the fix changes missing config keys to return `None`, we have to be careful about users who explicitly pass `None` as a value. Downstream code needs to gracefully handle both missing and explicit `None` values without silently failing or assuming a required value is present.
4. **Mismatched Retriever Types:** If a developer passes configurations for different types of retrievers at the same time—like a vector-based FAISS retriever and a text-based BM25 retriever—the dynamic index handler must be smart enough to know it can't reuse a FAISS index for a BM25 operation.

---

### 3.1.5 Initial Prompt

You are tasked with contributing to the MetaGPT repository, a highly popular Python framework that simulates a multi-agent software company. Your goal is to implement the exact fixes proposed in Pull Request #1224. This PR resolves a critical bug inside the RAG (Retrieval-Augmented Generation) module that accidentally embeds documents twice when `SimpleEngine.from_docs()` is called alongside `retriever_configs`.

To give you some background, MetaGPT's RAG module is located in the [`metagpt/rag/`](https://github.com/FoundationAgents/MetaGPT/tree/main/metagpt/rag/) directory. The main workhorse is `SimpleEngine` inside [`metagpt/rag/engines/simple.py`](https://github.com/FoundationAgents/MetaGPT/blob/main/metagpt/rag/engines/simple.py). It takes documents, embeds them, and lets agents query them. The `RetrieverFactory` ([`metagpt/rag/factories/retriever.py`](https://github.com/FoundationAgents/MetaGPT/blob/main/metagpt/rag/factories/retriever.py)) creates the actual retriever objects, while `ConfigBasedFactory` ([`metagpt/rag/factories/base.py`](https://github.com/FoundationAgents/MetaGPT/blob/main/metagpt/rag/factories/base.py)) handles the setup configs.

Right now, if someone calls `SimpleEngine.from_docs(..., retriever_configs=[FAISSRetrieverConfig()])`, the code embeds all documents to build a `VectorStoreIndex`. Then, it hands the config off to `get_retriever()`, which foolishly builds a brand new index from scratch. This means the user pays double the API costs and waits twice as long. 

Here is exactly what you need to do:

1. **Refactor SimpleEngine:** Open [`metagpt/rag/engines/simple.py`](https://github.com/FoundationAgents/MetaGPT/blob/main/metagpt/rag/engines/simple.py). You need to rip out the direct dependency on holding a `VectorStoreIndex`. Instead, implement a transformation-based architecture. Add a new `_from_nodes` method that builds the engine from pre-processed document nodes using a transformation pipeline. All the existing methods in this class need to be updated to work with this pipeline rather than a direct index.

2. **Update the RetrieverFactory:** Open [`metagpt/rag/factories/retriever.py`](https://github.com/FoundationAgents/MetaGPT/blob/main/metagpt/rag/factories/retriever.py). Introduce decorator-based logic to handle the indices dynamically. Before `get_retriever()` tries to build an index, it must check if a valid one already exists in the engine's state. If it finds one, it should immediately return it. If not, it builds a new one. 

3. **Fix the Factory Config Base:** In [`metagpt/rag/factories/base.py`](https://github.com/FoundationAgents/MetaGPT/blob/main/metagpt/rag/factories/base.py), modify `_val_from_config_or_kwargs`. If it can't find a key, make it return `None` instead of crashing with a `KeyError`.

Your implementation must meet these acceptance criteria:
- The embedding API must only be called exactly once per document.
- Calling `get_retriever()` on an engine that already has a valid index must never trigger a new embedding call.
- The default path (calling without configs) must not break.
- Missing keys must safely return `None`.

Be highly aware of a few edge cases. First, if an empty document list is passed, do not call the API; fail gracefully. Second, if `get_retriever()` is called multiple times on the same engine, your reuse logic must catch every single subsequent call. Finally, make sure that if multiple retriever types (like FAISS and BM25) are mixed, the system doesn't try to reuse an incompatible index type.

Make sure to update the test files in [`tests/metagpt/rag/`](https://github.com/FoundationAgents/MetaGPT/tree/main/tests/metagpt/rag/) to reflect the transformation pipeline changes and ensure everything passes perfectly!
