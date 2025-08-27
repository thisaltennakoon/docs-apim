# Semantic Caching

The Semantic Cache is a custom policy for the WSO2 API Manager Universal Gateway that provides intelligent, meaning-based caching for AI requests. Unlike traditional caches that rely on exact matches, the Semantic Cache uses vector embeddings to identify and serve responses for semantically similar queries. This reduces latency, lowers costs associated with calling backend AI models, and improves overall API performance.

## Features

- **Intelligent Similarity Detection**: Uses embedding models to understand semantic similarity between queries.
- **Cost Optimization**: Reduces AI provider API calls by serving cached responses for similar queries.
- **Performance Enhancement**: Faster response times for semantically similar requests.
- **Multiple Embedding Providers**: Support for Mistral, Azure OpenAI, and OpenAI embedding models.
- **Vector Database Integration**: Leverages Zilliz for efficient similarity search.
- **Configurable Similarity Thresholds**: Fine-tune cache behavior based on your use case.

## Configure the Environment

Before using the Semantic Cache policy, you must configure an embedding provider and a vector database in the `deployment.toml` file.

### 1. Embedding Provider Configuration

Choose one of the following embedding providers and add the configuration to your `<APIM_HOME>/repository/conf/deployment.toml` file:

=== "Mistral"

    ```toml
    [apim.ai.embedding_provider]
    type = "mistral"
    [apim.ai.embedding_provider.properties]
    apikey = "<your-mistral-api-key>"
    embedding_endpoint = "https://api.mistral.ai/v1/embeddings"
    embedding_model = "mistral-embed"
    ```

=== "Azure OpenAI"

    ```toml
    [apim.ai.embedding_provider]
    type = "azure"
    [apim.ai.embedding_provider.properties]
    apikey = "<your-azure-openai-api-key>"
    embedding_endpoint = "<your-azure-openai-embedding-endpoint>"
    ```

=== "OpenAI"

    ```toml
    [apim.ai.embedding_provider]
    type = "openai"
    [apim.ai.embedding_provider.properties]
    apikey = "<your-openai-api-key>"
    embedding_endpoint = "https://api.openai.com/v1/embeddings"
    embedding_model = "<openai-embedding-model>"
    ```

### 2. Vector Database Configuration
Currently, only Zilliz is supported as the vector database provider. Configure Zilliz in your `deployment.toml` file as shown below:

```toml
[apim.ai.vector_db_provider]
type = "zilliz"
[apim.ai.vector_db_provider.properties]
uri = "your-zilliz-instance-uri"
token = "your-zilliz-token"
```

You can optionally specify a `ttl` property to control the time-to-live for cached entries. If not set, the default TTL is 3600 seconds.

## How to Use

Follow these steps to integrate the Semantic Cache policy into your WSO2 API Manager AI APIs:

1. Log in to the API Publisher Portal at `https://<host>:<port>/publisher`.
2. Select the API you want to configure.
3. Navigate to **Develop > API Configurations > Policies > Request/Response Flow**.
4. Click **Add Policy** and choose **Semantic Cache** from the list.
5. Fill in the required settings, such as the cache policy name and similarity threshold.
6. Save your changes and deploy the API.

## Example Policy Configuration

??? example "Click to expand configuration steps"
    1. Create an AI API using OpenAI or any supported AI provider.  
    2. Add the Semantic Cache policy to the API with the following configuration:

    | Field                         | Example Value              | Description |
    |-------------------------------|----------------------------|--------------------------------------------------------|
    | `Semantic Cache Policy Name`   | `SemanticCache`            | Unique name for the semantic cache policy |
    | `Threshold`                   | `0.35`                     | Dissimilarity threshold for semantic matching (decimal value)|
    | `JSONPath`                    | `$.messages[-1].content`   | JSONPath expression to extract content for cache|

    **Threshold**:  
    The threshold sets the maximum allowed dissimilarity for semantic cache matches, measured using L2 (Euclidean) distance between embeddings.

    - For **normalized embeddings** (unit length vectors), the typical threshold range is **0.0** (exact match) to **2.0** (maximum distance).
    - For **unnormalized embeddings**, the range depends on your embedding model and data.

    Lower values (closer to 0) enforce stricter semantic similarity; higher values allow weaker matches. Refer to your embedding provider's documentation for recommended threshold values and normalization details.


    3. Save and re-deploy the API.  
    4. Invoke the API with similar queries to test semantic cache:

    **First Request:**
    ```json
    {
      "messages": [
        {
          "role": "user",
          "content": "What is the capital of France?"
        }
      ]
    }
    ```

    **Semantically Similar Request (Cache Hit):**
    ```json
    {
      "messages": [
        {
          "role": "user",
          "content": "Tell me the capital city of France"
        }
      ]
    }
    ```

    The second request will return a cached response much faster and with reduced costs, as it's semantically similar to the first query.

## Troubleshooting

### Common Issues

| Issue | Possible Cause | Solution |
|-------|----------------|----------|
| Low cache hit rate | Threshold (dissimilarity) too low | Increase the dissimilarity/distance threshold. |
| Irrelevant cached responses | Threshold too high | Decrease the dissimilarity/distance threshold. |
| Vector database connection errors | Invalid credentials or endpoint | Verify your Zilliz URI and token in the `deployment.toml` file. |
