## **1 Fixed-Size & Character-Based Chunking**

### **1. Intuition**

* **Explain like I'm 15:** Imagine you have a 100-page book and want to split it into pieces so a computer can read it quickly [[02:48](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=168)]. The easiest way is to take a pair of scissors and cut the text every 500 words, no matter what [[00:43](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=43)]. To make sure you don't cut a sentence exactly in half and lose the meaning, you leave a little bit of overlapping text at the end of one piece and the start of the next [[01:10](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=70)]. It's quick and simple, but it's blind—you might end up cutting a single topic or a table completely in half [[01:24](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=84)].

### **2. Mathematics**

Let a document string $D$ be tokenized into a sequence of scalar token IDs $T = [t_1, t_2, \dots, t_N]$. Given a predefined window chunk length parameter $L_c$ and a sliding overlap parameter $O_c$ (where $O_c < L_c$), the $i$-th chunk $C_i$ is defined by the token subset:

$$C_i = T[s_i : s_i + L_c]$$

Where the sequence index starting position $s_i$ is determined by the step stride:

$$s_i = i \cdot (L_c - O_c)$$

The trade-off dictates that if $L_c \to \text{small}$ (e.g., 100 tokens), semantic precision increases, but overall context window capacity drops [[03:10](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=190)]. Conversely, if $L_c \to \text{large}$ (e.g., 2000 tokens), it causes **Context Dilution**, where noise reduces the vector embedding's structural sensitivity [[03:47](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=227)].

---

### **3. Code**

The standard implementation leverages utility frameworks like LangChain's recursive separators to prevent splitting critical words [[01:44](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=104)]:

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

# Fixed-size split orchestration
def execution_fixed_chunking(document_text: str):
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=500,       # L_c target window length
        chunk_overlap=50,     # O_c overlapping buffer zone
        length_function=len,
        separators=["\n\n", "\n", " ", ""] # Order of recursive fallback operators
    )
    chunks = splitter.split_text(document_text)
    return chunks

```

---

### **4. Architecture Diagram**

$$\begin{array}{lccccccc}
\textbf{Document Tokens:} & [t_1 \ \dots \ t_{450}] & [t_{451} \ \dots \ t_{500}] & [t_{501} \ \dots \ t_{950}] & [t_{951} \ \dots \ t_{1000}] \\
& \underbrace{\qquad\qquad\qquad\quad}_{C_1 \text{ (Length: 500)}} & \underbrace{\qquad\qquad\qquad\qquad\qquad\quad}_{C_2 \text{ (Length: 500 with Overlap)}}
\end{array}$$

---

### **5. Interview Questions**

1. **When should you opt for a small chunk size (~100 tokens) vs. a large chunk size (~2000 tokens)?**
* *Answer:* Use small chunks for highly modular documents like company FAQ datasets, where questions and answers are localized and concise [[04:48](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=288)]. Use larger chunks for complex domains like legal filings or financial contracts, where references stretch across long distances and require broad context [[04:14](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=254)].


2. **What are the primary performance disadvantages of simple fixed-size character chunking?**
* *Answer:* It lacks semantic awareness. It creates arbitrary cuts that split paragraphs or sentences mid-thought, causing context fragmentation or context dilution, which ultimately lowers vector retrieval accuracy [[02:20](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=140)].



---

### **6. Common Mistakes**

* **Setting Zero Overlap:** Omitting the overlap buffer parameter breaks context continuity. Sentences split exactly at the boundary will fail to match semantic queries during vector search [[01:01](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=61)].

---

### **7. Connections to GPT**

* While modern LLMs like GPT-4 feature massive context windows (up to 128k+ tokens), sending oversized, un-chunked data is costly and inefficient. Effective chunking filters out noise, keeping costs down and preventing accuracy drops in long inputs.

---

## **2.2 Structure-Aware & Hierarchical (Parent-Child) Chunking**

### **1. Intuition**

* **Explain like I'm 15:** Instead of cutting text blindly, **Structure-Aware Chunking** acts like a reader who pays attention to the book's layout [[05:57](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=357)]. It splits text along natural boundaries, like chapters, subheadings, or paragraphs [[06:19](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=379)].
* **Parent-Child Split:** To get the best of both worlds, you can break the text into tiny sentences (**Child Chunks**) for the vector search to scan easily [[11:53](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=713)]. But when the system finds a match, it pulls the entire paragraph or section (**Parent Chunk**) to give the LLM the complete context it needs to answer [[12:17](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=737)].

---

### **2. Mathematics**

Let a document possess a hierarchical tree layout $\mathcal{T}$. Parse operations split nodes based on syntax markers (e.g., HTML tags `<h1>`, Markdown `#`) [[06:36](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=396)]:

