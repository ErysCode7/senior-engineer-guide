# Content Recommendations and Personalization

## Overview

Recommendation systems are essential for modern applications, driving user engagement by suggesting relevant content, products, or connections. This tutorial covers collaborative filtering, content-based filtering, hybrid approaches, and machine learning integration for building personalized recommendation engines.

## Use Cases

- **E-commerce**: Product recommendations based on browsing/purchase history
- **Streaming Services**: Movie, music, and content suggestions
- **Social Media**: Friend suggestions and content feeds
- **News/Blogs**: Article recommendations based on reading history
- **Education**: Course and learning material suggestions

## Recommendation Algorithms

### 1. Collaborative Filtering

Collaborative filtering recommends items based on similar users' preferences.

#### User-Based Collaborative Filtering

```typescript
// src/services/collaborative-filtering.service.ts
import { Injectable } from "@nestjs/common";

interface UserRating {
  userId: string;
  itemId: string;
  rating: number;
}

@Injectable()
export class CollaborativeFilteringService {
  // Calculate cosine similarity between two users
  private calculateCosineSimilarity(
    user1Ratings: Map<string, number>,
    user2Ratings: Map<string, number>
  ): number {
    const commonItems = Array.from(user1Ratings.keys()).filter((itemId) =>
      user2Ratings.has(itemId)
    );

    if (commonItems.length === 0) return 0;

    let dotProduct = 0;
    let magnitude1 = 0;
    let magnitude2 = 0;

    for (const itemId of commonItems) {
      const rating1 = user1Ratings.get(itemId);
      const rating2 = user2Ratings.get(itemId);

      dotProduct += rating1 * rating2;
      magnitude1 += rating1 ** 2;
      magnitude2 += rating2 ** 2;
    }

    if (magnitude1 === 0 || magnitude2 === 0) return 0;

    return dotProduct / (Math.sqrt(magnitude1) * Math.sqrt(magnitude2));
  }

  // Find K nearest neighbors
  async findSimilarUsers(
    targetUserId: string,
    allRatings: UserRating[],
    k: number = 10
  ): Promise<Array<{ userId: string; similarity: number }>> {
    // Group ratings by user
    const userRatingsMap = new Map<string, Map<string, number>>();

    for (const rating of allRatings) {
      if (!userRatingsMap.has(rating.userId)) {
        userRatingsMap.set(rating.userId, new Map());
      }
      userRatingsMap.get(rating.userId).set(rating.itemId, rating.rating);
    }

    const targetUserRatings = userRatingsMap.get(targetUserId);
    if (!targetUserRatings) return [];

    // Calculate similarity with all other users
    const similarities: Array<{ userId: string; similarity: number }> = [];

    for (const [userId, ratings] of userRatingsMap.entries()) {
      if (userId === targetUserId) continue;

      const similarity = this.calculateCosineSimilarity(
        targetUserRatings,
        ratings
      );

      if (similarity > 0) {
        similarities.push({ userId, similarity });
      }
    }

    // Sort by similarity and return top K
    return similarities.sort((a, b) => b.similarity - a.similarity).slice(0, k);
  }

  // Predict rating for an item
  async predictRating(
    targetUserId: string,
    itemId: string,
    allRatings: UserRating[],
    k: number = 10
  ): Promise<number> {
    const similarUsers = await this.findSimilarUsers(
      targetUserId,
      allRatings,
      k
    );

    let weightedSum = 0;
    let similaritySum = 0;

    for (const { userId, similarity } of similarUsers) {
      const userRating = allRatings.find(
        (r) => r.userId === userId && r.itemId === itemId
      );

      if (userRating) {
        weightedSum += similarity * userRating.rating;
        similaritySum += similarity;
      }
    }

    return similaritySum > 0 ? weightedSum / similaritySum : 0;
  }

  // Recommend items for user
  async recommendItems(
    targetUserId: string,
    allRatings: UserRating[],
    candidateItemIds: string[],
    topN: number = 10
  ): Promise<Array<{ itemId: string; predictedRating: number }>> {
    const recommendations: Array<{ itemId: string; predictedRating: number }> =
      [];

    // Get items already rated by user
    const ratedItems = new Set(
      allRatings.filter((r) => r.userId === targetUserId).map((r) => r.itemId)
    );

    // Predict ratings for unrated items
    for (const itemId of candidateItemIds) {
      if (ratedItems.has(itemId)) continue;

      const predictedRating = await this.predictRating(
        targetUserId,
        itemId,
        allRatings
      );

      if (predictedRating > 0) {
        recommendations.push({ itemId, predictedRating });
      }
    }

    // Sort by predicted rating and return top N
    return recommendations
      .sort((a, b) => b.predictedRating - a.predictedRating)
      .slice(0, topN);
  }
}
```

