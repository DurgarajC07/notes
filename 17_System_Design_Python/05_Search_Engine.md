# System Design: Search Engine

## 📖 Problem Statement

Design a search engine system like Elasticsearch, Solr, or Algolia that:

- Indexes millions of documents efficiently
- Provides fast full-text search (< 100ms)
- Ranks results by relevance (TF-IDF, BM25)
- Supports autocomplete and fuzzy search
- Handles typos and synonyms
- Provides faceted search (filters)
- Scales horizontally with sharding

## 🎯 Requirements

### Functional Requirements

1. Index documents with text content
2. Full-text search with ranking
3. Autocomplete suggestions
4. Fuzzy matching (handle typos)
5. Faceted search (filter by category, date, etc.)
6. Highlighting of matched terms
7. Pagination of results

### Non-Functional Requirements

1. **Low Latency**: <100ms search response time
2. **High Throughput**: 1000+ searches/second
3. **Scalability**: Handle billions of documents
4. **Availability**: 99.9% uptime
5. **Indexing Speed**: Index 10K documents/second
6. **Relevance**: High-quality ranking

## 📊 Architecture

```
                 ┌────────────┐
                 │   Client   │
                 └──────┬─────┘
                        │
                        ▼
              ┌──────────────────┐
              │  Search API      │
              │  (FastAPI/Flask) │
              └────────┬─────────┘
                       │
         ┌─────────────┼─────────────┐
         │             │             │
         ▼             ▼             ▼
    ┌────────┐    ┌────────┐    ┌────────┐
    │ Shard 1│    │ Shard 2│    │ Shard 3│
    │ (Index)│    │ (Index)│    │ (Index)│
    └────────┘    └────────┘    └────────┘
         │             │             │
         └─────────────┼─────────────┘
                       │
                       ▼
              ┌──────────────────┐
              │  Result Merger   │
              │  (Rank & Sort)   │
              └──────────────────┘
```

## 🔑 Core Components

### 1. Inverted Index

```python
from typing import Dict, Set, List
from collections import defaultdict
import re

class InvertedIndex:
    """
    Inverted index for full-text search

    Maps: term -> set of document IDs containing term

    Example:
        "hello" -> {1, 5, 10}
        "world" -> {1, 8, 15}
    """

    def __init__(self):
        """Initialize inverted index"""
        self.index: Dict[str, Set[int]] = defaultdict(set)
        self.documents: Dict[int, str] = {}

    def tokenize(self, text: str) -> List[str]:
        """
        Tokenize text into words

        Args:
            text: Input text

        Returns:
            List of lowercase tokens
        """
        # Convert to lowercase and split on non-alphanumeric
        tokens = re.findall(r'\w+', text.lower())
        return tokens

    def add_document(self, doc_id: int, text: str):
        """
        Add document to index

        Args:
            doc_id: Document ID
            text: Document text
        """
        self.documents[doc_id] = text

        # Tokenize and add to index
        tokens = self.tokenize(text)

        for token in tokens:
            self.index[token].add(doc_id)

    def search(self, query: str) -> Set[int]:
        """
        Search for documents matching query

        Args:
            query: Search query

        Returns:
            Set of matching document IDs
        """
        tokens = self.tokenize(query)

        if not tokens:
            return set()

        # Start with documents containing first token
        result = self.index[tokens[0]].copy()

        # Intersect with documents containing other tokens
        for token in tokens[1:]:
            result &= self.index[token]

        return result

    def search_or(self, query: str) -> Set[int]:
        """
        Search with OR logic (any term matches)

        Args:
            query: Search query

        Returns:
            Set of matching document IDs
        """
        tokens = self.tokenize(query)

        result = set()

        for token in tokens:
            result |= self.index[token]

        return result

# Example usage
index = InvertedIndex()

# Add documents
index.add_document(1, "Python is a programming language")
index.add_document(2, "Java is also a programming language")
index.add_document(3, "Python is easy to learn")
index.add_document(4, "JavaScript is used for web development")

# Search
results = index.search("python programming")
print(f"AND search: {results}")  # {1}

results = index.search_or("python java")
print(f"OR search: {results}")  # {1, 2, 3}
```