$$\text{Split}(D) \to \{S_1, S_2, \dots, S_m\} \quad \text{where } S_k \text{ maps to a native layout header block}$$

In a **Parent-Child Architecture**, we decouple the vector index from the text payload passed to the LLM [[11:53](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=713)]:

1. A parent node $P_j$ is broken down into a set of child nodes $\{C_{j,1}, C_{j,2}, \dots, C_{j,n}\}$ [[12:11](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=731)].
2. The vector embedding function $\phi$ indexes only the child nodes: $\mathbf{e} = \phi(C_{j,k})$.
3. When a query matches child node $C_{j,k}$, the system uses a key-value store mapping to swap it and return the parent text instead:

$$\text{Retrieve}(C_{j,k}) \longrightarrow \text{Lookup}(\text{ID}_{parent}) \longrightarrow P_j$$

---

### **3. Code**

Here is the implementation strategy using explicit metadata injection and hierarchical parent-child pointer associations [[09:09](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=549)]:

```python
import uuid

def build_parent_child_datastructure(raw_sections: list):
    """
    Simulates mapping child text vectors to larger parent text blocks
    """
    vector_index_db = []
    doc_store_payload = {}
    
    for section in raw_sections:
        parent_id = str(uuid.uuid4())
        # Cache the large parent block
        doc_store_payload[parent_id] = section["parent_text"]
        
        # Break down into smaller child segments
        for child_txt in section["child_sentences"]:
            child_metadata = {
                "parent_ptr": parent_id,
                "document_source": "Q1_Report.pdf",
                "page_num": section["page"] # Explicit structural tracking
            }
            vector_index_db.append({"text": child_txt, "meta": child_metadata})
            
    return vector_index_db, doc_store_payload

```

---

### **4. Architecture Diagram**

[Image showing a hierarchical database structure with multiple child text node pointers linking back to a single parent context node]

$$\begin{array}{ccccc}
& & \boxed{\text{Parent Chunk: Full Legal Paragraph (1000 Tokens)}} & & \\
& \swarrow & \downarrow & \searrow & \\
\boxed{\text{Child 1 (200 T)}} & & \boxed{\text{Child 2 (200 T)}} & & \boxed{\text{Child 3 (200 T)}} \\
\downarrow & & \downarrow & & \downarrow \\
\text{Embed Vector }\mathbf{e}_1 & & \text{Embed Vector }\mathbf{e}_2 & & \text{Embed Vector }\mathbf{e}_3
\end{array}$$

---

### **5. Interview Questions**

1. **Why should you append structural metadata (like page numbers or section titles) to individual chunks before indexing them?**
* *Answer:* Metadata acts as a filter during retrieval. For example, if a user specifies a target year or chapter, the system can clear irrelevant chunks before running vector matches, boosting speed and accuracy while preserving key contextual details [[09:57](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=597)].


2. **How does the Parent-Child (or Sentence-Window) retrieval method improve on standard fixed-size chunking configurations?**
* *Answer:* It resolves the conflict between embedding quality and generation context. Small chunks produce highly targeted embeddings free of noise, while retrieving the larger parent chunk gives the LLM the broader context needed to synthesize an accurate answer [[12:17](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=737)].



---

### **6. Common Mistakes**

* **Blind Semantic Retrieval:** Searching chunks without sorting by metadata can pull identical phrases from different parts of a document (like different financial quarters), causing the LLM to mix up data and hallucinate answers [[09:29](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=569)].

---

### **7. Connections to GPT**

* Frameworks like LlamaIndex use this architecture for advanced applications. By pinpointing data via small child nodes, they can supply GPT models with precise parent blocks, optimizing answer quality within token limits.

---

## **2.3 Semantic Chunking & Propositions (Advanced)**

### **1. Intuition**