#### Item-Based Collaborative Filtering

```typescript
// src/services/item-based-cf.service.ts
import { Injectable } from "@nestjs/common";

@Injectable()
export class ItemBasedCFService {
  // Calculate similarity between two items
  private calculateItemSimilarity(
    item1Ratings: Map<string, number>,
    item2Ratings: Map<string, number>
  ): number {
    const commonUsers = Array.from(item1Ratings.keys()).filter((userId) =>
      item2Ratings.has(userId)
    );

    if (commonUsers.length === 0) return 0;

    let dotProduct = 0;
    let magnitude1 = 0;
    let magnitude2 = 0;

    for (const userId of commonUsers) {
      const rating1 = item1Ratings.get(userId);
      const rating2 = item2Ratings.get(userId);

      dotProduct += rating1 * rating2;
      magnitude1 += rating1 ** 2;
      magnitude2 += rating2 ** 2;
    }

    return dotProduct / (Math.sqrt(magnitude1) * Math.sqrt(magnitude2));
  }

  // Build item similarity matrix
  async buildSimilarityMatrix(
    allRatings: UserRating[]
  ): Promise<Map<string, Map<string, number>>> {
    // Group ratings by item
    const itemRatingsMap = new Map<string, Map<string, number>>();

    for (const rating of allRatings) {
      if (!itemRatingsMap.has(rating.itemId)) {
        itemRatingsMap.set(rating.itemId, new Map());
      }
      itemRatingsMap.get(rating.itemId).set(rating.userId, rating.rating);
    }

    const similarityMatrix = new Map<string, Map<string, number>>();

    // Calculate pairwise similarities
    const itemIds = Array.from(itemRatingsMap.keys());

    for (let i = 0; i < itemIds.length; i++) {
      const item1Id = itemIds[i];
      const similarities = new Map<string, number>();

      for (let j = 0; j < itemIds.length; j++) {
        if (i === j) continue;

        const item2Id = itemIds[j];
        const similarity = this.calculateItemSimilarity(
          itemRatingsMap.get(item1Id),
          itemRatingsMap.get(item2Id)
        );

        if (similarity > 0) {
          similarities.set(item2Id, similarity);
        }
      }

      similarityMatrix.set(item1Id, similarities);
    }

    return similarityMatrix;
  }

  // Recommend based on similar items
  async recommendSimilarItems(
    itemId: string,
    similarityMatrix: Map<string, Map<string, number>>,
    topN: number = 10
  ): Promise<Array<{ itemId: string; similarity: number }>> {
    const similarities = similarityMatrix.get(itemId);

    if (!similarities) return [];

    return Array.from(similarities.entries())
      .map(([itemId, similarity]) => ({ itemId, similarity }))
      .sort((a, b) => b.similarity - a.similarity)
      .slice(0, topN);
  }
}
```

### 2. Content-Based Filtering

Recommends items similar to those the user has liked, based on item features.

