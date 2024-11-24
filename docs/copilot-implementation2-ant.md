# Perplexica Copilot Implementation Guide

## Overview
This document provides a comprehensive guide for implementing Copilot functionality in Perplexica, enhancing the application's search capabilities through self-defining and self-refining processes to deliver more accurate and detailed search results.

## Core Architecture

### 1. Primary Components

#### File Structure
```
/home/perplexica/src/agents/
├── webSearchAgent.ts         # Main search functionality
└── suggestionGeneratorAgent.ts # Follow-up suggestions
```

#### Key Features
- Self-Define Phase: Query analysis and search planning
- Self-Refine Phase: Result evaluation and refinement
- Enhanced Search Logic: Multiple search variations and iterative improvement
- Smart Suggestions: Context-aware follow-up queries

### 2. Prompt Templates Implementation

#### A. Search Retriever Prompt
```typescript
const basicSearchRetrieverPrompt = `
You will be given a conversation below and a follow-up question. Your task is to rephrase the follow-up question for web searching.

### Self-Define Phase:
1. **Understand the Query:**
   - Identify key components
   - Determine special handling needs

2. **Formulate Rephrased Questions:**
   - Create multiple search variations
   - Format for links/summarization if needed

3. **Advanced Search Logic:**
   - Synthesize comprehensive search query
   - Combine different query aspects

4. **Finalize Question:**
   - Present as single query or XML blocks

[Example format templates follow...]

Conversation:
{chat_history}

Follow-up question: {query}
Rephrased question:
`;

#### B. Web Search Response Prompt
```typescript
const basicWebSearchResponsePrompt = `
You are Perplexica, specialized in comprehensive web search responses.

### Quality Guidelines:
1. **Interaction Quality:**
   - Relevance, Clarity, Helpfulness, User Experience

2. **Content Quality:**
   - Depth, Accuracy, Authority

### Response Structure:
1. Introduction
2. Main Content
3. Supporting Information
4. Conclusion

[Detailed formatting and citation guidelines follow...]

<context>
{context}
</context>
`;

#### C. Suggestion Generator Prompt
```typescript
const suggestionGeneratorPrompt = `
Generate 4-5 relevant search suggestions based on conversation.

### Quality Guidelines:
1. Relevance
2. Diversity
3. Helpfulness
4. Length Appropriateness

[Detailed suggestion generation guidelines follow...]