* **Explain like I'm 15:** * **Semantic Chunking:** Instead of using fixed word counts or layout markers, you let the math decide where to split the text [[14:15](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=855)]. You evaluate sentences one by one; if sentence B shares a similar meaning with sentence A, you keep them in the same group. The moment the topic shifts and sentence C looks different, you break the group and start a new chunk [[15:22](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=922)].
* **Propositions:** You use an LLM to rewrite a complex paragraph into a clean list of standalone facts [[17:45](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=1065)]. Each fact contains all the context it needs to make sense on its own, ensuring no details are lost during retrieval [[18:24](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=1104)].



---

### **2. Mathematics**

#### **Semantic Clustering Strategy**

Given an ordered sequence of sentences $S = [s_1, s_2, \dots, s_n]$, we compute their normalized vector embeddings $\mathbf{v}_i = \phi(s_i)$. We track semantic continuity by calculating the **Cosine Similarity** between adjacent sentences [[15:07](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=907)]:

$$\text{Sim}(s_i, s_{i+1}) = \frac{\mathbf{v}_i \cdot \mathbf{v}_{i+1}}{\|\mathbf{v}_i\| \|\mathbf{v}_{i+1}\|}$$

A topic boundary or split point is automatically generated whenever the similarity falls below a tuned threshold parameter $\tau$ [[16:05](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=965)]:

$$\text{Split Boundary Generated if } \text{Sim}(s_i, s_{i+1}) < \tau$$

#### **Propositional Fact Extraction**

An LLM processes a raw text block $B$ and breaks it down into distinct propositions:

$$\text{LLM}(B) \to \{P_1, P_2, \dots, P_k\}$$

Where each $P_i$ is a atomic sentence containing explicit noun references, stripping away ambiguous pronouns like "it" or "they" [[17:45](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=1065)].

---

### **3. Code**

This design pattern models the calculation loop for tracking semantic shifts across adjacent text blocks [[15:43](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=943)]:

```python
import numpy as np

def calculate_cosine_similarity(v1: np.ndarray, v2: np.ndarray) -> float:
    return float(np.dot(v1, v2) / (np.linalg.norm(v1) * np.linalg.norm(v2)))

def generate_semantic_chunks(sentences: list, embeddings: list, threshold=0.7):
    chunks = []
    current_chunk = [sentences[0]]
    
    for i in range(len(sentences) - 1):
        similarity = calculate_cosine_similarity(embeddings[i], embeddings[i+1])
        
        if similarity >= threshold:
            # Topic continues
            current_chunk.append(sentences[i+1])
        else:
            # Topic shifts; emit chunk and start fresh
            chunks.append(" ".join(current_chunk))
            current_chunk = [sentences[i+1]]
            
    chunks.append(" ".join(current_chunk))
    return chunks

```

---

### **4. Architecture Diagram**

$$\begin{array}{cccccc}
\text{Sentence 1} & \xrightarrow{\quad \text{Similarity: } 0.88 \quad} & \text{Sentence 2} & \xrightarrow{\quad \text{Similarity: } 0.42 \quad} & \text{Sentence 3} \\
\underbrace{\qquad\qquad\qquad\qquad\qquad\qquad\quad}_{\textbf{Chunk 1 (Topic A)}} & & \underbrace{\qquad\qquad\qquad\quad}_{\textbf{Chunk 2 (Topic B: New Axis)}}
\end{array}$$

---

### **5. Interview Questions**

1. **What is the primary computational bottleneck when implementing Semantic Chunking across large document setups?**
* *Answer:* It requires significant compute power. The system must generate embeddings for every single sentence and calculate distance metrics for all adjacent pairs, making it costly for large-scale operations [[17:03](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=1023)].


2. **In what scenarios is Propositional Chunking preferred over character splitters?**
* *Answer:* It is ideal for high-stakes, high-accuracy fields like medicine, finance, and law [[19:52](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=1192)]. In these spaces, information density is high and missing a single detail can cause major issues, justifying the extra LLM cost to extract exact, standalone facts [[18:33](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=1113)].



---

### **6. Common Mistakes**

* **Using Fixed Thresholds Globally:** Applying a single similarity cutoff ($\tau$) across diverse document types. A cutoff that works well for structured narratives might over-split data-heavy reports, creating fragmented chunks.

---

### **7. Connections to GPT**

* Proposition-based chunking uses an initial LLM pass (like GPT-3.5) to clean and structure text into clean components. This refined layout ensures later retrieval steps provide highly relevant context for the final generation model.

---

## **2.4 Evaluation Frameworks & Chonking Metrics**

### **1. Intuition**

