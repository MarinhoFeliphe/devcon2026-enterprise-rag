# Enterprise RAG in Liferay AI Hub: Grounded, Governed, Permission-Aware

Resources for the Liferay DEVCON 2026 London session **"Enterprise RAG in Liferay AI Hub: Grounded, Governed, Permission-Aware"** by José Abelenda and Feliphe Marinho.

- **Video:** [ENTERPRISE RAG IN LIFERAY AI HUB: GROUNDED, GOVERNED, PERMISSION-AWARE | Liferay Devcon 2026 London](https://www.youtube.com/watch?v=vqi1ZZelJME) (Liferay on YouTube)
- **Slides:** [Enterprise RAG in Liferay AI Hub: Grounded, Governed and Permission Aware (PDF)](slides/enterprise-rag-in-liferay-ai-hub-grounded-governed-and-permission-aware.pdf)

> [!IMPORTANT]
> **This is a reference repository, not a runnable project.** It is not meant to be compiled, built, deployed, or run as it is. The files show what the resources of the demo look like, so you can read them alongside the talk. They were extracted from the Liferay Portal source tree where the demo was built, and they only work inside AI Hub.

## Contents

- [Talk Outline](#talk-outline)
- [Repository Layout](#repository-layout)
- [The Demo Agent: Objects Docs Assistant](#the-demo-agent-objects-docs-assistant)
  - [Grounded Answers](#grounded-answers)
  - [Follow-up Questions: the Compressing Query Transformer](#follow-up-questions-the-compressing-query-transformer)
- [What Is Not in This Repository](#what-is-not-in-this-repository)
- [Commit History](#commit-history)

## Talk Outline

RAG has two stages: **indexing** prepares the content (splitting, embedding, and storing it in a searchable vector index), and **retrieval** finds the exact context needed to answer the user's prompt.

| Topic | What the Slides Cover | Resources in This Repository |
| --- | --- | --- |
| Indexing: data sources | **Websites**: define what to crawl and what to skip; each data source has its own index. **Liferay DXP**: connect AI Hub to a DXP instance, and searches run on behalf of the user who asked. | — (see [What Is Not in This Repository](#what-is-not-in-this-repository)) |
| Retrieval augmentor pipeline | Query transformer → query router → content retrievers → content aggregator → content injector | — |
| Query transformer (pre-retrieval) | **Compressing**, **Expanding** (n paraphrases, 3 by default), **Expanding Keyword** (keywords for full-text search), and **Chaining** (combine transformers) | **Compressing**, used by the demo agent |
| Query router (pre-retrieval) | **Propagating** (default: every query to every retriever) and **Query Router Agent** (work in progress: routes by retriever description) | — |
| Content retrievers | **Website**: semantic search over the crawled index, one index per data source, scoped by account. **Liferay Search**: calls the DXP Search REST API as the current user, optionally with an SXP Blueprint. Retrieve operations are auditable. | The demo agent answers from the indexed Liferay Learn documentation |
| Content aggregator (post-retrieval) | Deduplicates and re-ranks with Reciprocal Rank Fusion: `RRF(d) = Σ 1 / (k + rank)`, `k = 60` | — |
| Content injector (post-retrieval) | Injects the contents into the prompt; with no relevant content it stops instead of letting the LLM guess | The demo agent's prompt adds the same rule: answer only from retrieved documentation |

The talk maps these to its title:

- **Grounded:** the content injector and the prompt keep answers on retrieved content.
- **Governed:** retrieve operations are audited, and website indexes are isolated per data source and restricted by account.
- **Permission aware:** the Liferay Search retriever runs on behalf of the user who started the request.

## Repository Layout

Everything the demo needs lives on the **AI Hub side**, so this repository has an `ai-hub-side/` folder only. It mirrors the layout of the companion repository for the multi-agents session, where a `client-side/` folder holds the client extensions.

```text
ai-hub-side/
└── site-initializer/
    ├── object-entries/
    │   └── agent-definition-object-entries.json       # the agent definition (agent card)
    └── workflow-definitions/
        └── objects-docs-assistant-workflow-definition/
            ├── workflow-definition.json                # workflow metadata (ERC, name, scope)
            └── workflow-definition.xml                 # the LLM node: prompt, RAG configuration
```

| Folder | Where It Lives in the Liferay Portal Source |
| --- | --- |
| `ai-hub-side/site-initializer/` | `workspaces/liferay-aihub-workspace/modules/dxp/apps/ai-hub/ai-hub-site-initializer/src/main/resources/site-initializer/` |

## The Demo Agent: Objects Docs Assistant

| Agent | ERC | Inputs | Output |
| --- | --- | --- | --- |
| [Objects Docs Assistant](ai-hub-side/site-initializer/workflow-definitions/objects-docs-assistant-workflow-definition/workflow-definition.xml) | `L_OBJECTS_DOCS_ASSISTANT` | request | response |

The assistant answers questions about Liferay Objects from the Liferay Learn documentation that AI Hub has indexed. It has two parts:

- **Agent definition (agent card):** [`agent-definition-object-entries.json`](ai-hub-side/site-initializer/object-entries/agent-definition-object-entries.json). It holds the agent's ERC, title, description, input and output variables, and the name of the workflow that runs it. The AI Hub site initializer has more agents in this file, so this repository includes only the demo agent.
- **Agent workflow:** [`workflow-definition.xml`](ai-hub-side/site-initializer/workflow-definitions/objects-docs-assistant-workflow-definition/workflow-definition.xml). A single **LLM** node holds the prompt, the input and output variables, and the RAG configuration.

```text
start ─► Objects Docs Assistant (LLM node + RAG) ─► end
```

### Grounded Answers

The prompt keeps the answers grounded in the retrieved documentation:

> You are the Liferay Objects Docs Assistant. Answer questions about Liferay Objects using only the retrieved Liferay Learn documentation and the chat history. Cite the documentation URL for each answer. If the documentation does not cover the question, say so. Your tone is professional, helpful, and concise.

- **Only retrieved content:** the agent answers from the retrieved documentation and the chat history, not from what the model already knows.
- **Citations:** each answer cites the documentation URL it came from, so the reader can check the source.
- **No guessing:** when the documentation does not cover the question, the agent says so.

### Follow-up Questions: the Compressing Query Transformer

The LLM node carries a `rag` configuration:

```json
{
	"queryTransformer": {
		"key": "compressing"
	}
}
```

A follow-up question such as *"How do I add a validation to it?"* does not say what "it" is, so searching with it alone finds the wrong documentation. The **compressing** query transformer runs before retrieval. It combines the chat history and the latest question into a single standalone query, for example a query about adding a validation to the object discussed earlier in the chat. Retrieval then searches with that query instead of the raw question.

## What Is Not in This Repository

The demo environment also included work on AI Hub itself (the website data source and auditing parts of the talk), which is not published here. It is described below so the talk is easier to follow.

- **Crawling the documentation:** the AI Hub crawler builds its configuration from the crawl targets set in AI Hub. A target sets the domain, its seed URLs, and include and exclude path patterns, along with limits on crawl depth, duration, links per page, and unique URLs. The crawled pages are indexed for retrieval.
- **Auditing retrieval (governed):** the audit message AI Hub records when it retrieves content for RAG now includes the retrieval query. With the compressing transformer, that is the standalone query that was actually searched.

## Commit History

| Commit | Adds |
| --- | --- |
| **DEMO** - ai-hub-side | The Objects Docs Assistant agent definition and its LLM workflow with the compressing query transformer |
| **DOCS** | This README |
| **DOCS** | The slides and the talk outline |

```bash
git log --reverse
```