Conversation:
{chat_history}
`;

## Implementation Steps

### 1. Initial Setup

1. **Backup Original Files**
```bash
cp /home/perplexica/src/agents/webSearchAgent.ts webSearchAgent.ts.backup
cp /home/perplexica/src/agents/suggestionGeneratorAgent.ts suggestionGeneratorAgent.ts.backup
```

2. **Update Dependencies**
```typescript
import {
  BaseMessage,
  PromptTemplate,
  ChatPromptTemplate,
  MessagesPlaceholder,
  RunnableSequence,
  RunnableMap,
  RunnableLambda,
  StringOutputParser
} from '@langchain/core';
```

### 2. Core Functionality Implementation

#### A. Iterative Search Function
```typescript
const iterateSearch = async (input, maxIterations = 3) => {
  let iteration = 0;
  let result = await runSearch(input);

  while (iteration < maxIterations) {
    const refinedQuery = await llm.predict(
      `Evaluate results and suggest improvements: ${JSON.stringify(result.docs)}`
    );
    const refinedResult = await runSearch(refinedQuery);
    result.docs = result.docs.concat(refinedResult.docs);
    iteration++;
  }

  return result;
};
```

#### B. Search Chain Creation
```typescript
const createBasicWebSearchRetrieverChain = (llm: BaseChatModel) => {
  return RunnableSequence.from([
    PromptTemplate.fromTemplate(basicSearchRetrieverPrompt),
    llm,
    strParser,
    RunnableLambda.from(input => iterateSearch(input.query))
  ]);
};
```

#### C. Response Chain Creation
```typescript
const createBasicWebSearchAnsweringChain = (
  llm: BaseChatModel,
  embeddings: Embeddings,
) => {
  const basicWebSearchRetrieverChain = createBasicWebSearchRetrieverChain(llm);
  
  // Document processing implementation
  const processDocs = async (docs: Document[]) => {
    return docs
      .map((_, index) => `${index + 1}. ${docs[index].pageContent}`)
      .join('\n');
  };

  // Document reranking implementation
  const rerankDocs = async ({
    query,
    docs,
  }: {
    query: string;
    docs: Document[];
  }) => {
    if (docs.length === 0) {
      return docs;
    }

    // Filter out empty documents
    const docsWithContent = docs.filter(
      (doc) => doc.pageContent && doc.pageContent.length > 0,
    );

    // Get embeddings for documents and query
    const [docEmbeddings, queryEmbedding] = await Promise.all([
      embeddings.embedDocuments(docsWithContent.map((doc) => doc.pageContent)),
      embeddings.embedQuery(query),
    ]);

    // Calculate similarity scores
    const similarity = docEmbeddings.map((docEmbedding, i) => {
      const sim = computeSimilarity(queryEmbedding, docEmbedding);
      return {
        index: i,
        similarity: sim,
      };
    });

    // Sort and filter documents by similarity
    const sortedDocs = similarity
      .sort((a, b) => b.similarity - a.similarity)
      .filter((sim) => sim.similarity > 0.5)
      .slice(0, 15)
      .map((sim) => docsWithContent[sim.index]);

    return sortedDocs;
  };

  // Create the complete chain
  return RunnableSequence.from([
    RunnableMap.from({
      query: (input: BasicChainInput) => input.query,
      chat_history: (input: BasicChainInput) => input.chat_history,
      context: RunnableSequence.from([
        (input) => ({
          query: input.query,
          chat_history: formatChatHistoryAsString(input.chat_history),
        }),
        basicWebSearchRetrieverChain
          .pipe(rerankDocs)
          .withConfig({
            runName: 'FinalSourceRetriever',
          })
          .pipe(processDocs),
      ]),
    }),
    ChatPromptTemplate.fromMessages([
      ['system', basicWebSearchResponsePrompt],
      new MessagesPlaceholder('chat_history'),
      ['user', '{query}'],
    ]),
    llm,
    strParser,
  ]).withConfig({
    runName: 'FinalResponseGenerator',
  });
};
```

### 3. Integration Points

#### A. Embeddings Configuration
```typescript
import { LlamaEmbeddings } from '@langchain/community/embeddings/llama';
import { OpenAIEmbeddings } from '@langchain/openai';
import { HuggingFaceInferenceEmbeddings } from '@langchain/community/embeddings/hf';
import { CohereEmbeddings } from '@langchain/cohere';

interface EmbeddingConfig {
  modelName?: string;
  maxRetries?: number;
  timeout?: number;
  batchSize?: number;
}

const configureEmbeddings = (modelType: string, config: EmbeddingConfig = {}) => {
  const defaultConfig = {
    maxRetries: 3,
    timeout: 30000,
    batchSize: 512
  };

  const finalConfig = { ...defaultConfig, ...config };

  switch(modelType.toLowerCase()) {
    case 'llama2:13b':
    case 'llama2:70b':
      return new LlamaEmbeddings({
        modelName: finalConfig.modelName || 'llama2',
        maxRetries: finalConfig.maxRetries,
        timeout: finalConfig.timeout
      });

    case 'openai':
      return new OpenAIEmbeddings({
        modelName: finalConfig.modelName || 'text-embedding-ada-002',
        maxRetries: finalConfig.maxRetries,
        timeout: finalConfig.timeout,
        batchSize: finalConfig.batchSize
      });

    case 'huggingface':
      return new HuggingFaceInferenceEmbeddings({
        modelName: finalConfig.modelName || 'sentence-transformers/all-MiniLM-L6-v2',
        maxRetries: finalConfig.maxRetries,
        timeout: finalConfig.timeout
      });

    case 'cohere':
      return new CohereEmbeddings({
        modelName: finalConfig.modelName || 'large',
        maxRetries: finalConfig.maxRetries,
        timeout: finalConfig.timeout,
        batchSize: finalConfig.batchSize
      });

    default:
      // Fallback to Llama embeddings
      return new LlamaEmbeddings({
        modelName: 'llama2',
        maxRetries: finalConfig.maxRetries,
        timeout: finalConfig.timeout
      });
  }
};