* **Explain like I'm 15:** Once you set up your chunking strategy, you need a reliable way to test if it's working well [[20:24](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=1224)]. Think of this like checking an exam score sheet. You use a test set of 50 sample questions to evaluate your pipeline across different metrics: how accurately it finds the right information (**Recall**), how much noise gets into the results (**Precision**), and whether the final answer stays grounded in the source text without hallucinating (**Faithfulness**) [[21:19](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=1279), [23:59](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=1439)].

---

### **2. Mathematics**

RAG evaluation measures the performance of both retrieval and synthesis stages [[21:13](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=1273)]:

#### **Context Precision**

Measures what fraction of the retrieved chunks are actually relevant to the query [[23:34](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=1414)]:


$$\text{Context Precision} = \frac{|\{\text{Retrieved Chunks}\} \cap \{\text{Truly Relevant Chunks}\}|}{|\{\text{Retrieved Chunks}\}|}$$

#### **Retrieval Recall**

Measures the system's ability to find all the necessary target text snippets [[21:19](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=1279)]:


$$\text{Retrieval Recall} = \frac{|\{\text{Retrieved Chunks}\} \cap \{\text{Truly Relevant Chunks}\}|}{|\{\text{Truly Relevant Chunks}\}|}$$

#### **Mean Reciprocal Rank (MRR)**

Evaluates the position of the first relevant chunk in the returned list, prioritizing systems that return the correct answer at the very top [[21:53](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=1313)]:


$$\text{MRR} = \frac{1}{|Q|} \sum_{i=1}^{|Q|} \frac{1}{\text{rank}_i}$$


Where $\text{rank}_i$ is the position of the first correct chunk returned for query $i$ [[22:14](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=1334)].

---

### **3. Code**

A standard evaluation configuration tracking performance across different chunking methods [[24:27](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=1467)]:

```python
def execute_rag_triad_evaluation(query_dataset: list, pipeline_retrieval_func):
    """
    Evaluates retrieval performance across a test dataset
    """
    total_mrr = 0.0
    
    for item in query_dataset:
        true_gold_chunk_id = item["target_chunk_id"]
        # Run retrieval step
        retrieved_nodes = pipeline_retrieval_func(item["question"])
        
        # Calculate Reciprocal Rank
        for rank, node in enumerate(retrieved_nodes, start=1):
            if node.id == true_gold_chunk_id:
                total_mrr += (1.0 / rank)
                break
                
    mean_reciprocal_rank = total_mrr / len(query_dataset)
    return {"Evaluation Mean MRR": mean_reciprocal_rank}

```

---

### **4. Architecture Diagram**

$$\begin{array}{ccccc}
& \boxed{\textbf{Ground Truth Question}} & & \\
& \swarrow \quad \text{(Retrieval Step)} \quad \searrow & & \\
\boxed{\text{Context Precision \& Recall}} & \longleftrightarrow & \boxed{\text{Answer Faithfulness \& Relevance}}
\end{array}$$

---

### **5. Interview Questions**

1. **What is the difference between Answer Faithfulness and Answer Relevance in RAG metrics?**
* *Answer:* **Faithfulness** checks if the generated answer is strictly grounded in the retrieved context to detect hallucinations [[22:28](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=1348)]. **Relevance** measures how well the answer addresses the user's prompt, ensuring it answers the actual question asked rather than giving an unrelated response [[23:08](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=1388)].


2. **How do you design a robust automated evaluation pipeline to select the optimal chunk size parameter?**
* *Answer:* Build a test set of 50 to 100 representative questions with mapped ground-truth context links [[23:59](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=1439)]. Run different chunk sizes as hyperparameter variants against this dataset, measuring performance across metrics like MRR, Recall, and Precision to select the best configuration [[24:27](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=1467)].



---

### **6. Common Mistakes**

* **Relying Solely on Semantic Distance Scores:** Relying only on raw cosine scores (e.g., 0.92) to judge retrieval quality. High vector similarity does not guarantee a chunk actually contains the correct answer to satisfy the user's prompt [[22:05](http://www.youtube.com/watch?v=9ztE_pX7Ltc&t=1325)].

---

### **7. Connections to GPT**

* Production evaluation tools like **Ragas** or **TruLens** rely on GPT-4 as an automated judge. They use structured prompts to analyze chunks and answers, scoring performance across the entire evaluation matrix without requiring slow, manual human reviews.