### 2. TF-IDF Ranking

```python
import math
from typing import Dict, List, Tuple

class TFIDFRanker:
    """
    TF-IDF (Term Frequency - Inverse Document Frequency) ranker

    TF = (# of times term appears in doc) / (total terms in doc)
    IDF = log(total docs / docs containing term)
    TF-IDF = TF * IDF
    """

    def __init__(self, index: InvertedIndex):
        """
        Args:
            index: Inverted index
        """
        self.index = index

    def calculate_tf(self, term: str, doc_id: int) -> float:
        """
        Calculate term frequency

        Args:
            term: Search term
            doc_id: Document ID

        Returns:
            Term frequency (0.0 to 1.0)
        """
        text = self.index.documents[doc_id]
        tokens = self.index.tokenize(text)

        if not tokens:
            return 0.0

        term_count = tokens.count(term)
        return term_count / len(tokens)

    def calculate_idf(self, term: str) -> float:
        """
        Calculate inverse document frequency

        Args:
            term: Search term

        Returns:
            IDF value
        """
        total_docs = len(self.index.documents)
        docs_with_term = len(self.index.index[term])

        if docs_with_term == 0:
            return 0.0

        return math.log(total_docs / docs_with_term)

    def calculate_tfidf(self, term: str, doc_id: int) -> float:
        """
        Calculate TF-IDF score

        Args:
            term: Search term
            doc_id: Document ID

        Returns:
            TF-IDF score
        """
        tf = self.calculate_tf(term, doc_id)
        idf = self.calculate_idf(term)

        return tf * idf

    def rank(self, query: str, doc_ids: Set[int]) -> List[Tuple[int, float]]:
        """
        Rank documents by TF-IDF score

        Args:
            query: Search query
            doc_ids: Document IDs to rank

        Returns:
            List of (doc_id, score) sorted by score
        """
        query_terms = self.index.tokenize(query)

        scores = {}

        for doc_id in doc_ids:
            # Sum TF-IDF for all query terms
            score = sum(
                self.calculate_tfidf(term, doc_id)
                for term in query_terms
            )
            scores[doc_id] = score

        # Sort by score (descending)
        ranked = sorted(scores.items(), key=lambda x: x[1], reverse=True)

        return ranked

# Example usage
index = InvertedIndex()
index.add_document(1, "Python is a programming language")
index.add_document(2, "Java is also a programming language")
index.add_document(3, "Python is easy to learn Python Python")

ranker = TFIDFRanker(index)

# Search and rank
doc_ids = index.search_or("python programming")
ranked = ranker.rank("python programming", doc_ids)

print("Ranked results:")
for doc_id, score in ranked:
    print(f"  Doc {doc_id}: {score:.4f} - {index.documents[doc_id][:50]}")
```

### 3. Autocomplete with Trie