// Usage example:
const embeddings = configureEmbeddings('llama2:13b', {
  maxRetries: 5,
  timeout: 45000,
  batchSize: 256
});
```

#### B. Error Handling
```typescript
interface SearchError extends Error {
  code?: string;
  details?: any;
  retry?: boolean;
}

class SearchHandlingError extends Error implements SearchError {
  code: string;
  details: any;
  retry: boolean;

  constructor(message: string, code: string, details?: any, retry: boolean = false) {
    super(message);
    this.name = 'SearchHandlingError';
    this.code = code;
    this.details = details;
    this.retry = retry;
  }
}

const handleSearchErrors = async (err: SearchError): Promise<void> => {
  // Log error details
  console.error(`Search error encountered: ${err.message}`);
  if (err.details) {
    console.error('Error details:', err.details);
  }

  // Handle specific error types
  switch(err.code) {
    case 'EMBEDDINGS_ERROR':
      // Handle embedding-related errors
      await handleEmbeddingError(err);
      break;

    case 'SEARCH_TIMEOUT':
      // Handle timeout errors
      await handleTimeoutError(err);
      break;

    case 'INVALID_QUERY':
      // Handle invalid query errors
      await handleInvalidQueryError(err);
      break;

    case 'SOURCE_UNREACHABLE':
      // Handle source access errors
      await handleSourceError(err);
      break;

    default:
      // Handle unknown errors
      await handleUnknownError(err);
  }

  // Attempt recovery if possible
  if (err.retry) {
    await attemptErrorRecovery(err);
  }
};

// Error type-specific handlers
const handleEmbeddingError = async (err: SearchError): Promise<void> => {
  console.error('Embedding error:', err.message);
  // Attempt to reconfigure embeddings
  try {
    const newEmbeddings = configureEmbeddings('fallback_model');
    // Update embeddings configuration
    await updateEmbeddingsConfig(newEmbeddings);
  } catch (configError) {
    throw new SearchHandlingError(
      'Failed to reconfigure embeddings',
      'EMBEDDINGS_CONFIG_ERROR',
      configError
    );
  }
};

const handleTimeoutError = async (err: SearchError): Promise<void> => {
  console.error('Search timeout:', err.message);
  // Implement exponential backoff retry
  const maxRetries = 3;
  let retryCount = 0;
  let delay = 1000; // Start with 1 second delay

  while (retryCount < maxRetries) {
    try {
      await new Promise(resolve => setTimeout(resolve, delay));
      // Retry the search operation
      await retrySearch(err.details);
      return;
    } catch (retryError) {
      retryCount++;
      delay *= 2; // Double the delay for next retry
    }
  }

  throw new SearchHandlingError(
    'Max retries exceeded for timeout',
    'MAX_RETRIES_EXCEEDED',
    err.details
  );
};

const handleInvalidQueryError = async (err: SearchError): Promise<void> => {
  console.error('Invalid query:', err.message);
  // Attempt to clean and validate query
  try {
    const cleanedQuery = await cleanQuery(err.details.query);
    if (cleanedQuery) {
      await retrySearch({ ...err.details, query: cleanedQuery });
    }
  } catch (cleanError) {
    throw new SearchHandlingError(
      'Failed to clean invalid query',
      'QUERY_CLEANUP_ERROR',
      cleanError
    );
  }
};

const handleSourceError = async (err: SearchError): Promise<void> => {
  console.error('Source access error:', err.message);
  // Log source access failure and try alternative sources
  try {
    await logSourceFailure(err.details.source);
    const alternativeSources = await findAlternativeSources(err.details.query);
    if (alternativeSources.length > 0) {
      await retrySearch({ ...err.details, sources: alternativeSources });
    }
  } catch (sourceError) {
    throw new SearchHandlingError(
      'Failed to find alternative sources',
      'SOURCE_ALTERNATIVE_ERROR',
      sourceError
    );
  }
};

const handleUnknownError = async (err: SearchError): Promise<void> => {
  console.error('Unknown error:', err.message);
  // Log unknown error and try general recovery
  await logUnknownError(err);
  if (err.retry) {
    await attemptErrorRecovery(err);
  }
};

