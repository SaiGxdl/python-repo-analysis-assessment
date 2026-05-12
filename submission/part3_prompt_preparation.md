# Part 3: Prompt Preparation

## Task 3.1: Comprehensive Prompt Documentation

**Chosen PR:** #1224 — Fix the potential duplicate embeddings in the RAG module  
**Repository:** FoundationAgents/MetaGPT  

---

### 3.1.1 Repository Context

MetaGPT is an all-python framework, which is akin to building a software company inside your computer. While most frameworks treat Large Language Model as a conversational bot, MetaGPT gives more structure. Instead of generating a monolith chunk of code after giving a one-liner instruction such as "build a basic 2048 web game", MetaGPT will spawn a team of AI agents, including a virtual product manager to write the specification, a systems architect to design the system, an engineer to write the code and a QA agent to review it. All these bots communicate following strict SOPs. Assembly line approach to programming is a very important part of MetaGPT since it prevents the AI from diverging into unrelated topics or hallucinations. One particularly important feature of the framework is a RAG (Retrieval Augmented Generation) module. In lieu of asking the agents to use only their trained model, MetaGPT enables reading certain documents before answering questions or writing a piece of code. For example, the "Data Interpreter" AI uses the module extensively when trying to analyze the information from the domain-specific datasets. The target users for MetaGPT are primarily AI developers, researchers and automation engineers. These are individuals who are interested in automating complicated software development projects or exploring how AI agents can work together to solve problems. They are usually quite proficient in using Python programming language and familiar with connecting various LLM providers, such as OpenAI or AWS Bedrock. They can also easily manipulate the vectors database to fully exploit the capabilities of the system.

---

### 3.1.2 Pull Request Description

In this pull request, we fix an annoying bug in the RAG module which caused document duplication while initializing indexes. If one used the function `SimpleEngine.from_docs()` along with passing certain `retriever_configs`, the previous version was doing two different index builds. First, all documents were indexed using `VectorStoreIndex.build_from()` function creating the initial empty `VectorStoreIndex`, which included the expensive step of embedding all files through the API. Then, it proceeded with sending the config further to the function `get_retriever()`. The problem is that the latter was unable to recognize that there's no need to build a full index again and repeated the process. As a result, 1,000 documents were embedded twice, leading to twice more expensive costs and doubled the initialization time.

The fix dramatically alters the indexing process flow inside the system. In particular, the PR changes the initialization of the `SimpleEngine`, which no longer needs to store its `VectorStoreIndex` instance statically from the start. Instead, the PR introduces a concept of "transformation pipeline," allowing the engine to convert the raw document files into intermediate nodes. Moreover, the developer introduces a convenient decorator for the `RetrieverFactory`, enabling it to pause the creation process and inspect the current state of the engine. In case the latter already has an adequate index built, it is reused without any further actions; otherwise, a new one is created from scratch. Finally, the developer fixes another minor inconvenience by handling the base factory configuration gracefully upon encountering a missing key, returning `None` instead of raising `KeyError`.

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
