# RAG Pipeline Architecture

## 1. Tổng quan

RAG (Retrieval-Augmented Generation) Pipeline trong Dify là hệ thống phức tạp để:
- **Ingestion**: Trích xuất và xử lý documents từ nhiều sources
- **Chunking**: Chia nhỏ documents thành chunks phù hợp
- **Embedding**: Tạo vector embeddings cho semantic search
- **Storage**: Lưu trữ trong vector databases
- **Retrieval**: Tìm kiếm thông tin relevant
- **Reranking**: Sắp xếp lại kết quả theo relevance

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           RAG PIPELINE                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      INGESTION PHASE                             │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │   │
│  │  │  Extractor   │  │   Cleaner    │  │      Splitter        │  │   │
│  │  │  (PDF, Doc,  │─►│  (Normalize) │─►│  (Chunk Documents)   │  │   │
│  │  │   HTML...)   │  │              │  │                      │  │   │
│  │  └──────────────┘  └──────────────┘  └──────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                    │
│                                    ▼                                    │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      EMBEDDING PHASE                             │   │
│  │  ┌──────────────────────────────────────────────────────────┐  │   │
│  │  │                  CachedEmbedding                          │  │   │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │  │   │
│  │  │  │  DB Cache   │  │ Redis Cache │  │ LLM Provider│      │  │   │
│  │  │  │ (Documents) │  │  (Queries)  │  │  (Fallback) │      │  │   │
│  │  │  └─────────────┘  └─────────────┘  └─────────────┘      │  │   │
│  │  └──────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                    │
│                                    ▼                                    │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      STORAGE PHASE                               │   │
│  │  ┌──────────────────────────────────────────────────────────┐  │   │
│  │  │                   Vector Factory                          │  │   │
│  │  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ │  │   │
│  │  │  │Weaviate│ │Pinecone│ │PGVector│ │ Qdrant │ │  30+   │ │  │   │
│  │  │  │        │ │        │ │        │ │        │ │ others │ │  │   │
│  │  │  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘ │  │   │
│  │  └──────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                    │
│                                    ▼                                    │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      RETRIEVAL PHASE                             │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐                │   │
│  │  │  Semantic  │  │ Full-Text  │  │   Hybrid   │                │   │
│  │  │   Search   │  │   Search   │  │   Search   │                │   │
│  │  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘                │   │
│  │        └───────────────┼───────────────┘                        │   │
│  │                        ▼                                        │   │
│  │                 ┌────────────┐                                  │   │
│  │                 │  Reranker  │                                  │   │
│  │                 │ (Optional) │                                  │   │
│  │                 └────────────┘                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## 3. Document Ingestion Flow

### 3.1 Extractor System

**Reference**: `/api/core/rag/extractor/extract_processor.py`

```python
class ExtractProcessor:
    """Document extraction from various sources."""

    @classmethod
    def load_from_upload_file(
        cls,
        upload_file: UploadFile,
        return_text: bool = True,
    ) -> list[Document]:
        """Load document from uploaded file."""
        file_path = upload_file.path
        file_extension = upload_file.extension.lower()

        # Select appropriate extractor
        extractor = cls._get_extractor(file_extension)

        # Extract content
        return extractor.extract(file_path, return_text=return_text)

    @classmethod
    def load_from_url(
        cls,
        url: str,
        return_text: bool = True,
    ) -> list[Document]:
        """Extract content from URL."""
        # Determine web extractor
        if dify_config.WEB_EXTRACTOR == "jina":
            extractor = JinaReaderWebExtractor(url)
        elif dify_config.WEB_EXTRACTOR == "firecrawl":
            extractor = FirecrawlWebExtractor(url)
        else:
            extractor = WaterCrawlWebExtractor(url)

        return extractor.extract(return_text=return_text)

    @staticmethod
    def _get_extractor(extension: str) -> BaseExtractor:
        """Get extractor for file type."""
        extractors = {
            "pdf": PdfExtractor,
            "docx": WordExtractor,
            "doc": UnstructuredWordExtractor,
            "xlsx": ExcelExtractor,
            "xls": ExcelExtractor,
            "csv": CSVExtractor,
            "html": HtmlExtractor,
            "htm": HtmlExtractor,
            "md": MarkdownExtractor,
            "markdown": MarkdownExtractor,
            "txt": TextExtractor,
            "json": JsonExtractor,
            "xml": XmlExtractor,
            "pptx": PowerPointExtractor,
            "epub": UnstructuredEpubExtractor,
            "eml": UnstructuredEmailExtractor,
            "msg": UnstructuredEmailExtractor,
        }
        return extractors.get(extension, TextExtractor)()
```

### 3.2 Supported Extractors

