# Ask the Docs --- RAG Assistant

A small Retrieval-Augmented Generation (RAG) assistant that answers
questions using five public pages from the official Python
documentation. It retrieves relevant text chunks, asks an LLM to answer
from those excerpts, includes source references, and abstains when the
documents do not provide enough evidence.

## Project status

-   **Retrieval evaluation:** 10/10 checks passed (100%) on the small,
    hand-written test set described below.
-   **Answer-quality evaluation:** Not yet completed for all 10
    questions because the Gemini API free-tier daily request quota was
    exhausted during testing. API errors are not counted as incorrect
    model answers.
-   This is a prototype evaluation, not a claim of production-level
    accuracy.

## Knowledge sources

The notebook downloads these pages at runtime:

1.  [Python Tutorial](https://docs.python.org/3/tutorial/index.html)
2.  [Python Data
    Structures](https://docs.python.org/3/tutorial/datastructures.html)
3.  [Python Functions and Control
    Flow](https://docs.python.org/3/tutorial/controlflow.html)
4.  [Python Errors and
    Exceptions](https://docs.python.org/3/tutorial/errors.html)
5.  [Python Classes](https://docs.python.org/3/tutorial/classes.html)

Each source keeps its title and URL so answers can be traced back to the
original documentation.

## Approach

1.  **Load and clean documents:** Download the HTML pages and extract
    the main text with BeautifulSoup, removing navigation and other page
    elements where possible.
2.  **Chunking:** Split text into chunks of up to **180 words** with
    **40 words of overlap**. Overlap helps preserve context that might
    otherwise be split across chunk boundaries.
3.  **Embeddings:** Encode each chunk with
    `sentence-transformers/all-MiniLM-L6-v2`, producing
    **384-dimensional** embeddings. The embeddings are normalized.
4.  **Vector retrieval:** Store embeddings in a FAISS `IndexFlatIP`
    index. Because vectors are normalized, inner-product ranking is
    equivalent to cosine-similarity ranking. The assistant retrieves the
    top-k chunks for each question.
5.  **Evidence filtering and generation:** Use a configurable similarity
    threshold (default `0.35`) to filter retrieved chunks. Gemini is
    prompted to use only the supplied excerpts and to return `NOT_FOUND`
    when the excerpts do not support an answer.
6.  **Citations:** Each chunk carries a chunk ID, document title, and
    source URL. The answer function returns source metadata for the
    evidence supplied to the model. Citation quality should also be
    checked during answer-quality review.
7.  **Evaluation:** Run a retrieval-only check independently from LLM
    generation, so retrieval can be tested even when the LLM API quota
    is unavailable.

## Chunking and embedding choices

-   **180-word chunks:** A simple, readable baseline that limits the
    amount of text sent to the LLM.
-   **40-word overlap:** Reduces the chance that a useful definition or
    explanation is split across adjacent chunks.
-   **MiniLM embeddings:** A compact sentence-transformer model suitable
    for a small prototype and quick local embedding generation.
-   **Normalized vectors + FAISS inner product:** Makes similarity
    ranking efficient and straightforward for this small collection.
-   **Similarity threshold of 0.35:** An initial heuristic, not a
    calibrated confidence score. It should be tuned on a larger
    validation set and evaluated for both false retrievals and missed
    evidence.

## Evaluation results

### Retrieval evaluation

The hand-written test set contains nine questions expected to be
answerable from the Python documentation and one out-of-scope question:
"What is the capital of France?"

The retrieval check passed **10/10 (100%)** using the notebook's
criteria: - For each of the nine in-scope questions, the expected
document appeared among the top three retrieved chunks with a score at
or above `0.35`. - For the out-of-scope France question, the top
similarity score was below `0.35`.

This is a small, manually designed retrieval sanity check. It does
**not** establish that the generated answers are all correct.

### Answer-quality evaluation

The full 10-question answer-quality evaluation is **pending**. During
development, some Gemini requests succeeded, while others returned
`503 UNAVAILABLE` and `429 RESOURCE_EXHAUSTED`. The final 10-question
answer accuracy should be reported only after the model successfully
answers the test set and each response is reviewed against its
expected-answer criterion. Do not count quota failures as wrong answers.

## Observed failures and limitations

1.  **False abstention on a list question:** For "What is a Python
    list?", retrieval returned relevant chunks from *Python Data
    Structures* with scores around `0.57–0.59`, but the answer function
    returned the abstention message in one test. This suggests the
    generation/abstention decision can fail even when relevant evidence
    is retrieved. The case should be rerun after API quota reset and the
    prompt, evidence threshold, and retrieved excerpts inspected.
2.  **Incomplete LLM evaluation due to API quota:** Gemini returned
    `429 RESOURCE_EXHAUSTED` after the free-tier daily request limit was
    reached; some earlier requests also returned `503 UNAVAILABLE`. This
    is an evaluation/infrastructure limitation, not evidence that the
    model gave an incorrect answer. The full answer-quality score
    remains unreported until the test set can be completed.

Other limitations: - The current test set is small and manually
written. - The threshold is heuristic and may need adjustment. - HTML
extraction can leave encoding artifacts or irrelevant text in chunks. -
The assistant has not yet been evaluated on a broad range of
paraphrases, ambiguous questions, or adversarial prompts. - The current
notebook is a Colab prototype rather than a deployed web application.

## How to run

1.  Open `Ask_the_Docs_RAG_Assistant_Clean.ipynb` in Google Colab.
2.  Run the notebook cells from top to bottom.
3.  The notebook installs its dependencies, downloads the documentation,
    chunks it, creates embeddings, builds the FAISS index, and runs
    retrieval evaluation.
4.  To enable answer generation, create a Colab Secret named
    `GEMINI_API_KEY`, add your own Gemini API key, and allow notebook
    access to it.
5.  Run the answer-generation cells only when your API quota is
    available. Do not hard-code or commit your API key.

## Improvements I would make next

-   Complete the 10-question answer-quality evaluation and manually
    score correctness, faithfulness to the excerpts, abstention
    behaviour, and citation validity.
-   Expand the test set with more questions, paraphrases, and genuinely
    unanswerable questions.
-   Tune chunk size, overlap, `top_k`, and the similarity threshold
    using a validation set rather than one-off examples.
-   Validate that cited source IDs are actually mentioned by the model
    and restrict displayed citations to those verified IDs.
-   Improve HTML cleanup and normalize text encoding before chunking.
-   Compare MiniLM with another embedding model and consider reranking
    retrieved chunks.
-   Add a simple Streamlit interface and more robust API error handling
    if deployment is needed.

## Repository contents

-   `Ask_the_Docs_RAG_Assistant_Clean.ipynb` --- main runnable notebook.
-   `README.md` --- project overview, design choices, current
    evaluation, limitations, and next steps.

## Security note

Never commit API keys, tokens, or secrets. Store the Gemini key in Colab
Secrets or an environment variable.