```typescript
// src/services/content-based-filtering.service.ts
import { Injectable } from "@nestjs/common";

interface Item {
  id: string;
  title: string;
  description: string;
  category: string;
  tags: string[];
  metadata: Record<string, any>;
}

interface UserProfile {
  userId: string;
  preferredCategories: Map<string, number>; // category -> weight
  preferredTags: Map<string, number>; // tag -> weight
}

@Injectable()
export class ContentBasedFilteringService {
  // Build user profile from interaction history
  async buildUserProfile(
    userId: string,
    interactedItems: Item[],
    ratings: Map<string, number> // itemId -> rating
  ): Promise<UserProfile> {
    const categoryWeights = new Map<string, number>();
    const tagWeights = new Map<string, number>();

    for (const item of interactedItems) {
      const rating = ratings.get(item.id) || 0;

      // Weight by rating (higher ratings = more important)
      const weight = rating / 5; // Normalize to 0-1

      // Accumulate category weights
      const currentCategoryWeight = categoryWeights.get(item.category) || 0;
      categoryWeights.set(item.category, currentCategoryWeight + weight);

      // Accumulate tag weights
      for (const tag of item.tags) {
        const currentTagWeight = tagWeights.get(tag) || 0;
        tagWeights.set(tag, currentTagWeight + weight);
      }
    }

    return {
      userId,
      preferredCategories: categoryWeights,
      preferredTags: tagWeights,
    };
  }

  // Calculate relevance score for an item
  private calculateRelevanceScore(item: Item, profile: UserProfile): number {
    let score = 0;

    // Category score
    const categoryWeight = profile.preferredCategories.get(item.category) || 0;
    score += categoryWeight * 0.3; // 30% weight

    // Tags score
    let tagScore = 0;
    for (const tag of item.tags) {
      const tagWeight = profile.preferredTags.get(tag) || 0;
      tagScore += tagWeight;
    }
    score += (tagScore / Math.max(item.tags.length, 1)) * 0.7; // 70% weight

    return score;
  }

  // Recommend items based on user profile
  async recommendItems(
    profile: UserProfile,
    candidateItems: Item[],
    topN: number = 10
  ): Promise<Array<{ item: Item; score: number }>> {
    const recommendations = candidateItems.map((item) => ({
      item,
      score: this.calculateRelevanceScore(item, profile),
    }));

    return recommendations.sort((a, b) => b.score - a.score).slice(0, topN);
  }

  // TF-IDF based text similarity
  async calculateTextSimilarity(
    targetItem: Item,
    candidateItems: Item[]
  ): Promise<Array<{ item: Item; similarity: number }>> {
    // Simple TF-IDF implementation
    const allTexts = [targetItem, ...candidateItems].map((item) =>
      `${item.title} ${item.description} ${item.tags.join(" ")}`.toLowerCase()
    );

    const targetText = allTexts[0];
    const targetWords = new Set(targetText.split(/\s+/));

    const similarities = candidateItems.map((item, index) => {
      const candidateText = allTexts[index + 1];
      const candidateWords = new Set(candidateText.split(/\s+/));

      // Calculate Jaccard similarity
      const intersection = new Set(
        Array.from(targetWords).filter((word) => candidateWords.has(word))
      );
      const union = new Set([...targetWords, ...candidateWords]);

      const similarity = intersection.size / union.size;

      return { item, similarity };
    });

    return similarities.sort((a, b) => b.similarity - a.similarity);
  }
}
```

### 3. Hybrid Approach

Combines collaborative and content-based filtering for better recommendations.

```typescript
// src/services/hybrid-recommendation.service.ts
import { Injectable } from "@nestjs/common";
import { CollaborativeFilteringService } from "./collaborative-filtering.service";
import { ContentBasedFilteringService } from "./content-based-filtering.service";

interface HybridRecommendation {
  itemId: string;
  score: number;
  sources: {
    collaborative: number;
    contentBased: number;
  };
}

@Injectable()
export class HybridRecommendationService {
  constructor(
    private readonly cfService: CollaborativeFilteringService,
    private readonly cbService: ContentBasedFilteringService
  ) {}

  async recommendItems(
    userId: string,
    allRatings: UserRating[],
    items: Map<string, Item>,
    userProfile: UserProfile,
    topN: number = 10,
    weights: { collaborative: number; contentBased: number } = {
      collaborative: 0.6,
      contentBased: 0.4,
    }
  ): Promise<HybridRecommendation[]> {
    // Get collaborative filtering recommendations
    const cfRecommendations = await this.cfService.recommendItems(
      userId,
      allRatings,
      Array.from(items.keys()),
      topN * 2 // Get more to ensure diversity
    );

    // Get content-based recommendations
    const cbRecommendations = await this.cbService.recommendItems(
      userProfile,
      Array.from(items.values()),
      topN * 2
    );

    // Combine scores
    const combinedScores = new Map<string, HybridRecommendation>();

    // Add CF scores
    for (const rec of cfRecommendations) {
      combinedScores.set(rec.itemId, {
        itemId: rec.itemId,
        score: rec.predictedRating * weights.collaborative,
        sources: {
          collaborative: rec.predictedRating,
          contentBased: 0,
        },
      });
    }

    // Add CB scores
    for (const rec of cbRecommendations) {
      const existing = combinedScores.get(rec.item.id);

      if (existing) {
        existing.score += rec.score * weights.contentBased;
        existing.sources.contentBased = rec.score;
      } else {
        combinedScores.set(rec.item.id, {
          itemId: rec.item.id,
          score: rec.score * weights.contentBased,
          sources: {
            collaborative: 0,
            contentBased: rec.score,
          },
        });
      }
    }

    // Sort by combined score
    return Array.from(combinedScores.values())
      .sort((a, b) => b.score - a.score)
      .slice(0, topN);
  }
}
```