| File Type | Extractor | Location |
|-----------|-----------|----------|
| PDF | `PdfExtractor` | `/api/core/rag/extractor/pdf/` |
| Word (.docx) | `WordExtractor` | `/api/core/rag/extractor/word/` |
| Word (.doc) | `UnstructuredWordExtractor` | `/api/core/rag/extractor/unstructured/` |
| Excel | `ExcelExtractor` | `/api/core/rag/extractor/excel/` |
| CSV | `CSVExtractor` | `/api/core/rag/extractor/csv/` |
| HTML | `HtmlExtractor` | `/api/core/rag/extractor/html/` |
| Markdown | `MarkdownExtractor` | `/api/core/rag/extractor/markdown/` |
| EPUB | `UnstructuredEpubExtractor` | `/api/core/rag/extractor/unstructured/` |
| Email | `UnstructuredEmailExtractor` | `/api/core/rag/extractor/unstructured/` |
| Web URL | `JinaReaderWebExtractor` | `/api/core/rag/extractor/web/` |
| Web URL | `FirecrawlWebExtractor` | `/api/core/rag/extractor/web/` |

### 3.3 Clean Processor

**Reference**: `/api/core/rag/cleaner/clean_processor.py` (lines 5-50)

```python
class CleanProcessor:
    """Document cleaning and normalization."""

    @staticmethod
    def clean(
        text: str,
        rules: list[str] | None = None,
    ) -> str:
        """Apply cleaning rules to text."""
        if rules is None:
            rules = ["remove_extra_spaces", "remove_urls_emails"]

        for rule in rules:
            if rule == "remove_extra_spaces":
                # Remove multiple spaces
                text = re.sub(r"\s+", " ", text)
                # Remove leading/trailing whitespace
                text = text.strip()

            elif rule == "remove_urls_emails":
                # Remove URLs but preserve Markdown image URLs
                text = re.sub(
                    r"(?<!!)\[([^\]]+)\]\(https?://[^\)]+\)",
                    r"\1",
                    text,
                )
                # Remove standalone URLs
                text = re.sub(
                    r"https?://\S+",
                    "",
                    text,
                )
                # Remove emails
                text = re.sub(
                    r"\b[\w.-]+@[\w.-]+\.\w{2,}\b",
                    "",
                    text,
                )

        return text
```

## 4. Document Splitting Strategies

### 4.1 Text Splitter Base

**Reference**: `/api/core/rag/splitter/text_splitter.py` (lines 39-163)

```python
from abc import abstractmethod

class TextSplitter:
    """Base class for text splitting strategies."""

    def __init__(
        self,
        chunk_size: int = 4000,
        chunk_overlap: int = 200,
        length_function: Callable[[str], int] = len,
        keep_separator: bool = False,
    ):
        self._chunk_size = chunk_size
        self._chunk_overlap = chunk_overlap
        self._length_function = length_function
        self._keep_separator = keep_separator

    @abstractmethod
    def split_text(self, text: str) -> list[str]:
        """Split text into chunks."""
        ...

    def create_documents(
        self,
        texts: list[str],
        metadatas: list[dict] | None = None,
    ) -> list[Document]:
        """Create Document objects from text chunks."""
        documents = []
        for i, text in enumerate(texts):
            chunks = self.split_text(text)
            for chunk in chunks:
                metadata = metadatas[i] if metadatas else {}
                documents.append(
                    Document(page_content=chunk, metadata=metadata)
                )
        return documents

    def _merge_splits(
        self,
        splits: list[str],
        separator: str,
    ) -> list[str]:
        """Merge small chunks with overlap."""
        merged = []
        current_chunk = []
        current_length = 0

        for split in splits:
            split_length = self._length_function(split)

            if current_length + split_length > self._chunk_size:
                # Save current chunk
                if current_chunk:
                    merged.append(separator.join(current_chunk))

                # Start new chunk with overlap
                overlap_splits = self._get_overlap_splits(
                    current_chunk,
                    separator,
                )
                current_chunk = overlap_splits + [split]
                current_length = sum(
                    self._length_function(s) for s in current_chunk
                )
            else:
                current_chunk.append(split)
                current_length += split_length

        if current_chunk:
            merged.append(separator.join(current_chunk))

        return merged
```

### 4.2 Recursive Character Text Splitter

**Reference**: `/api/core/rag/splitter/fixed_text_splitter.py` (lines 52-153)

```python
class FixedRecursiveCharacterTextSplitter(TextSplitter):
    """Recursive splitting with multiple separators."""

    SEPARATORS = ["\n\n", "\n", "。", ". ", " ", ""]

    def __init__(
        self,
        fixed_separator: str = "\n",
        separators: list[str] | None = None,
        **kwargs,
    ):
        super().__init__(**kwargs)
        self._fixed_separator = fixed_separator
        self._separators = separators or self.SEPARATORS

    def split_text(self, text: str) -> list[str]:
        """Split text using fixed separator first, then recursively."""
        # First split by fixed separator
        splits = text.split(self._fixed_separator)

        # Then recursively split large chunks
        final_chunks = []
        for split in splits:
            if self._length_function(split) <= self._chunk_size:
                final_chunks.append(split)
            else:
                # Recursively split
                sub_chunks = self._recursive_split(split, self._separators)
                final_chunks.extend(sub_chunks)

        # Merge small chunks
        return self._merge_splits(final_chunks, self._fixed_separator)

    def _recursive_split(
        self,
        text: str,
        separators: list[str],
    ) -> list[str]:
        """Recursively split text with fallback separators."""
        final_chunks = []

        # Find appropriate separator
        separator = separators[-1]  # Default to last (empty string)
        for sep in separators:
            if sep == "":
                separator = sep
                break
            if sep in text:
                separator = sep
                break

        # Split by separator
        if separator:
            splits = text.split(separator)
        else:
            splits = list(text)  # Character level

        # Process splits
        current_chunk = []
        for split in splits:
            if self._length_function(split) > self._chunk_size:
                # Still too large, recurse with next separators
                if current_chunk:
                    final_chunks.extend(
                        self._merge_splits(current_chunk, separator)
                    )
                    current_chunk = []

                sub_chunks = self._recursive_split(
                    split,
                    separators[separators.index(separator) + 1:],
                )
                final_chunks.extend(sub_chunks)
            else:
                current_chunk.append(split)

        if current_chunk:
            final_chunks.extend(
                self._merge_splits(current_chunk, separator)
            )

        return final_chunks
```