```python
class TrieNode:
    """Node in trie data structure"""

    def __init__(self):
        self.children: Dict[str, 'TrieNode'] = {}
        self.is_end_of_word = False
        self.frequency = 0  # For ranking suggestions

class Trie:
    """
    Trie (prefix tree) for autocomplete

    - Efficient prefix matching
    - O(m) search where m = query length
    - Common for autocomplete, spell check
    """

    def __init__(self):
        """Initialize trie"""
        self.root = TrieNode()

    def insert(self, word: str, frequency: int = 1):
        """
        Insert word into trie

        Args:
            word: Word to insert
            frequency: Word frequency (for ranking)
        """
        node = self.root

        for char in word.lower():
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]

        node.is_end_of_word = True
        node.frequency = frequency

    def search(self, word: str) -> bool:
        """
        Check if word exists

        Args:
            word: Word to search

        Returns:
            True if word exists
        """
        node = self._find_node(word)
        return node is not None and node.is_end_of_word

    def starts_with(self, prefix: str) -> bool:
        """
        Check if prefix exists

        Args:
            prefix: Prefix to check

        Returns:
            True if any word starts with prefix
        """
        return self._find_node(prefix) is not None

    def _find_node(self, prefix: str) -> TrieNode:
        """Find node for prefix"""
        node = self.root

        for char in prefix.lower():
            if char not in node.children:
                return None
            node = node.children[char]

        return node

    def autocomplete(self, prefix: str, max_results: int = 10) -> List[Tuple[str, int]]:
        """
        Get autocomplete suggestions

        Args:
            prefix: Search prefix
            max_results: Maximum results to return

        Returns:
            List of (word, frequency) tuples
        """
        # Find prefix node
        node = self._find_node(prefix)

        if not node:
            return []

        # Collect all words with this prefix
        results = []
        self._collect_words(node, prefix, results)

        # Sort by frequency and limit
        results.sort(key=lambda x: x[1], reverse=True)

        return results[:max_results]

    def _collect_words(self, node: TrieNode, prefix: str, results: List[Tuple[str, int]]):
        """
        Recursively collect all words from node

        Args:
            node: Current node
            prefix: Current prefix
            results: List to append results
        """
        if node.is_end_of_word:
            results.append((prefix, node.frequency))

        for char, child_node in node.children.items():
            self._collect_words(child_node, prefix + char, results)

# Example usage
trie = Trie()

# Insert words with frequencies
words = [
    ("python", 1000),
    ("programming", 800),
    ("program", 700),
    ("programmer", 600),
    ("product", 500),
    ("production", 400)
]

for word, freq in words:
    trie.insert(word, freq)

# Autocomplete
suggestions = trie.autocomplete("prog", max_results=5)
print("Suggestions for 'prog':")
for word, freq in suggestions:
    print(f"  {word} (frequency: {freq})")
```

### 4. Fuzzy Search (Levenshtein Distance)

```python
def levenshtein_distance(s1: str, s2: str) -> int:
    """
    Calculate edit distance between two strings

    Args:
        s1: First string
        s2: Second string

    Returns:
        Minimum number of edits (insert/delete/replace)
    """
    m, n = len(s1), len(s2)

    # Create DP table
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    # Initialize base cases
    for i in range(m + 1):
        dp[i][0] = i

    for j in range(n + 1):
        dp[0][j] = j

    # Fill DP table
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if s1[i-1] == s2[j-1]:
                dp[i][j] = dp[i-1][j-1]
            else:
                dp[i][j] = 1 + min(
                    dp[i-1][j],    # Delete
                    dp[i][j-1],    # Insert
                    dp[i-1][j-1]   # Replace
                )

    return dp[m][n]

class FuzzySearcher:
    """
    Fuzzy search with edit distance

    - Handles typos
    - Finds similar words
    """

    def __init__(self, index: InvertedIndex, max_distance: int = 2):
        """
        Args:
            index: Inverted index
            max_distance: Maximum edit distance
        """
        self.index = index
        self.max_distance = max_distance

        # Build vocabulary
        self.vocabulary = set(self.index.index.keys())

    def find_similar_terms(self, term: str) -> List[Tuple[str, int]]:
        """
        Find terms similar to input

        Args:
            term: Search term

        Returns:
            List of (similar_term, distance)
        """
        similar = []

        for vocab_term in self.vocabulary:
            distance = levenshtein_distance(term, vocab_term)

            if distance <= self.max_distance:
                similar.append((vocab_term, distance))

        # Sort by distance
        similar.sort(key=lambda x: x[1])

        return similar

    def fuzzy_search(self, query: str) -> Set[int]:
        """
        Search with typo tolerance

        Args:
            query: Search query (may have typos)

        Returns:
            Set of matching document IDs
        """
        tokens = self.index.tokenize(query)

        result = set()

        for token in tokens:
            # Find similar terms
            similar_terms = self.find_similar_terms(token)

            # Union documents for all similar terms
            for similar_term, _ in similar_terms:
                result |= self.index.index[similar_term]

        return result

# Example usage
index = InvertedIndex()
index.add_document(1, "Python programming is fun")
index.add_document(2, "Java programming is powerful")
index.add_document(3, "JavaScript is versatile")

fuzzy = FuzzySearcher(index, max_distance=2)

# Search with typo
results = fuzzy.fuzzy_search("pythn progaming")
print(f"Fuzzy search results: {results}")

# Find similar terms
similar = fuzzy.find_similar_terms("pythn")
print(f"Similar to 'pythn': {similar}")
```

