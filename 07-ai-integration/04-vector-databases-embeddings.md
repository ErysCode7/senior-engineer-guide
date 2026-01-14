# Vector Databases and Embeddings

## Overview

Vector databases store and query high-dimensional vector embeddings, enabling semantic search, recommendation systems, and Retrieval Augmented Generation (RAG). This tutorial covers embedding generation, vector database integration (Pinecone, Weaviate, Chroma), similarity search, and building RAG applications.

## Use Cases

- **Semantic Search**: Find documents by meaning, not just keywords
- **RAG Applications**: Enhance AI responses with relevant context
- **Recommendation Systems**: Find similar items based on embeddings
- **Duplicate Detection**: Identify similar content automatically
- **Question Answering**: Build knowledge bases with semantic retrieval

## Understanding Vector Embeddings

Vector embeddings are numerical representations of text, images, or other data that capture semantic meaning.

```typescript
// src/services/embedding.service.ts
import { Injectable } from "@nestjs/common";
import { OpenAI } from "openai";

@Injectable()
export class EmbeddingService {
  private openai: OpenAI;

  constructor() {
    this.openai = new OpenAI({
      apiKey: process.env.OPENAI_API_KEY,
    });
  }

  async generateEmbedding(text: string): Promise<number[]> {
    try {
      const response = await this.openai.embeddings.create({
        model: "text-embedding-3-small", // 1536 dimensions, cost-effective
        input: text,
      });

      return response.data[0].embedding;
    } catch (error) {
      console.error("Embedding generation error:", error);
      throw new Error("Failed to generate embedding");
    }
  }

  async generateEmbeddings(texts: string[]): Promise<number[][]> {
    // Batch processing for efficiency
    const response = await this.openai.embeddings.create({
      model: "text-embedding-3-small",
      input: texts,
    });

    return response.data.map((item) => item.embedding);
  }

  // Calculate cosine similarity between two vectors
  calculateCosineSimilarity(vector1: number[], vector2: number[]): number {
    if (vector1.length !== vector2.length) {
      throw new Error("Vectors must have the same length");
    }

    let dotProduct = 0;
    let magnitude1 = 0;
    let magnitude2 = 0;

    for (let i = 0; i < vector1.length; i++) {
      dotProduct += vector1[i] * vector2[i];
      magnitude1 += vector1[i] ** 2;
      magnitude2 += vector2[i] ** 2;
    }

    const magnitude = Math.sqrt(magnitude1) * Math.sqrt(magnitude2);

    return magnitude === 0 ? 0 : dotProduct / magnitude;
  }

  // Find most similar vectors
  findMostSimilar(
    queryVector: number[],
    vectors: Array<{ id: string; vector: number[]; metadata?: any }>,
    topK: number = 5
  ): Array<{ id: string; similarity: number; metadata?: any }> {
    const similarities = vectors.map((item) => ({
      id: item.id,
      similarity: this.calculateCosineSimilarity(queryVector, item.vector),
      metadata: item.metadata,
    }));

    return similarities
      .sort((a, b) => b.similarity - a.similarity)
      .slice(0, topK);
  }
}
```

## Pinecone Integration

Pinecone is a fully managed vector database optimized for similarity search.

### Setup

```bash
npm install @pinecone-database/pinecone
```

### Basic Operations