### 4.3 Token-Based Splitter

```python
class TokenTextSplitter(TextSplitter):
    """Split by token count using tiktoken."""

    def __init__(
        self,
        encoding_name: str = "cl100k_base",
        **kwargs,
    ):
        super().__init__(**kwargs)
        self._tokenizer = tiktoken.get_encoding(encoding_name)

    def split_text(self, text: str) -> list[str]:
        """Split text by token count."""
        tokens = self._tokenizer.encode(text)
        chunks = []

        for i in range(0, len(tokens), self._chunk_size - self._chunk_overlap):
            chunk_tokens = tokens[i:i + self._chunk_size]
            chunk_text = self._tokenizer.decode(chunk_tokens)
            chunks.append(chunk_text)

        return chunks
```

## 5. Index Structure Types

### 5.1 Paragraph Index (Default)

**Reference**: `/api/core/rag/index_processor/processor/paragraph_index_processor.py`

```
Document → Chunks → Embeddings → Vector DB

┌──────────────────────────────────────────────────────────────┐
│  Original Document                                            │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Lorem ipsum dolor sit amet, consectetur adipiscing   │   │
│  │ elit. Sed do eiusmod tempor incididunt ut labore et  │   │
│  │ dolore magna aliqua. Ut enim ad minim veniam, quis   │   │
│  │ nostrud exercitation ullamco laboris nisi ut aliquip │   │
│  │ ex ea commodo consequat...                           │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼ Chunking
┌──────────────────────────────────────────────────────────────┐
│  Chunks (with overlap)                                        │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐               │
│  │  Chunk 1   │ │  Chunk 2   │ │  Chunk 3   │  ...          │
│  │ (overlap)  │ │ (overlap)  │ │ (overlap)  │               │
│  └────────────┘ └────────────┘ └────────────┘               │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼ Embedding
┌──────────────────────────────────────────────────────────────┐
│  Vector Representations                                       │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐               │
│  │ [0.1, 0.2, │ │ [0.3, 0.4, │ │ [0.5, 0.6, │  ...          │
│  │  0.3, ...] │ │  0.1, ...] │ │  0.2, ...] │               │
│  └────────────┘ └────────────┘ └────────────┘               │
└──────────────────────────────────────────────────────────────┘
```

### 5.2 Parent-Child Index

**Reference**: `/api/core/rag/index_processor/processor/parent_child_index_processor.py`

```
Document → Parent Chunks → Child Chunks → Embeddings

┌──────────────────────────────────────────────────────────────┐
│  Parent Chunk (Large, for context)                            │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Full paragraph with complete context                  │   │
│  │ Lorem ipsum dolor sit amet, consectetur adipiscing   │   │
│  │ elit. Sed do eiusmod tempor incididunt ut labore... │   │
│  └──────────────────────────────────────────────────────┘   │
│                              │                               │
│                              ▼                               │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐   │
│  │  Child 1  │ │  Child 2  │ │  Child 3  │ │  Child 4  │   │
│  │ (indexed) │ │ (indexed) │ │ (indexed) │ │ (indexed) │   │
│  └───────────┘ └───────────┘ └───────────┘ └───────────┘   │
└──────────────────────────────────────────────────────────────┘

Retrieval: Search children → Return with parent context
```

### 5.3 QA Index

**Reference**: `/api/core/rag/index_processor/processor/qa_index_processor.py`

```
Document → Generated Q&A Pairs → Embeddings

┌──────────────────────────────────────────────────────────────┐
│  Original Document                                            │
│  "The capital of France is Paris. Paris is known for         │
│   the Eiffel Tower, which was built in 1889..."              │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼ LLM Generation
┌──────────────────────────────────────────────────────────────┐
│  Generated Q&A Pairs                                          │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Q: What is the capital of France?                     │   │
│  │ A: The capital of France is Paris.                    │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Q: When was the Eiffel Tower built?                   │   │
│  │ A: The Eiffel Tower was built in 1889.               │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘

Retrieval: Match query to questions → Return answers
```

## 6. Embedding System

### 6.1 Cached Embedding

**Reference**: `/api/core/rag/embedding/cached_embedding.py` (lines 23-273)