## Machine Learning Integration

### Using TensorFlow.js for Recommendations

```typescript
// src/services/ml-recommendation.service.ts
import { Injectable } from "@nestjs/common";
import * as tf from "@tensorflow/tfjs-node";

@Injectable()
export class MLRecommendationService {
  private model: tf.LayersModel;

  async trainModel(
    userFeatures: number[][],
    itemFeatures: number[][],
    ratings: number[]
  ) {
    // Create a neural network for collaborative filtering
    const userInput = tf.input({ shape: [userFeatures[0].length] });
    const itemInput = tf.input({ shape: [itemFeatures[0].length] });

    // User embedding
    let userLayer = tf.layers
      .dense({ units: 64, activation: "relu" })
      .apply(userInput);
    userLayer = tf.layers
      .dense({ units: 32, activation: "relu" })
      .apply(userLayer);

    // Item embedding
    let itemLayer = tf.layers
      .dense({ units: 64, activation: "relu" })
      .apply(itemInput);
    itemLayer = tf.layers
      .dense({ units: 32, activation: "relu" })
      .apply(itemLayer);

    // Concatenate and predict
    const concatenated = tf.layers.concatenate().apply([userLayer, itemLayer]);
    let combined = tf.layers
      .dense({ units: 64, activation: "relu" })
      .apply(concatenated);
    combined = tf.layers.dropout({ rate: 0.2 }).apply(combined);
    const output = tf.layers
      .dense({ units: 1, activation: "sigmoid" })
      .apply(combined);

    this.model = tf.model({
      inputs: [userInput, itemInput],
      outputs: output as any,
    });

    this.model.compile({
      optimizer: tf.train.adam(0.001),
      loss: "meanSquaredError",
      metrics: ["mae"],
    });

    // Convert data to tensors
    const userTensor = tf.tensor2d(userFeatures);
    const itemTensor = tf.tensor2d(itemFeatures);
    const ratingTensor = tf.tensor2d(ratings.map((r) => [r / 5])); // Normalize to 0-1

    // Train the model
    await this.model.fit([userTensor, itemTensor], ratingTensor, {
      epochs: 50,
      batchSize: 32,
      validationSplit: 0.2,
      callbacks: {
        onEpochEnd: (epoch, logs) => {
          console.log(`Epoch ${epoch}: loss = ${logs.loss}, mae = ${logs.mae}`);
        },
      },
    });

    // Cleanup
    userTensor.dispose();
    itemTensor.dispose();
    ratingTensor.dispose();
  }

  async predict(
    userFeatures: number[],
    itemFeatures: number[]
  ): Promise<number> {
    const userTensor = tf.tensor2d([userFeatures]);
    const itemTensor = tf.tensor2d([itemFeatures]);

    const prediction = this.model.predict([
      userTensor,
      itemTensor,
    ]) as tf.Tensor;
    const rating = (await prediction.data())[0] * 5; // Denormalize from 0-1 to 0-5

    // Cleanup
    userTensor.dispose();
    itemTensor.dispose();
    prediction.dispose();

    return rating;
  }

  async saveModel(path: string) {
    await this.model.save(`file://${path}`);
  }

  async loadModel(path: string) {
    this.model = await tf.loadLayersModel(`file://${path}/model.json`);
  }
}
```

## Real-Time Recommendations API

```typescript
// src/controllers/recommendations.controller.ts
import { Controller, Get, Post, Body, Param, Query } from "@nestjs/common";
import { HybridRecommendationService } from "../services/hybrid-recommendation.service";
import { RecommendationCacheService } from "../services/recommendation-cache.service";

@Controller("api/recommendations")
export class RecommendationsController {
  constructor(
    private readonly hybridService: HybridRecommendationService,
    private readonly cacheService: RecommendationCacheService
  ) {}