```typescript
// src/services/pinecone.service.ts
import { Injectable } from "@nestjs/common";
import { Pinecone } from "@pinecone-database/pinecone";

interface VectorMetadata {
  text: string;
  source?: string;
  category?: string;
  [key: string]: any;
}

@Injectable()
export class PineconeService {
  private client: Pinecone;
  private indexName: string = "my-index";

  constructor() {
    this.client = new Pinecone({
      apiKey: process.env.PINECONE_API_KEY,
    });
  }

  async createIndex(dimension: number = 1536) {
    try {
      await this.client.createIndex({
        name: this.indexName,
        dimension,
        metric: "cosine",
        spec: {
          serverless: {
            cloud: "aws",
            region: "us-east-1",
          },
        },
      });

      console.log(`Index ${this.indexName} created successfully`);
    } catch (error) {
      if (error.message.includes("already exists")) {
        console.log(`Index ${this.indexName} already exists`);
      } else {
        throw error;
      }
    }
  }

  async upsertVectors(
    vectors: Array<{
      id: string;
      values: number[];
      metadata?: VectorMetadata;
    }>
  ) {
    const index = this.client.index(this.indexName);

    // Batch upsert (Pinecone handles large batches efficiently)
    const batchSize = 100;

    for (let i = 0; i < vectors.length; i += batchSize) {
      const batch = vectors.slice(i, i + batchSize);
      await index.upsert(batch);
    }

    console.log(`Upserted ${vectors.length} vectors`);
  }

  async queryVectors(
    queryVector: number[],
    topK: number = 5,
    filter?: Record<string, any>
  ) {
    const index = this.client.index(this.indexName);

    const results = await index.query({
      vector: queryVector,
      topK,
      includeMetadata: true,
      filter,
    });

    return results.matches.map((match) => ({
      id: match.id,
      score: match.score,
      metadata: match.metadata as VectorMetadata,
    }));
  }

  async deleteVectors(ids: string[]) {
    const index = this.client.index(this.indexName);
    await index.deleteMany(ids);
  }

  async deleteAll() {
    const index = this.client.index(this.indexName);
    await index.deleteAll();
  }
}

// Usage example
const vectors = [
  {
    id: "doc1",
    values: await embeddingService.generateEmbedding(
      "Machine learning is fascinating"
    ),
    metadata: { text: "Machine learning is fascinating", category: "AI" },
  },
  {
    id: "doc2",
    values: await embeddingService.generateEmbedding(
      "Python is a great programming language"
    ),
    metadata: {
      text: "Python is a great programming language",
      category: "Programming",
    },
  },
];

await pineconeService.upsertVectors(vectors);

const queryVector = await embeddingService.generateEmbedding(
  "artificial intelligence"
);
const results = await pineconeService.queryVectors(queryVector, 5);
```

## Weaviate Integration

Weaviate is an open-source vector database with built-in ML models.

### Setup

```bash
npm install weaviate-ts-client
```

### Basic Operations

```typescript
// src/services/weaviate.service.ts
import { Injectable } from "@nestjs/common";
import weaviate, { WeaviateClient } from "weaviate-ts-client";

@Injectable()
export class WeaviateService {
  private client: WeaviateClient;
  private className: string = "Document";

  constructor() {
    this.client = weaviate.client({
      scheme: "http",
      host: process.env.WEAVIATE_HOST || "localhost:8080",
      headers: {
        "X-OpenAI-Api-Key": process.env.OPENAI_API_KEY,
      },
    });
  }

  async createSchema() {
    const schema = {
      class: this.className,
      vectorizer: "text2vec-openai",
      moduleConfig: {
        "text2vec-openai": {
          model: "text-embedding-3-small",
          vectorizeClassName: false,
        },
      },
      properties: [
        {
          name: "content",
          dataType: ["text"],
          description: "The document content",
        },
        {
          name: "title",
          dataType: ["text"],
          description: "The document title",
        },
        {
          name: "category",
          dataType: ["text"],
          description: "The document category",
        },
        {
          name: "source",
          dataType: ["text"],
          description: "The document source",
        },
      ],
    };

    try {
      await this.client.schema.classCreator().withClass(schema).do();
      console.log("Schema created successfully");
    } catch (error) {
      console.log("Schema might already exist:", error.message);
    }
  }

  async addDocument(
    id: string,
    content: string,
    metadata: {
      title?: string;
      category?: string;
      source?: string;
    } = {}
  ) {
    await this.client.data
      .creator()
      .withClassName(this.className)
      .withId(id)
      .withProperties({
        content,
        ...metadata,
      })
      .do();
  }

  async addDocuments(
    documents: Array<{
      id: string;
      content: string;
      metadata?: Record<string, string>;
    }>
  ) {
    // Batch import
    let batcher = this.client.batch.objectsBatcher();

    for (const doc of documents) {
      batcher = batcher.withObject({
        class: this.className,
        id: doc.id,
        properties: {
          content: doc.content,
          ...doc.metadata,
        },
      });
    }

    await batcher.do();
    console.log(`Added ${documents.length} documents`);
  }

  async semanticSearch(query: string, limit: number = 5, filter?: any) {
    let queryBuilder = this.client.graphql
      .get()
      .withClassName(this.className)
      .withFields("content title category source _additional { id distance }")
      .withNearText({ concepts: [query] })
      .withLimit(limit);

    if (filter) {
      queryBuilder = queryBuilder.withWhere(filter);
    }

    const result = await queryBuilder.do();

    return result.data.Get[this.className].map((item) => ({
      id: item._additional.id,
      distance: item._additional.distance,
      content: item.content,
      title: item.title,
      category: item.category,
      source: item.source,
    }));
  }

  async hybridSearch(
    query: string,
    alpha: number = 0.5, // 0 = pure keyword, 1 = pure vector
    limit: number = 5
  ) {
    const result = await this.client.graphql
      .get()
      .withClassName(this.className)
      .withFields("content title category _additional { id score }")
      .withHybrid({
        query,
        alpha,
      })
      .withLimit(limit)
      .do();

    return result.data.Get[this.className];
  }

  async deleteDocument(id: string) {
    await this.client.data
      .deleter()
      .withClassName(this.className)
      .withId(id)
      .do();
  }
}
```