```python
class CacheEmbedding(Embeddings):
    """Embedding with database and Redis caching."""

    def __init__(
        self,
        model_instance: TextEmbeddingModel,
        user_id: str,
    ):
        self._model_instance = model_instance
        self._user_id = user_id

    def embed_documents(
        self,
        texts: list[str],
    ) -> list[list[float]]:
        """Embed documents with database caching."""
        embeddings = []
        texts_to_embed = []
        text_indices = []

        for i, text in enumerate(texts):
            # Check cache first
            text_hash = helper.generate_text_hash(text)
            cached = self._get_from_cache(text_hash)

            if cached:
                embeddings.append(cached)
            else:
                texts_to_embed.append(text)
                text_indices.append(i)

        # Batch embed remaining texts
        if texts_to_embed:
            new_embeddings = self._batch_embed(texts_to_embed)

            # Normalize embeddings (L2 normalization)
            for emb in new_embeddings:
                norm = np.linalg.norm(emb)
                if norm > 0:
                    emb = (np.array(emb) / norm).tolist()

            # Cache and insert at correct positions
            for idx, emb in zip(text_indices, new_embeddings):
                text_hash = helper.generate_text_hash(texts[idx])
                self._save_to_cache(text_hash, emb)
                embeddings.insert(idx, emb)

        return embeddings

    def embed_query(self, text: str) -> list[float]:
        """Embed query with Redis caching."""
        # Check Redis cache (600 second TTL)
        cache_key = f"embedding:query:{helper.generate_text_hash(text)}"
        cached = redis_client.get(cache_key)

        if cached:
            return pickle.loads(base64.b64decode(cached))

        # Embed via model
        embedding = self._model_instance.invoke_text_embedding(
            texts=[text],
            input_type=TextEmbeddingInputType.QUERY,
        )

        # L2 normalize
        embedding = embedding[0]
        norm = np.linalg.norm(embedding)
        if norm > 0:
            embedding = (np.array(embedding) / norm).tolist()

        # Cache in Redis
        encoded = base64.b64encode(pickle.dumps(embedding)).decode()
        redis_client.setex(cache_key, 600, encoded)

        return embedding

    def _batch_embed(
        self,
        texts: list[str],
        batch_size: int = 1000,
    ) -> list[list[float]]:
        """Batch embed texts."""
        embeddings = []

        for i in range(0, len(texts), batch_size):
            batch = texts[i:i + batch_size]
            batch_embeddings = self._model_instance.invoke_text_embedding(
                texts=batch,
                input_type=TextEmbeddingInputType.DOCUMENT,
            )
            embeddings.extend(batch_embeddings)

        return embeddings
```

### 6.2 Embedding Base Interface

**Reference**: `/api/core/rag/embedding/embedding_base.py` (lines 4-33)

```python
from abc import ABC, abstractmethod

class Embeddings(ABC):
    """Abstract base class for embeddings."""

    @abstractmethod
    def embed_documents(self, texts: list[str]) -> list[list[float]]:
        """Embed multiple documents."""
        ...

    @abstractmethod
    def embed_query(self, text: str) -> list[float]:
        """Embed a single query."""
        ...

    # Multimodal support
    def embed_multimodal_documents(
        self,
        multimodal_documents: list[MultimodalDocument],
    ) -> list[list[float]]:
        """Embed multimodal content (images, etc.)."""
        raise NotImplementedError

    def embed_multimodal_query(
        self,
        multimodal_document: MultimodalDocument,
    ) -> list[float]:
        """Embed multimodal query."""
        raise NotImplementedError
```

## 7. Vector Database Integration

### 7.1 Vector Factory

**Reference**: `/api/core/rag/datasource/vdb/vector_factory.py` (lines 38-253)

```python
class Vector:
    """Unified vector database interface."""

    def __init__(
        self,
        dataset: Dataset,
        attributes: list[str] | None = None,
    ):
        self._dataset = dataset
        self._attributes = attributes or ["doc_id", "dataset_id", "document_id"]
        self._embeddings = self._get_embeddings()
        self._vector_processor = self._init_vector()

    def _init_vector(self) -> BaseVector:
        """Initialize appropriate vector store."""
        vector_type = dify_config.VECTOR_STORE

        vector_classes = {
            "weaviate": WeaviateVector,
            "pinecone": PineconeVector,
            "pgvector": PGVectorVector,
            "qdrant": QdrantVector,
            "milvus": MilvusVector,
            "chroma": ChromaVector,
            "elasticsearch": ElasticsearchVector,
            "opensearch": OpenSearchVector,
            "tidb_vector": TiDBVector,
            # ... 30+ more
        }

        vector_class = vector_classes.get(vector_type)
        if not vector_class:
            raise ValueError(f"Unknown vector store: {vector_type}")

        return vector_class(
            collection_name=self._get_collection_name(),
            config=self._get_config(),
        )

    def create(
        self,
        texts: list[Document],
        batch_size: int = 1000,
    ) -> None:
        """Create embeddings and store in vector DB."""
        for i in range(0, len(texts), batch_size):
            batch = texts[i:i + batch_size]

            # Generate embeddings
            text_contents = [doc.page_content for doc in batch]
            embeddings = self._embeddings.embed_documents(text_contents)

            # Store in vector DB
            self._vector_processor.create(
                texts=batch,
                embeddings=embeddings,
            )

    def search_by_vector(
        self,
        query_vector: list[float],
        top_k: int = 4,
        score_threshold: float | None = None,
        filter: dict | None = None,
    ) -> list[Document]:
        """Semantic search by vector."""
        return self._vector_processor.search_by_vector(
            query_vector=query_vector,
            k=top_k,
            score_threshold=score_threshold,
            filter=filter,
        )

    def search_by_full_text(
        self,
        query: str,
        top_k: int = 4,
    ) -> list[Document]:
        """Full-text search."""
        return self._vector_processor.search_by_full_text(
            query=query,
            top_k=top_k,
        )
```