  @Get("users/:userId")
  async getUserRecommendations(
    @Param("userId") userId: string,
    @Query("limit") limit: number = 10,
    @Query("refresh") refresh: boolean = false
  ) {
    // Check cache first
    if (!refresh) {
      const cached = await this.cacheService.get(userId);
      if (cached) {
        return {
          recommendations: cached.slice(0, limit),
          fromCache: true,
        };
      }
    }

    // Generate fresh recommendations
    const recommendations = await this.hybridService.recommendItems(
      userId,
      // ... pass required data
      limit
    );

    // Cache results
    await this.cacheService.set(userId, recommendations, 3600); // 1 hour TTL

    return {
      recommendations,
      fromCache: false,
    };
  }

  @Get("items/:itemId/similar")
  async getSimilarItems(
    @Param("itemId") itemId: string,
    @Query("limit") limit: number = 10
  ) {
    // Get items similar to the given item
    const similar = await this.hybridService.findSimilarItems(itemId, limit);

    return { similarItems: similar };
  }

  @Post("feedback")
  async recordFeedback(
    @Body()
    body: {
      userId: string;
      itemId: string;
      action: "like" | "dislike" | "view" | "purchase";
      rating?: number;
    }
  ) {
    // Record user interaction for model retraining
    await this.feedbackService.record(body);

    // Invalidate cache
    await this.cacheService.invalidate(body.userId);

    return { success: true };
  }
}
```

## A/B Testing Recommendations

```typescript
// src/services/ab-test-recommendation.service.ts
import { Injectable } from "@nestjs/common";

interface Experiment {
  id: string;
  variants: Array<{
    id: string;
    weight: number;
    algorithm: "collaborative" | "content" | "hybrid" | "ml";
  }>;
}

@Injectable()
export class ABTestRecommendationService {
  private experiments: Map<string, Experiment> = new Map();

  registerExperiment(experiment: Experiment) {
    this.experiments.set(experiment.id, experiment);
  }

  assignVariant(userId: string, experimentId: string): string {
    const experiment = this.experiments.get(experimentId);
    if (!experiment) return "control";

    // Deterministic assignment based on user ID hash
    const hash = this.hashCode(userId);
    const normalizedHash = Math.abs(hash % 100);

    let cumulativeWeight = 0;
    for (const variant of experiment.variants) {
      cumulativeWeight += variant.weight * 100;
      if (normalizedHash < cumulativeWeight) {
        return variant.id;
      }
    }

    return experiment.variants[0].id;
  }

  private hashCode(str: string): number {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i);
      hash = (hash << 5) - hash + char;
    }
    return hash;
  }

  async trackExperimentMetric(
    experimentId: string,
    variantId: string,
    metric: string,
    value: number
  ) {
    // Track metrics for analysis
    await this.metricsService.record({
      experimentId,
      variantId,
      metric,
      value,
      timestamp: new Date(),
    });
  }
}
```

## Best Practices

### 1. **Handle Cold Start Problem**

- New users: Use popular items, demographic info
- New items: Use content-based features
- Implement hybrid approaches

### 2. **Diversity and Serendipity**

- Don't just show similar items
- Include some exploratory recommendations
- Use randomization with high-score items

### 3. **Real-Time Updates**

- Cache recommendations
- Invalidate cache on new interactions
- Update models periodically

### 4. **Privacy and Ethics**

- Be transparent about data usage
- Allow users to control recommendations
- Avoid filter bubbles

### 5. **Performance Optimization**

- Pre-compute similarities offline
- Use caching extensively
- Implement batch processing

### 6. **Evaluation Metrics**

- Precision@K and Recall@K
- Mean Average Precision (MAP)
- Normalized Discounted Cumulative Gain (NDCG)
- Click-through rate (CTR)
- Conversion rate

### 7. **A/B Testing**

- Test different algorithms
- Monitor business metrics
- Iterate based on results

## Summary

Building effective recommendation systems requires:

- Understanding different algorithms (collaborative, content-based, hybrid)
- Balancing accuracy with diversity
- Optimizing for performance and scale
- Handling cold start problems
- Continuous evaluation and improvement
- Privacy-conscious design
- A/B testing and experimentation

These practices ensure recommendations drive engagement while respecting user preferences and privacy.