## ChromaDB Integration

ChromaDB is a lightweight, open-source embedding database.

### Setup

```bash
npm install chromadb
```

### Basic Operations

```typescript
// src/services/chroma.service.ts
import { Injectable } from "@nestjs/common";
import { ChromaClient, OpenAIEmbeddingFunction } from "chromadb";

@Injectable()
export class ChromaService {
  private client: ChromaClient;
  private embeddingFunction: OpenAIEmbeddingFunction;
  private collectionName: string = "documents";

  constructor() {
    this.client = new ChromaClient({
      path: process.env.CHROMA_URL || "http://localhost:8000",
    });

    this.embeddingFunction = new OpenAIEmbeddingFunction({
      openai_api_key: process.env.OPENAI_API_KEY,
      openai_model: "text-embedding-3-small",
    });
  }

  async createCollection() {
    try {
      const collection = await this.client.createCollection({
        name: this.collectionName,
        embeddingFunction: this.embeddingFunction,
        metadata: { description: "Document embeddings" },
      });

      return collection;
    } catch (error) {
      // Collection might already exist
      return this.client.getCollection({
        name: this.collectionName,
        embeddingFunction: this.embeddingFunction,
      });
    }
  }

  async addDocuments(
    documents: Array<{
      id: string;
      content: string;
      metadata?: Record<string, any>;
    }>
  ) {
    const collection = await this.createCollection();

    await collection.add({
      ids: documents.map((doc) => doc.id),
      documents: documents.map((doc) => doc.content),
      metadatas: documents.map((doc) => doc.metadata || {}),
    });

    console.log(`Added ${documents.length} documents to Chroma`);
  }

  async query(
    queryText: string,
    nResults: number = 5,
    filter?: Record<string, any>
  ) {
    const collection = await this.createCollection();

    const results = await collection.query({
      queryTexts: [queryText],
      nResults,
      where: filter,
    });

    return {
      ids: results.ids[0],
      distances: results.distances[0],
      documents: results.documents[0],
      metadatas: results.metadatas[0],
    };
  }

  async deleteDocuments(ids: string[]) {
    const collection = await this.createCollection();
    await collection.delete({ ids });
  }

  async deleteCollection() {
    await this.client.deleteCollection({ name: this.collectionName });
  }

  async peek(limit: number = 10) {
    const collection = await this.createCollection();
    return collection.peek({ limit });
  }
}
```

## Building a RAG Application

Retrieval Augmented Generation combines vector search with LLM generation.