### 7.2 Supported Vector Databases

| Vector DB | Integration | Key Features |
|-----------|-------------|--------------|
| **Weaviate** | `WeaviateVector` | GraphQL, hybrid search |
| **Pinecone** | `PineconeVector` | Serverless, fast |
| **PGVector** | `PGVectorVector` | PostgreSQL extension |
| **Qdrant** | `QdrantVector` | Rust-based, fast |
| **Milvus** | `MilvusVector` | Distributed, scalable |
| **Chroma** | `ChromaVector` | Simple, embedded |
| **Elasticsearch** | `ElasticsearchVector` | Full-text + vector |
| **OpenSearch** | `OpenSearchVector` | AWS managed option |
| **TiDB Vector** | `TiDBVector` | MySQL compatible |
| **AnalyticDB** | `AnalyticDBVector` | Alibaba Cloud |
| **Couchbase** | `CouchbaseVector` | Document + vector |
| **Oracle** | `OracleVector` | Enterprise grade |
| ... | ... | 30+ more integrations |

### 7.3 Vector Base Interface

**Reference**: `/api/core/rag/datasource/vdb/vector_base.py` (lines 9-67)

```python
from abc import ABC, abstractmethod

class BaseVector(ABC):
    """Abstract base class for vector stores."""

    def __init__(
        self,
        collection_name: str,
        config: VectorConfig,
    ):
        self._collection_name = collection_name
        self._config = config

    @abstractmethod
    def create(
        self,
        texts: list[Document],
        embeddings: list[list[float]],
    ) -> None:
        """Create collection with documents."""
        ...

    @abstractmethod
    def add_texts(
        self,
        documents: list[Document],
        embeddings: list[list[float]],
    ) -> None:
        """Add documents to existing collection."""
        ...

    @abstractmethod
    def text_exists(self, id: str) -> bool:
        """Check if document exists."""
        ...

    @abstractmethod
    def delete_by_ids(self, ids: list[str]) -> None:
        """Delete documents by ID."""
        ...

    @abstractmethod
    def search_by_vector(
        self,
        query_vector: list[float],
        k: int = 4,
        score_threshold: float | None = None,
        filter: dict | None = None,
    ) -> list[Document]:
        """Vector similarity search."""
        ...

    @abstractmethod
    def search_by_full_text(
        self,
        query: str,
        top_k: int = 4,
    ) -> list[Document]:
        """Full-text search."""
        ...
```

## 8. Retrieval System

### 8.1 Retrieval Methods

**Reference**: `/api/core/rag/retrieval/retrieval_methods.py` (lines 4-8)

```python
from enum import StrEnum

class RetrievalMethod(StrEnum):
    """Available retrieval strategies."""
    SEMANTIC_SEARCH = "semantic_search"    # Vector similarity
    FULL_TEXT_SEARCH = "full_text_search"  # BM25/keyword
    HYBRID_SEARCH = "hybrid_search"        # Combined
    KEYWORD_SEARCH = "keyword_search"      # Legacy keyword
```

### 8.2 Retrieval Service

**Reference**: `/api/core/rag/datasource/retrieval_service.py` (lines 42-356)