// Helper functions for error handling
const attemptErrorRecovery = async (err: SearchError): Promise<void> => {
  console.log('Attempting error recovery...');
  try {
    // Reset search state
    await resetSearchState();
    // Retry the failed operation
    await retrySearch(err.details);
  } catch (recoveryError) {
    throw new SearchHandlingError(
      'Recovery attempt failed',
      'RECOVERY_FAILED',
      recoveryError
    );
  }
};

const resetSearchState = async (): Promise<void> => {
  // Implementation for resetting search state
  // Clear caches, reset counters, etc.
};

const retrySearch = async (details: any): Promise<void> => {
  // Implementation for retrying a search operation
  // Use details to reconstruct the search
};

const cleanQuery = async (query: string): Promise<string> => {
  // Implementation for cleaning and validating a query
  return query.trim().replace(/[^\w\s-]/g, '');
};

const logSourceFailure = async (source: string): Promise<void> => {
  // Implementation for logging source failures
};

const findAlternativeSources = async (query: string): Promise<string[]> => {
  // Implementation for finding alternative sources
  return [];
};

const logUnknownError = async (err: SearchError): Promise<void> => {
  // Implementation for logging unknown errors
};

const updateEmbeddingsConfig = async (newEmbeddings: any): Promise<void> => {
  // Implementation for updating embeddings configuration
};
```

### 4. Quality Control Implementation

#### A. Citation Verification
```typescript
interface Citation {
  id: number;
  text: string;
  source: string;
  location: {
    start: number;
    end: number;
  };
}

interface CitationVerificationResult {
  valid: boolean;
  missingCitations: string[];
  invalidCitations: Citation[];
  suggestions: string[];
}

const verifyCitations = (content: string, sources: string[]): CitationVerificationResult => {
  // Extract citations from content using regex
  const citationRegex = /\[(\d+)\]/g;
  const citations: Citation[] = [];
  let match;
  
  while ((match = citationRegex.exec(content)) !== null) {
    const citationId = parseInt(match[1]);
    const citation: Citation = {
      id: citationId,
      text: match[0],
      source: sources[citationId - 1] || '',
      location: {
        start: match.index,
        end: match.index + match[0].length
      }
    };
    citations.push(citation);
  }

  // Verify

## Testing and Deployment

### 1. Testing Procedure
```bash
# Build test environment
docker-compose -f docker-compose.test.yml up -d

# Run tests
npm run test:copilot
```

### 2. Deployment Steps
```bash
# Build production image
docker build -t perplexica:latest .

# Deploy
docker-compose up -d
```

## Best Practices and Guidelines

### 1. Code Quality
- Maintain consistent error handling
- Document all major functions
- Use TypeScript strictly

### 2. Performance Optimization
- Implement caching for frequent queries
- Monitor response times
- Optimize search iterations

### 3. Maintenance
- Regular prompt template updates
- Monitor error rates
- Track user feedback

## Troubleshooting

### Common Issues
1. **Cosine Similarity Errors**
   - Verify input array lengths
   - Check embedding dimensions

2. **Reference Accuracy**
   - Validate source links
   - Verify citation format

3. **Performance Issues**
   - Monitor iteration counts
   - Check response times

## Future Enhancements

### Planned Features
1. Parallel search capabilities
2. Advanced source validation
3. Enhanced suggestion algorithms

### Development Roadmap
1. Q2 2024: Implement parallel search
2. Q3 2024: Enhance source validation
3. Q4 2024: Improve suggestion quality

## Monitoring and Metrics

### Key Performance Indicators
1. Response accuracy
2. Query processing time
3. Source reliability score

### Monitoring Implementation
```typescript
const monitorPerformance = async (metrics: {
  responseTime: number,
  accuracy: number,
  sourceReliability: number
}) => {
  // Implementation details...
};
```

## Documentation Updates

Keep documentation updated with:
1. New feature implementations
2. Bug fixes and workarounds
3. Performance optimization techniques
4. Best practices and guidelines

## Contribution Guidelines

1. **Code Submissions**
   - Follow TypeScript standards
   - Include tests
   - Document changes

2. **Prompt Updates**
   - Test thoroughly
   - Document improvements
   - Provide example outputs