```typescript
// src/services/rag.service.ts
import { Injectable } from "@nestjs/common";
import { EmbeddingService } from "./embedding.service";
import { PineconeService } from "./pinecone.service";
import { OpenAI } from "openai";

@Injectable()
export class RAGService {
  private openai: OpenAI;

  constructor(
    private readonly embeddingService: EmbeddingService,
    private readonly pineconeService: PineconeService
  ) {
    this.openai = new OpenAI({
      apiKey: process.env.OPENAI_API_KEY,
    });
  }

  async indexDocuments(
    documents: Array<{
      id: string;
      content: string;
      metadata?: Record<string, any>;
    }>
  ) {
    // Split documents into chunks for better retrieval
    const chunks = this.splitIntoChunks(documents);

    // Generate embeddings
    const embeddings = await this.embeddingService.generateEmbeddings(
      chunks.map((chunk) => chunk.content)
    );

    // Upsert to vector database
    const vectors = chunks.map((chunk, index) => ({
      id: chunk.id,
      values: embeddings[index],
      metadata: {
        content: chunk.content,
        ...chunk.metadata,
      },
    }));

    await this.pineconeService.upsertVectors(vectors);

    return { indexed: chunks.length };
  }

  private splitIntoChunks(
    documents: Array<{ id: string; content: string; metadata?: any }>,
    chunkSize: number = 500,
    overlap: number = 50
  ) {
    const chunks = [];

    for (const doc of documents) {
      const words = doc.content.split(" ");

      for (let i = 0; i < words.length; i += chunkSize - overlap) {
        const chunkWords = words.slice(i, i + chunkSize);
        const chunkContent = chunkWords.join(" ");

        chunks.push({
          id: `${doc.id}-chunk-${chunks.length}`,
          content: chunkContent,
          metadata: {
            ...doc.metadata,
            parentDocId: doc.id,
            chunkIndex: Math.floor(i / (chunkSize - overlap)),
          },
        });

        if (i + chunkSize >= words.length) break;
      }
    }

    return chunks;
  }

  async query(
    question: string,
    options?: {
      topK?: number;
      temperature?: number;
      systemPrompt?: string;
    }
  ): Promise<{ answer: string; sources: any[]; context: string[] }> {
    // 1. Generate embedding for the question
    const questionEmbedding = await this.embeddingService.generateEmbedding(
      question
    );

    // 2. Retrieve relevant context from vector database
    const topK = options?.topK || 5;
    const results = await this.pineconeService.queryVectors(
      questionEmbedding,
      topK
    );

    // 3. Extract context
    const context = results.map((result) => result.metadata.content);

    // 4. Build prompt with context
    const systemPrompt =
      options?.systemPrompt ||
      `You are a helpful assistant that answers questions based on the provided context. 
If the answer cannot be found in the context, say "I don't have enough information to answer that question."`;

    const userPrompt = `Context:
${context.map((c, i) => `[${i + 1}] ${c}`).join("\n\n")}

Question: ${question}

Please provide a detailed answer based on the context above. Cite the relevant context numbers in your answer.`;

    // 5. Generate answer using LLM
    const completion = await this.openai.chat.completions.create({
      model: "gpt-4-turbo-preview",
      messages: [
        { role: "system", content: systemPrompt },
        { role: "user", content: userPrompt },
      ],
      temperature: options?.temperature || 0.3,
      max_tokens: 1000,
    });

    const answer = completion.choices[0].message.content;

    return {
      answer,
      sources: results.map((r) => ({
        id: r.id,
        score: r.score,
        metadata: r.metadata,
      })),
      context,
    };
  }

  async conversationalRAG(
    question: string,
    conversationHistory: Array<{ role: "user" | "assistant"; content: string }>
  ) {
    // Generate embedding
    const embedding = await this.embeddingService.generateEmbedding(question);

    // Retrieve context
    const results = await this.pineconeService.queryVectors(embedding, 5);
    const context = results.map((r) => r.metadata.content).join("\n\n");

    // Build messages with history
    const messages = [
      {
        role: "system" as const,
        content: `You are a helpful assistant. Use the following context to answer questions:\n\n${context}`,
      },
      ...conversationHistory,
      {
        role: "user" as const,
        content: question,
      },
    ];

    // Generate response
    const completion = await this.openai.chat.completions.create({
      model: "gpt-4-turbo-preview",
      messages,
      temperature: 0.7,
    });

    return {
      answer: completion.choices[0].message.content,
      sources: results,
    };
  }
}

// Controller
@Controller("api/rag")
export class RAGController {
  constructor(private readonly ragService: RAGService) {}

  @Post("index")
  async indexDocuments(@Body() body: { documents: any[] }) {
    return this.ragService.indexDocuments(body.documents);
  }

  @Post("query")
  async query(@Body() body: { question: string; options?: any }) {
    return this.ragService.query(body.question, body.options);
  }

  @Post("chat")
  async chat(@Body() body: { question: string; history: any[] }) {
    return this.ragService.conversationalRAG(body.question, body.history);
  }
}
```