```python
from concurrent.futures import ThreadPoolExecutor

class RetrievalService:
    """Main retrieval service."""

    @classmethod
    def retrieve(
        cls,
        retrieval_method: RetrievalMethod,
        dataset_id: str,
        query: str,
        top_k: int = 4,
        score_threshold: float | None = None,
        reranking_model: dict | None = None,
        attachment_ids: list[str] | None = None,
    ) -> list[Document]:
        """
        Execute retrieval with specified method.

        Args:
            retrieval_method: Search strategy
            dataset_id: Target dataset
            query: Search query
            top_k: Number of results
            score_threshold: Minimum score
            reranking_model: Optional reranker config
            attachment_ids: Image attachments for multimodal

        Returns:
            List of matching documents
        """
        dataset = Dataset.query.get(dataset_id)
        if not dataset:
            return []

        # Execute retrieval based on method
        if retrieval_method == RetrievalMethod.SEMANTIC_SEARCH:
            results = cls.embedding_search(
                dataset=dataset,
                query=query,
                top_k=top_k,
                score_threshold=score_threshold,
                reranking_model=reranking_model,
                attachment_ids=attachment_ids,
            )
        elif retrieval_method == RetrievalMethod.FULL_TEXT_SEARCH:
            results = cls.full_text_index_search(
                dataset=dataset,
                query=query,
                top_k=top_k,
                reranking_model=reranking_model,
            )
        elif retrieval_method == RetrievalMethod.HYBRID_SEARCH:
            # Parallel execution
            with ThreadPoolExecutor(max_workers=2) as executor:
                semantic_future = executor.submit(
                    cls.embedding_search,
                    dataset, query, top_k, score_threshold,
                )
                fulltext_future = executor.submit(
                    cls.full_text_index_search,
                    dataset, query, top_k,
                )

            semantic_results = semantic_future.result()
            fulltext_results = fulltext_future.result()

            # Merge and deduplicate
            results = cls._merge_results(
                semantic_results,
                fulltext_results,
                reranking_model,
            )
        else:
            results = cls.keyword_search(dataset, query, top_k)

        return results

    @classmethod
    def embedding_search(
        cls,
        dataset: Dataset,
        query: str,
        top_k: int = 4,
        score_threshold: float | None = None,
        reranking_model: dict | None = None,
        attachment_ids: list[str] | None = None,
    ) -> list[Document]:
        """Semantic search using embeddings."""
        vector = Vector(dataset)

        # Text or multimodal query
        if attachment_ids:
            # Multimodal search with images
            results = vector.search_by_file(
                file_ids=attachment_ids,
                top_k=top_k,
                score_threshold=score_threshold,
            )
        else:
            # Text query embedding
            query_embedding = vector._embeddings.embed_query(query)
            results = vector.search_by_vector(
                query_vector=query_embedding,
                top_k=top_k,
                score_threshold=score_threshold,
            )

        # Optional reranking
        if reranking_model:
            results = DataPostProcessor.invoke(
                query=query,
                documents=results,
                score_threshold=score_threshold,
                reranking_model=reranking_model,
            )

        return results

    @classmethod
    def full_text_index_search(
        cls,
        dataset: Dataset,
        query: str,
        top_k: int = 4,
        reranking_model: dict | None = None,
    ) -> list[Document]:
        """Full-text search using inverted index."""
        vector = Vector(dataset)

        # Escape query for search safety
        escaped_query = cls._escape_query(query)

        results = vector.search_by_full_text(
            query=escaped_query,
            top_k=top_k,
        )

        # Optional reranking
        if reranking_model:
            results = DataPostProcessor.invoke(
                query=query,
                documents=results,
                reranking_model=reranking_model,
            )

        return results
```

### 8.3 Dataset Retrieval (High-Level API)

**Reference**: `/api/core/rag/retrieval/dataset_retrieval.py` (lines 77-1457)

```python
class DatasetRetrieval:
    """High-level retrieval across datasets."""

    def retrieve(
        self,
        dataset_configs: list[DatasetConfig],
        query: str,
        invoke_from: InvokeFrom,
        show_retrieve_source: bool = False,
        hit_callback: Callable | None = None,
    ) -> list[Document]:
        """
        Retrieve from multiple datasets.

        Args:
            dataset_configs: List of dataset configurations
            query: User query
            invoke_from: Source of invocation
            show_retrieve_source: Include source info
            hit_callback: Callback for hit tracking

        Returns:
            Combined results from all datasets
        """
        dataset_ids = [c.dataset_id for c in dataset_configs]

        if len(dataset_ids) == 1:
            # Single dataset - direct retrieval
            return self.single_retrieve(
                dataset_id=dataset_ids[0],
                query=query,
                config=dataset_configs[0],
            )
        else:
            # Multiple datasets - parallel retrieval
            return self.multiple_retrieve(
                dataset_ids=dataset_ids,
                query=query,
                configs=dataset_configs,
            )

    def single_retrieve(
        self,
        dataset_id: str,
        query: str,
        config: DatasetConfig,
    ) -> list[Document]:
        """Retrieve from single dataset."""
        return RetrievalService.retrieve(
            retrieval_method=config.retrieval_method,
            dataset_id=dataset_id,
            query=query,
            top_k=config.top_k,
            score_threshold=config.score_threshold,
            reranking_model=config.reranking_model,
        )

    def multiple_retrieve(
        self,
        dataset_ids: list[str],
        query: str,
        configs: list[DatasetConfig],
    ) -> list[Document]:
        """Parallel retrieval from multiple datasets."""
        all_results = []

        with ThreadPoolExecutor(max_workers=len(dataset_ids)) as executor:
            futures = {
                executor.submit(
                    self.single_retrieve,
                    dataset_id,
                    query,
                    config,
                ): dataset_id
                for dataset_id, config in zip(dataset_ids, configs)
            }

            for future in futures:
                try:
                    results = future.result()
                    all_results.extend(results)
                except Exception as e:
                    logger.exception(f"Retrieval failed: {e}")

        # Deduplicate and rerank combined results
        return self._deduplicate_and_rerank(all_results, query)
```

## 9. Reranking System

### 9.1 Data Post-Processor

**Reference**: `/api/core/rag/data_post_processor/data_post_processor.py` (lines 13-99)