## 🔍 Complete Search Engine

```python
from typing import Optional
import json

class SearchEngine:
    """
    Complete search engine

    Combines:
    - Inverted index
    - TF-IDF ranking
    - Autocomplete
    - Fuzzy search
    """

    def __init__(self):
        """Initialize search engine"""
        self.index = InvertedIndex()
        self.ranker = TFIDFRanker(self.index)
        self.trie = Trie()
        self.fuzzy = None  # Initialized after indexing

    def index_document(self, doc_id: int, text: str):
        """
        Index document

        Args:
            doc_id: Document ID
            text: Document text
        """
        # Add to inverted index
        self.index.add_document(doc_id, text)

        # Add terms to trie
        tokens = self.index.tokenize(text)
        for token in set(tokens):
            self.trie.insert(token)

    def build_index(self):
        """Build search structures after indexing"""
        self.fuzzy = FuzzySearcher(self.index)

    def search(
        self,
        query: str,
        use_fuzzy: bool = False,
        max_results: int = 10
    ) -> List[Dict]:
        """
        Search documents

        Args:
            query: Search query
            use_fuzzy: Enable fuzzy matching
            max_results: Maximum results

        Returns:
            List of search results with scores
        """
        # Find matching documents
        if use_fuzzy and self.fuzzy:
            doc_ids = self.fuzzy.fuzzy_search(query)
        else:
            doc_ids = self.index.search_or(query)

        if not doc_ids:
            return []

        # Rank by TF-IDF
        ranked = self.ranker.rank(query, doc_ids)

        # Format results
        results = []
        for doc_id, score in ranked[:max_results]:
            results.append({
                'doc_id': doc_id,
                'text': self.index.documents[doc_id],
                'score': score
            })

        return results

    def autocomplete(self, prefix: str, max_results: int = 10) -> List[str]:
        """
        Get autocomplete suggestions

        Args:
            prefix: Search prefix
            max_results: Maximum results

        Returns:
            List of suggestions
        """
        suggestions = self.trie.autocomplete(prefix, max_results)
        return [word for word, _ in suggestions]

# Example usage
engine = SearchEngine()

# Index documents
documents = [
    (1, "Python is a high-level programming language"),
    (2, "Java is an object-oriented programming language"),
    (3, "JavaScript is used for web development"),
    (4, "Python is great for data science and machine learning"),
    (5, "Programming languages are tools for developers")
]

for doc_id, text in documents:
    engine.index_document(doc_id, text)

engine.build_index()

# Search
results = engine.search("python programming", max_results=5)
print("Search results:")
for result in results:
    print(f"  Doc {result['doc_id']} (score: {result['score']:.4f})")
    print(f"    {result['text'][:60]}...")

# Autocomplete
suggestions = engine.autocomplete("prog")
print(f"\nAutocomplete for 'prog': {suggestions}")

# Fuzzy search
results = engine.search("pythn programing", use_fuzzy=True)
print(f"\nFuzzy search found {len(results)} results")
```

## 🚀 Integration with Elasticsearch