## Semantic Search Implementation

```typescript
// src/services/semantic-search.service.ts
import { Injectable } from "@nestjs/common";
import { EmbeddingService } from "./embedding.service";
import { PineconeService } from "./pinecone.service";

interface SearchOptions {
  limit?: number;
  filter?: Record<string, any>;
  includeScores?: boolean;
  minScore?: number;
}

@Injectable()
export class SemanticSearchService {
  constructor(
    private readonly embeddingService: EmbeddingService,
    private readonly pineconeService: PineconeService
  ) {}

  async search(query: string, options: SearchOptions = {}) {
    const { limit = 10, filter, includeScores = true, minScore = 0 } = options;

    // Generate query embedding
    const queryEmbedding = await this.embeddingService.generateEmbedding(query);

    // Search vector database
    const results = await this.pineconeService.queryVectors(
      queryEmbedding,
      limit,
      filter
    );

    // Filter by minimum score
    const filteredResults = results.filter(
      (result) => result.score >= minScore
    );

    return {
      query,
      results: filteredResults.map((result) => ({
        id: result.id,
        ...(includeScores && { score: result.score }),
        ...result.metadata,
      })),
      total: filteredResults.length,
    };
  }

  async multiSearch(queries: string[], options: SearchOptions = {}) {
    const results = await Promise.all(
      queries.map((query) => this.search(query, options))
    );

    // Combine and deduplicate results
    const combinedResults = new Map();

    for (const result of results) {
      for (const item of result.results) {
        if (!combinedResults.has(item.id)) {
          combinedResults.set(item.id, item);
        } else {
          // Keep the one with higher score
          const existing = combinedResults.get(item.id);
          if (item.score > existing.score) {
            combinedResults.set(item.id, item);
          }
        }
      }
    }

    return {
      queries,
      results: Array.from(combinedResults.values())
        .sort((a, b) => b.score - a.score)
        .slice(0, options.limit || 10),
    };
  }

  async findSimilar(documentId: string, limit: number = 5) {
    // Get the document's embedding
    const results = await this.pineconeService.queryVectors(
      [], // Would need to fetch the original embedding
      limit + 1, // +1 because the document itself will be in results
      { id: { $ne: documentId } } // Exclude the original document
    );

    return results.slice(0, limit);
  }
}
```

## Best Practices

### 1. **Chunking Strategy**

- Split long documents into smaller chunks (500-1000 tokens)
- Use overlap between chunks for context continuity
- Preserve semantic boundaries (paragraphs, sections)

### 2. **Embedding Model Selection**

- `text-embedding-3-small`: Cost-effective, good performance (1536 dimensions)
- `text-embedding-3-large`: Higher quality (3072 dimensions)
- Consider domain-specific models for specialized use cases

### 3. **Metadata and Filtering**

- Store relevant metadata with vectors
- Use filters to narrow search scope
- Include timestamps for freshness ranking

### 4. **Performance Optimization**

- Batch embed multiple texts together
- Cache embeddings when possible
- Use appropriate vector dimensions

### 5. **RAG Quality**

- Use clear, specific prompts
- Cite sources in generated answers
- Implement fallback for low-confidence answers
- Monitor hallucination rates

### 6. **Cost Management**

- Cache embeddings for frequently accessed content
- Use smaller embedding models when appropriate
- Monitor API usage and set limits

### 7. **Evaluation**

- Test retrieval accuracy regularly
- Measure answer quality
- A/B test different chunk sizes and retrieval strategies

## Summary

Vector databases and embeddings enable powerful semantic capabilities:

- Store and query high-dimensional vector representations
- Build semantic search that understands meaning
- Implement RAG for context-aware AI responses
- Create recommendation systems based on similarity
- Handle multi-modal data (text, images, etc.)

Key takeaways:

- Choose the right vector database for your needs
- Optimize chunking and embedding strategies
- Implement proper metadata and filtering
- Monitor performance and costs
- Continuously evaluate and improve retrieval quality

These technologies form the foundation of modern AI-powered applications.