```python
class DataPostProcessor:
    """Post-process retrieval results."""

    @classmethod
    def invoke(
        cls,
        query: str,
        documents: list[Document],
        score_threshold: float | None = None,
        reranking_model: dict | None = None,
        reordering: str | None = None,
    ) -> list[Document]:
        """
        Apply reranking and reordering.

        Args:
            query: Original query
            documents: Retrieved documents
            score_threshold: Minimum relevance score
            reranking_model: Reranking configuration
            reordering: Reordering strategy

        Returns:
            Reranked and filtered documents
        """
        # Get appropriate rerank runner
        if reranking_model:
            rerank_runner = cls._get_rerank_runner(reranking_model)
            documents = rerank_runner.run(
                query=query,
                documents=documents,
                score_threshold=score_threshold,
            )

        # Apply reordering if specified
        if reordering:
            documents = cls._reorder(documents, reordering)

        return documents

    @classmethod
    def _get_rerank_runner(
        cls,
        reranking_model: dict,
    ) -> BaseRerankRunner:
        """Get rerank runner based on config."""
        mode = reranking_model.get("mode", RerankMode.RERANKING_MODEL)

        if mode == RerankMode.WEIGHTED_SCORE:
            return WeightRerankRunner(
                vector_weight=reranking_model.get("vector_weight", 0.7),
                keyword_weight=reranking_model.get("keyword_weight", 0.3),
            )
        else:
            return RerankModelRunner(
                provider=reranking_model["provider"],
                model=reranking_model["model"],
            )
```

### 9.2 Weight-Based Reranking

**Reference**: `/api/core/rag/rerank/weight_rerank.py` (lines 17-196)

```python
import jieba

class WeightRerankRunner(BaseRerankRunner):
    """Combine keyword and vector scores."""

    def __init__(
        self,
        vector_weight: float = 0.7,
        keyword_weight: float = 0.3,
    ):
        self._vector_weight = vector_weight
        self._keyword_weight = keyword_weight

    def run(
        self,
        query: str,
        documents: list[Document],
        score_threshold: float | None = None,
    ) -> list[Document]:
        """Calculate combined scores."""
        results = []

        for doc in documents:
            # Calculate scores
            vector_score = doc.metadata.get("score", 0)
            keyword_score = self._calculate_keyword_score(
                query,
                doc.page_content,
            )

            # Combine scores
            combined_score = (
                self._vector_weight * vector_score
                + self._keyword_weight * keyword_score
            )

            if score_threshold and combined_score < score_threshold:
                continue

            doc.metadata["combined_score"] = combined_score
            results.append(doc)

        # Sort by combined score
        results.sort(
            key=lambda x: x.metadata["combined_score"],
            reverse=True,
        )

        return results

    def _calculate_keyword_score(
        self,
        query: str,
        content: str,
    ) -> float:
        """TF-IDF based keyword score."""
        # Tokenize with Jieba (supports Chinese)
        query_tokens = set(jieba.cut(query.lower()))
        content_tokens = list(jieba.cut(content.lower()))

        if not query_tokens or not content_tokens:
            return 0.0

        # Calculate TF
        content_token_set = set(content_tokens)
        matching_tokens = query_tokens & content_token_set

        if not matching_tokens:
            return 0.0

        # TF score
        tf_score = len(matching_tokens) / len(query_tokens)

        # Simple IDF approximation
        idf_score = sum(
            1.0 / (content_tokens.count(token) + 1)
            for token in matching_tokens
        )

        return tf_score * idf_score / len(matching_tokens)
```

### 9.3 Model-Based Reranking

```python
class RerankModelRunner(BaseRerankRunner):
    """Use dedicated reranking model."""

    def __init__(self, provider: str, model: str):
        self._provider = provider
        self._model = model
        self._rerank_model = self._get_model()

    def run(
        self,
        query: str,
        documents: list[Document],
        score_threshold: float | None = None,
    ) -> list[Document]:
        """Rerank using model API."""
        if not documents:
            return []

        # Call reranking API
        rerank_results = self._rerank_model.invoke(
            query=query,
            documents=[doc.page_content for doc in documents],
        )

        # Apply new scores
        for result, doc in zip(rerank_results, documents):
            doc.metadata["rerank_score"] = result.score

        # Filter and sort
        results = [
            doc for doc in documents
            if (
                score_threshold is None
                or doc.metadata["rerank_score"] >= score_threshold
            )
        ]
        results.sort(
            key=lambda x: x.metadata["rerank_score"],
            reverse=True,
        )

        return results
```

## 10. RAG in Workflows

### 10.1 Knowledge Retrieval Node

**Reference**: `/api/core/workflow/nodes/knowledge_retrieval/knowledge_retrieval_node.py`

```python
class KnowledgeRetrievalNode(Node[KnowledgeRetrievalNodeData]):
    """Workflow node for knowledge retrieval."""

    node_type = NodeType.KNOWLEDGE_RETRIEVAL

    def _run(self) -> NodeRunResult:
        """Execute retrieval in workflow."""
        # Get query from variable pool
        query = self.graph_runtime_state.variable_pool.get(
            self.node_data.query_variable
        )

        # Get dataset configs
        dataset_configs = self._get_dataset_configs()

        # Execute retrieval
        dataset_retrieval = DatasetRetrieval()
        results = dataset_retrieval.retrieve(
            dataset_configs=dataset_configs,
            query=query,
            invoke_from=self._invoke_from,
        )

        # Format output
        outputs = {
            "result": [doc.page_content for doc in results],
            "metadata": [doc.metadata for doc in results],
        }

        return NodeRunResult(
            status=WorkflowNodeExecutionStatus.SUCCEEDED,
            outputs=outputs,
        )
```

### 10.2 Knowledge Index Node