```python
from elasticsearch import Elasticsearch
from typing import List, Dict

class ElasticsearchEngine:
    """
    Search engine using Elasticsearch

    - Production-ready
    - Distributed, scalable
    - Advanced features (analyzers, aggregations)
    """

    def __init__(self, hosts: List[str] = None):
        """
        Args:
            hosts: Elasticsearch hosts
        """
        self.es = Elasticsearch(hosts or ['localhost:9200'])
        self.index_name = "documents"

    def create_index(self):
        """Create index with mappings"""
        mappings = {
            "mappings": {
                "properties": {
                    "text": {
                        "type": "text",
                        "analyzer": "standard"
                    },
                    "title": {
                        "type": "text",
                        "analyzer": "standard"
                    },
                    "category": {
                        "type": "keyword"
                    },
                    "created_at": {
                        "type": "date"
                    }
                }
            }
        }

        # Create index
        if not self.es.indices.exists(index=self.index_name):
            self.es.indices.create(index=self.index_name, body=mappings)

    def index_document(self, doc_id: str, document: Dict):
        """
        Index document

        Args:
            doc_id: Document ID
            document: Document data
        """
        self.es.index(
            index=self.index_name,
            id=doc_id,
            body=document
        )

    def search(
        self,
        query: str,
        filters: Dict = None,
        from_: int = 0,
        size: int = 10
    ) -> Dict:
        """
        Search documents

        Args:
            query: Search query
            filters: Optional filters
            from_: Offset for pagination
            size: Number of results

        Returns:
            Search results
        """
        # Build query
        search_body = {
            "from": from_,
            "size": size,
            "query": {
                "bool": {
                    "must": [
                        {
                            "multi_match": {
                                "query": query,
                                "fields": ["title^2", "text"],
                                "fuzziness": "AUTO"
                            }
                        }
                    ]
                }
            },
            "highlight": {
                "fields": {
                    "text": {},
                    "title": {}
                }
            }
        }

        # Add filters
        if filters:
            filter_clauses = []

            for field, value in filters.items():
                filter_clauses.append({
                    "term": {field: value}
                })

            search_body["query"]["bool"]["filter"] = filter_clauses

        # Execute search
        response = self.es.search(
            index=self.index_name,
            body=search_body
        )

        return response

    def autocomplete(self, prefix: str, field: str = "text") -> List[str]:
        """
        Autocomplete suggestions

        Args:
            prefix: Search prefix
            field: Field to search

        Returns:
            List of suggestions
        """
        search_body = {
            "suggest": {
                "autocomplete": {
                    "prefix": prefix,
                    "completion": {
                        "field": field,
                        "size": 10,
                        "skip_duplicates": True
                    }
                }
            }
        }

        response = self.es.search(
            index=self.index_name,
            body=search_body
        )

        suggestions = []
        for option in response["suggest"]["autocomplete"][0]["options"]:
            suggestions.append(option["text"])

        return suggestions

# Example usage
es_engine = ElasticsearchEngine()
es_engine.create_index()

# Index documents
documents = [
    {
        "title": "Python Tutorial",
        "text": "Learn Python programming from basics to advanced",
        "category": "programming",
        "created_at": "2024-01-01"
    },
    {
        "title": "Java Guide",
        "text": "Comprehensive guide to Java programming",
        "category": "programming",
        "created_at": "2024-01-02"
    }
]

for i, doc in enumerate(documents, 1):
    es_engine.index_document(str(i), doc)

# Search
results = es_engine.search("python programming", filters={"category": "programming"})
print(f"Found {results['hits']['total']['value']} results")
```

## ❓ Interview Questions

### Q1: How does inverted index work?

**Answer**:

- Maps terms to documents containing them
- Example: "hello" → {doc1, doc3, doc7}
- Fast lookups: O(1) per term
- Space trade-off: Stores all unique terms

### Q2: TF-IDF vs BM25?

**Answer**:

- **TF-IDF**: Term Frequency × Inverse Document Frequency
- **BM25**: Improved version, saturates TF (diminishing returns)
- **BM25 better for**: Long documents, repeated terms
- **Modern engines**: Use BM25 or neural embeddings

### Q3: How to handle typos?

**Answer**:

1. **Fuzzy matching**: Levenshtein distance
2. **Phonetic matching**: Soundex, Metaphone
3. **N-grams**: Break into character sequences
4. **ML models**: Learn common typos

### Q4: How to scale search to billions of documents?

**Answer**:

1. **Sharding**: Partition index across nodes
2. **Replication**: Copies for availability
3. **Caching**: Cache popular queries
4. **Pre-computation**: Pre-calculate rankings

## 📚 Summary

**Key Takeaways**:

1. **Inverted Index**: Core data structure
2. **TF-IDF**: Classic ranking algorithm
3. **Trie**: Efficient autocomplete
4. **Fuzzy Search**: Handle typos
5. **Elasticsearch**: Production-ready solution
6. **Sharding**: Scale horizontally
7. **Caching**: Cache query results
8. **Analyzers**: Tokenization, stemming, synonyms
9. **Facets**: Filter by attributes
10. **Neural Search**: Modern approach with embeddings

Search engines power modern applications!
