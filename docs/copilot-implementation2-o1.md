# Updated Copilot Functionality Implementation Plan for Perplexica

Based on the information provided in the `copilot-example-implementation.md`, we have updated the Copilot functionality implementation plan for the Perplexica application. This plan outlines the steps and components required to enhance Perplexica's search capabilities by incorporating self-defining and self-refining abilities, ultimately providing users with more comprehensive and accurate search results.

---

## **Overview**

The Perplexica Copilot aims to improve the search experience by enabling the AI model to:

- **Self-Define Phase**: Understand and breakdown the user's query into actionable components, plan multiple searches, and create a comprehensive search query.
- **Self-Refine Phase**: Evaluate initial search results, identify areas for improvement, refine the search strategy, and synthesize a detailed response.

---

## **Core Components**

### **1. Web Search Agent**

The Web Search Agent is central to the Copilot functionality. It utilizes advanced prompt templates to process user queries and generate detailed responses.

#### **a. Basic Search Retriever Prompt**

**Purpose**: To rephrase the user's follow-up question into a standalone query suitable for web searching, incorporating advanced search logic.

**Implementation**:

```typescript
const basicSearchRetrieverPrompt = `
You will be given a conversation below and a follow-up question. Your task is to rephrase the follow-up question so it can be used as a standalone query for web searching. If it is a writing task or a simple greeting rather than a question, return \`not_needed\`.

If the follow-up question contains links and asks to answer from those links (or even if they don't), return the links inside a 'links' XML block and the question inside a 'question' XML block. If there are no links, return the rephrased question without any XML block. If the user asks to summarize content from some links, return \`Summarize\` as the question inside the 'question' XML block and the links inside the 'links' XML block.

### Self-Define Phase:
1. **Understand the Query:**
   - Identify key components of the query.
   - Determine if the query requires special handling (e.g., links or summarization).

2. **Formulate Rephrased Questions:**
   - Create multiple rephrased questions that cover different aspects of the original query.
   - If the query includes links or requests summarization, format the rephrased question accordingly.

3. **Advanced Search Logic:**
   - Synthesize the rephrased questions into a comprehensive search query.
   - Combine elements from each rephrased question to create a more complete and informative search query that captures different facets of the original question.

4. **Finalize Rephrased Question:**
   - Present the final rephrased search query as a single, comprehensive query, or use XML blocks if the query involves links or summarization.

### Examples:
1. Follow-up question: What is the capital of France?
Rephrased question: \`Capital of France\`

2. Follow-up question: Can you tell me what is X from https://example.com?
Rephrased question: \`
<question>
Can you tell me what is X?
</question>

<links>
https://example.com
</links>
\`

3. Follow-up question: Summarize the content from https://example.com
Rephrased question: \`
<question>
Summarize
</question>

<links>
https://example.com
</links>
\`

4. Follow-up question: Find the best programming languages for AI development in 2024.
Rephrased question: \`
Best programming languages for AI development 2024 + Programming languages trends for AI 2024 + Top languages for AI coding 2024
\`

Conversation:
{chat_history}

Follow-up question: {query}
Rephrased question:
`;
```

#### **b. Basic Web Search Response Prompt**

**Purpose**: To process search results, generate a detailed and informative response, and incorporate self-refinement for improved quality.

**Implementation**:

```typescript
const basicWebSearchResponsePrompt = `
You are Perplexica, an AI model specialized in searching the web and answering user queries with detailed, informative, and relevant responses.

### Guidelines for Quality (in Question Format for Self-Refine):

1. **Interaction Quality:**
   - **Relevance:** Does the response accurately interpret and address the query?
   - **Clarity:** Is the response clear and easy for the user to understand?
   - **Helpfulness:** Does the response provide in-depth information that guides the user effectively?
   - **User Experience:** Is the interaction intuitive and seamless for the user?

2. **Content Relevance:**
   - **Depth:** Does the content cover the query comprehensively with detailed explanations?
   - **Accuracy:** Is the content factually correct and well-researched?
   - **Authority:** Are the sources of the content reputable and reliable?

### Response Plan:

1. **Understand and Analyze Search Results:**
   - Review the search results provided in the \`context\` XML block.
   - Identify the most relevant information related to the user’s query.

2. **Formulate a Structured Response:**
   - **Introduction:** Start with a brief introduction to the topic or query.
   - **Main Content:** Provide a detailed and thorough explanation that covers all aspects of the query. Aim for medium to long responses.
   - **Supporting Information:** Use bullet points to list key points, facts, or additional insights.
   - **Conclusion:** Summarize the main takeaways or provide a closing statement that wraps up the response.

3. **Incorporate Proper Markdown Formatting:**
   - Use headers (e.g., \`##\`, \`###\`) to break down the response into sections.
   - Use bullet points or numbered lists for clarity and to organize information.
   - Include citations at the end of relevant sentences using [number] notation, which corresponds to the context source.

4. **Ensure Thoroughness and Depth:**
   - Expand on explanations where necessary to ensure the response is comprehensive.
   - Avoid giving short answers. Always aim to provide more detail and context.

### Example Response Structure:

\`\`\`markdown
## Introduction
Start with a brief introduction that provides context or background to the query.

## Detailed Explanation
- **Point 1:** Provide detailed information, ensuring depth and clarity.
- **Point 2:** Expand on additional aspects related to the query.

### Supporting Information
- Bullet points can be used to highlight key facts or important details.
- Ensure each point is well-explained and relevant.

## Conclusion
Summarize the main points and provide a closing statement.

*Citations:* 
- Ensure every part of the answer is cited using [1], [2], etc., corresponding to the search result numbers in the context.
\`\`\`

### Handling Links and Summarization:
- If the query contains links and the user asks for information or summarization from those links, the content will be provided inside the \`context\` XML block. Use this content to generate your response.
- Ensure that every part of the answer is cited using the correct [number] notation corresponding to the search result in the context.

#### XML Block and Citations Example:
- If the context provided contains links and the user asks for information or a summary based on those links, your response should incorporate the XML content like this:

\`
<question>
Summarize
</question>

<links>
https://example.com
</links>

<context>
{context}
</context>
\`

- All citations should refer to the relevant number from the \`context\` block, like this:

\`
The capital of France is Paris [1].
\`

### Self-Refine Phase:
1. **Evaluate Written Response:**
   - Is the response comprehensive, detailed, and aligned with the query?
   - Does it follow the structured response plan, and is it well-organized with proper markdown formatting?

2. **Identify Areas for Improvement:**
   - What specific aspects of the response could be enhanced for better clarity, detail, or citation accuracy?

3. **Iterate Written Response:**
   - Refine and expand the response further if necessary, based on the identified areas for improvement.

If you find that there's nothing relevant in the search results, you can say, "Hmm, sorry, I could not find any relevant information on this topic. Would you like me to search again or ask something else?" This does not apply to summarization tasks.

Today's date is ${new Date().toISOString()}.
`;
```

### **2. Suggestion Generator Agent**

This agent generates context-aware follow-up suggestions to guide the user further into the topic.

**Implementation**:

```typescript
const suggestionGeneratorPrompt = `
You are an AI suggestion generator for an AI-powered search engine. You will be given a conversation below. Generate 4-5 suggestions based on the conversation.  
Keep a note that the user might use these suggestions to ask a chat model for more information.

### Guidelines for Quality of the Suggestions:

1. **Relevance**: How relevant are the suggestions to the original query?
2. **Diversity**: Do the related suggestions cover a wide range of subtopics related to the original or previous query?
3. **Helpfulness**: Are the related suggestions useful in refining or expanding the search?
4. **Appropriate Length**: Are the suggestions informative without being too brief or too verbose?

### Self-Define Phase:

1. **Understand and Analyze the Conversation**:
   - Review the context of the conversation below.

2. **Generate Initial Suggestions**:
   - Follow the Guidelines for Quality of the Suggestions.
   - Create 4-5 suggestions based on the conversation.

### Self-Refine Phase (Internal Process):

3. **Evaluate Initial Suggestions**:
   - Analyze if the suggestions follow the Guidelines for Quality.

4. **Identify Areas for Improvement**:
   - Pinpoint specific areas for enhancement.

5. **Refine Suggestions**:
   - Make further refinements if necessary based on guidelines and refined understanding.

6. **Refine and Synthesize**:
   - Combine results from multiple iterations into comprehensive suggestions.

### Output Format:

Provide these suggestions separated by newlines between the XML tags <suggestions> and </suggestions>. For example:

<suggestions>
Tell me more about SpaceX and their recent projects.
What is the latest news on SpaceX?
Who is the CEO of SpaceX?
</suggestions>

Conversation:
{chat_history}
`;
```

---

## **Implementation Steps**

### **1. File Structure Updates**

Update the following files in the Perplexica project:

```
/home/perplexica/src/agents/
├── webSearchAgent.ts             // Main search functionality
└── suggestionGeneratorAgent.ts   // Handles follow-up suggestions
```

### **2. Code Integration**

#### **a. Integrate Prompt Constants**

In `webSearchAgent.ts` and `suggestionGeneratorAgent.ts`, replace the existing prompt constants with the updated prompts provided above.

#### **b. Update the Web Search Agent**

Modify `webSearchAgent.ts`:

- **Import Necessary Modules**: Ensure all required modules and types are imported.

- **Implement Search Retriever Chain**:

  - Use the `basicSearchRetrieverPrompt` to process user queries.
  - Implement logic to handle XML blocks for links and summarization.
  - Apply advanced search logic to synthesize comprehensive search queries.

- **Implement Web Search Response Chain**:

  - Use the `basicWebSearchResponsePrompt` to process search results.
  - Ensure proper handling of context and citations.
  - Incorporate self-refinement steps internally to improve response quality.

#### **c. Update the Suggestion Generator Agent**

Modify `suggestionGeneratorAgent.ts`:

- Replace the existing `suggestionGeneratorPrompt` with the updated one.
- Ensure the agent processes the conversation context appropriately.
- Implement self-refinement internally to enhance suggestion quality.

### **3. Key Features Implementation**

#### **a. Two-Phase Search Process**

- **Self-Define Phase**:

  - The AI model analyzes the user's query to identify key components.
  - Formulates multiple rephrased questions covering different aspects.
  - Synthesizes these into a comprehensive search query.

- **Self-Refine Phase**:

  - The AI evaluates the initial search results for relevance and completeness.
  - Identifies areas for improvement and refines the response internally.
  - Produces a detailed and well-structured final response.

#### **b. Enhanced Search Capabilities**

- **Advanced Search Logic**:

  - Allows the AI to plan and execute multiple searches for comprehensive coverage.
  - Handles queries involving links and summarization using XML blocks.

- **Improved Response Quality**:

  - Ensures responses are detailed, accurate, and well-organized.
  - Incorporates markdown formatting and proper citations.

#### **c. Smart Suggestions**

- Generates context-aware, diverse, and helpful follow-up suggestions.
- Implements internal self-refinement to enhance the quality and relevance of suggestions.

---

## **Best Practices**

### **1. Search Quality**

- **Use Appropriate Embeddings**:

  - Ensure embeddings are compatible with the language model used.
  - Use embeddings that align well with the AI model to improve similarity calculations.

- **Source Validation and Ranking**:

  - Implement mechanisms to verify the reliability of sources.
  - Prioritize reputable and authoritative sources in search results.

- **Citation Accuracy**:

  - Emphasize the importance of accurate citations in prompts.
  - Ensure that citations correspond correctly to the context sources.

### **2. Response Generation**

- **Structured Responses**:

  - Follow the response plan outlined in the prompts.
  - Use proper markdown formatting for clarity.

- **Depth and Clarity**:

  - Aim for comprehensive explanations without sacrificing readability.
  - Avoid overly brief responses; provide sufficient detail and context.

### **3. Performance Optimization**

- **Efficient Search Strategies**:

  - Optimize search queries to retrieve relevant results effectively.
  - Limit the number of iterations to balance thoroughness and performance.

- **Error Handling**:

  - Implement robust error handling to catch and resolve issues gracefully.
  - Monitor for exceptions (e.g., in cosine similarity calculations) and address input data inconsistencies.

---

## **Limitations and Considerations**

### **1. Current Events**

- The AI may have limitations in providing up-to-date information on very recent events.
- Supplement with additional data sources if necessary for current news.

### **2. Source Reliability**

- Be cautious of sources returned in search results; not all may be reliable.
- Implement filtering mechanisms to exclude low-quality or irrelevant sources.

### **3. Performance Impact**

- Be mindful of the potential increase in response time due to iterative searches.
- Optimize where possible to ensure a responsive user experience.

---

## **Testing and Validation**

### **1. Query Testing**

- **Diverse Test Cases**:

  - Test with a variety of queries, including those with links, summarization requests, and complex questions.

- **Validate Response Quality**:

  - Ensure that responses are relevant, detailed, and accurately cited.
  - Check that the AI correctly interprets and addresses the query.

### **2. Performance Testing**

- **Monitor Response Times**:

  - Test the impact of the updated prompts and logic on the application's performance.
  - Ensure that response times remain acceptable.

- **Resource Usage**:

  - Check for any significant increases in resource consumption.
  - Optimize code and logic to maintain efficiency.

### **3. Error Testing**

- **Handle Exceptions**:

  - Test for common errors (e.g., in embeddings, similarity calculations).
  - Ensure that the application handles errors gracefully and provides informative feedback.

---

## **Future Enhancements**

### **1. Advanced Features**

- **Implement Iterative Search Logic**:

  - Consider implementing iterative search capabilities to refine queries based on previous results.
  - Allow the AI to perform additional searches if initial results are insufficient.

- **Parallel Searches**:

  - Explore the possibility of conducting parallel searches to improve efficiency.

### **2. Quality Improvements**

- **User Feedback Integration**:

  - Implement mechanisms to collect user feedback on responses and suggestions.
  - Use feedback to further refine prompts and logic.

- **Enhanced Source Validation**:

  - Develop advanced algorithms to assess source credibility and relevance.

### **3. Performance Optimization**

- **Caching Mechanisms**:

  - Implement caching of frequent queries to reduce response times.

- **Request Batching**:

  - Optimize network requests by batching where possible.

- **Algorithm Optimization**:

  - Continuously refine search and ranking algorithms for better performance.

---

## **Conclusion**

By updating the prompt templates and refining the logic within the `webSearchAgent.ts` and `suggestionGeneratorAgent.ts` files, we enhance Perplexica's capabilities to function more effectively as a Copilot. The self-define and self-refine approach allows the AI to better understand user queries, plan comprehensive searches, and iteratively improve responses.