**Reference**: `/api/core/workflow/nodes/knowledge_index/knowledge_index_node.py`

```python
class KnowledgeIndexNode(Node[KnowledgeIndexNodeData]):
    """Workflow node for document indexing."""

    node_type = NodeType.KNOWLEDGE_INDEX

    def _run(self) -> Generator[NodeEventBase, None, None]:
        """Index documents in workflow."""
        # Get document content from variable pool
        content = self.graph_runtime_state.variable_pool.get(
            self.node_data.content_variable
        )

        # Get target dataset
        dataset = Dataset.query.get(self.node_data.dataset_id)

        # Create index processor
        index_processor = IndexProcessorFactory(
            dataset.chunk_structure
        ).init_index_processor()

        # Create document
        document = Document(
            page_content=content,
            metadata=self._build_metadata(),
        )

        # Extract and chunk
        chunks = index_processor.transform([document])

        # Index into vector store
        index_processor.index(
            dataset=dataset,
            document=document,
            chunks=chunks,
        )

        yield NodeRunSucceededEvent(
            outputs={"indexed_chunks": len(chunks)},
        )
```

## 11. Complete RAG Flow Diagram

```
┌───────────────────────────────────────────────────────────────────────────┐
│                          DOCUMENT INGESTION                                │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌────────────┐     ┌────────────┐     ┌────────────┐     ┌────────────┐ │
│  │   Upload   │────►│ Extractor  │────►│  Cleaner   │────►│  Splitter  │ │
│  │   File     │     │(PDF, Word) │     │(Normalize) │     │ (Chunks)   │ │
│  └────────────┘     └────────────┘     └────────────┘     └─────┬──────┘ │
│                                                                  │        │
└──────────────────────────────────────────────────────────────────┼────────┘
                                                                   │
                                                                   ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                            EMBEDDING                                       │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌────────────┐     ┌────────────┐     ┌────────────┐                    │
│  │   Chunks   │────►│ CacheCheck │────►│  Embedding │                    │
│  │   (Text)   │     │ (DB Cache) │     │   Model    │                    │
│  └────────────┘     └────────────┘     └─────┬──────┘                    │
│                                              │                            │
│                                              ▼                            │
│                                       ┌────────────┐                      │
│                                       │ L2 Normalize│                      │
│                                       │  Vectors   │                      │
│                                       └─────┬──────┘                      │
│                                              │                            │
└──────────────────────────────────────────────┼────────────────────────────┘
                                               │
                                               ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                           VECTOR STORAGE                                   │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                      Vector Factory                                 │  │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐   │  │
│  │  │Weavi.│ │Pinec.│ │PGVec.│ │Qdrant│ │Milvus│ │Chroma│ │ 30+  │   │  │
│  │  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘   │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
└──────────────────────────────────────────────────────────────────────────┘

                            ═══════════════════════
                               QUERY TIME
                            ═══════════════════════

┌───────────────────────────────────────────────────────────────────────────┐
│                            RETRIEVAL                                       │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌────────────┐     ┌────────────┐     ┌────────────────────────────────┐│
│  │   Query    │────►│  Embed     │────►│        Search Strategy         ││
│  │   (User)   │     │  Query     │     │  ┌──────────────────────────┐ ││
│  └────────────┘     └────────────┘     │  │ Semantic  Full-Text Hybrid│ ││
│                                         │  │  Search    Search  Search │ ││
│                                         │  └──────────────────────────┘ ││
│                                         └───────────────┬───────────────┘│
│                                                         │                 │
│                                                         ▼                 │
│                                         ┌───────────────────────────────┐│
│                                         │        Results                ││
│                                         │   (Ranked Documents)          ││
│                                         └───────────────┬───────────────┘│
│                                                         │                 │
└─────────────────────────────────────────────────────────┼─────────────────┘
                                                          │
                                                          ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                           RERANKING                                        │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                     DataPostProcessor                               │  │
│  │  ┌──────────────────────┐  ┌──────────────────────────────────┐   │  │
│  │  │   Weight Reranker    │  │      Model Reranker              │   │  │
│  │  │ (Keyword + Vector)   │  │   (Cohere, Jina, etc.)          │   │  │
│  │  └──────────────────────┘  └──────────────────────────────────┘   │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                    │                                      │
│                                    ▼                                      │
│                         ┌─────────────────────┐                          │
│                         │  Final Results      │                          │
│                         │ (Top-K Documents)   │                          │
│                         └─────────────────────┘                          │
│                                                                            │
└───────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │   LLM Generation    │
                         │ (Context-Augmented) │
                         └─────────────────────┘
```

## 12. Key Design Patterns

| Pattern | Location | Purpose |
|---------|----------|---------|
| **Factory** | VectorFactory | Create vector store instances |
| **Strategy** | RetrievalMethod | Multiple retrieval strategies |
| **Template** | TextSplitter | Chunking algorithms |
| **Decorator** | CacheEmbedding | Caching layer |
| **Chain of Responsibility** | Pipeline stages | Sequential processing |
| **Parallel Execution** | ThreadPoolExecutor | Multi-dataset retrieval |
| **Abstract Factory** | IndexProcessorFactory | Index structure creation |

---

*Tiếp theo: [Frontend Web](./05-frontend-web.md)*
