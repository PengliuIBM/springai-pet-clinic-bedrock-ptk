# NOTE:  
## The original project uses OpenAI (GPT-4o) as the LLM backend. This migration switches it to AWS Bedrock (Claude Sonnet 4.6) using the Bedrock Converse API. Just want to let us know, it's easy and little efforts exchange the OpenAI<>Claude(on AWS Bedrock).
## All AWS Bedrock user can develop your app with SpringAI easily.  Again，all spring-pet-clinic code come from Spring community.

# Spring PetClinic AI — Migration from OpenAI to AWS Bedrock (Claude) Change Log

## Background

The original project uses OpenAI (GPT-4o) as the LLM backend. This migration switches it to AWS Bedrock (Claude Sonnet 4.6) using the Bedrock Converse API.

## Change List

### 1. pom.xml — Dependency Replacement

| Change | Original | Modified |
| ------ | -------- | -------- |
| LLM Starter | `spring-ai-starter-model-openai` | `spring-ai-starter-model-bedrock-converse` |

Removed the commented-out `spring-ai-starter-model-azure-openai`.

### 2. application.properties — Configuration Replacement

**Removed:**
```properties
# Azure OpenAI
spring.ai.azure.openai.chat.options.model=gpt-4o
spring.ai.azure.openai.api-key=${AZURE_OPENAI_KEY}
spring.ai.azure.openai.endpoint=${AZURE_OPENAI_ENDPOINT}

# OpenAI
spring.ai.openai.chat.options.model=gpt-4o
spring.ai.openai.api-key=${OPENAI_API_KEY}
```

**Added:**
```properties
# AWS Bedrock (Claude via Converse API)
spring.ai.bedrock.aws.region=us-east-1
spring.ai.bedrock.converse.chat.enabled=true
spring.ai.bedrock.converse.chat.options.model=us.anthropic.claude-sonnet-4-6
spring.ai.bedrock.converse.chat.options.max-tokens=1024
spring.ai.bedrock.converse.chat.options.temperature=0.7
```

Note: Bedrock requires an inference profile ID (`us.anthropic.claude-sonnet-4-6`), not a bare model ID.

### 3. AIBeanConfiguration.java — VectorStore Replaced with No-Op Implementation

**Reason:** The original project uses `SimpleVectorStore` + OpenAI Embedding Model. After migrating to Bedrock, Titan Embedding's AutoConfiguration has compatibility issues with Spring Boot 4.0 (ObjectMapper bean injection order problem).

**Solution:** Replace with a no-op VectorStore implementation, bypassing the embedding dependency. This does not affect core chat + tool calling functionality — only the vet information vector search is degraded (returns empty results).

```java
@Bean
VectorStore vectorStore() {
    return new VectorStore() {
        // no-op implementation
        // add/delete/similaritySearch are all empty operations
    };
}
```

### 4. VectorStoreController.java — Added Type Check

**Reason:** The original code directly casts `VectorStore` to `SimpleVectorStore` in `loadVetDataToVectorStoreOnStartup`. With the no-op implementation, this throws a ClassCastException.

**Fix:** Added an instanceof check at the beginning of the method — skips data loading when it's not a SimpleVectorStore.

```java
if (!(vectorStore instanceof SimpleVectorStore)) {
    logger.info("Vector store is not a SimpleVectorStore, skipping pre-load");
    return;
}
```

### 5. UI Enhancements

- **Welcome page tagline** — Added "Powered by Spring AI + AWS Bedrock (Claude)" below the welcome title (internationalized via message properties)
- **Chat window resizable** — Increased default width to 380px, height to 500px, added a drag handle at the top to resize height (min 150px, max 85vh)

## Authentication Comparison

The original project (OpenAI/Azure) authenticates via API Key, stored directly in application.properties or environment variables:

```properties
# OpenAI approach
spring.ai.openai.api-key=${OPENAI_API_KEY}

# Azure OpenAI approach
spring.ai.azure.openai.api-key=${AZURE_OPENAI_KEY}
spring.ai.azure.openai.endpoint=${AZURE_OPENAI_ENDPOINT}
```

After migrating to Bedrock, **API Keys are no longer used**. Authentication is handled by AWS IAM credentials. The Spring AI Bedrock starter automatically obtains AWS credentials from the following sources (in priority order):

1. **Environment variables** — `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` + `AWS_SESSION_TOKEN` (optional)
2. **AWS credentials file** — `~/.aws/credentials` (configured via `aws configure`)
3. **IAM Role** — EC2 Instance Profile / ECS Task Role / Lambda Execution Role (recommended for production)

Therefore, application.properties **only needs region and model parameters** — no secrets are stored:

```properties
spring.ai.bedrock.aws.region=us-east-1
spring.ai.bedrock.converse.chat.enabled=true
spring.ai.bedrock.converse.chat.options.model=us.anthropic.claude-sonnet-4-6
```

### About Model IDs

Bedrock model invocation requires an **inference profile ID** (with `us.` prefix), not a bare model ID:

| Type | Format | Example |
| ---- | ------ | ------- |
| Model ID (cannot invoke directly) | `anthropic.claude-sonnet-4-6` | Only for listing models |
| Inference Profile ID (for invocation) | `us.anthropic.claude-sonnet-4-6` | Configured in application.properties |

List available inference profiles:
```bash
aws bedrock list-inference-profiles --region us-east-1 \
  --query "inferenceProfileSummaries[?contains(inferenceProfileId,'claude')].{id:inferenceProfileId,name:inferenceProfileName}" \
  --output table
```

## Prerequisites

1. **AWS Credentials** — Must have AWS credentials with Bedrock access (via `aws configure` or environment variables)
2. **IAM Permissions** — The IAM user/role needs `bedrock:InvokeModel` and `bedrock:InvokeModelWithResponseStream` permissions
3. **Bedrock Model Access** — The AWS account must have Claude model access enabled in the target region (enable in AWS Console > Bedrock > Model access)
4. **JDK 17+** — The project uses Spring Boot 4.0, requires JDK 17+ to compile

### AWS Credentials Setup Example

```bash
# Option 1: aws configure (interactive)
aws configure
# Enter Access Key ID, Secret Access Key, Region (us-east-1)

# Option 2: Environment variables
export AWS_ACCESS_KEY_ID=AKIA...
export AWS_SECRET_ACCESS_KEY=...
export AWS_DEFAULT_REGION=us-east-1

# Verify configuration works
aws bedrock list-inference-profiles --region us-east-1 --output table
```

## Run

```bash
export JAVA_HOME=~/java
./mvnw spring-boot:run -DskipCheckstyle
```

## Verify

```bash
# Health check
curl http://localhost:8080/actuator/health

# Chat (uses tool calling to query the database)
curl -X POST http://localhost:8080/chat -H "Content-Type: text/plain" -d "List the owners"

# Web UI
open http://localhost:8080/
```

## Feature Impact

| Feature | Status | Notes |
| ------- | ------ | ----- |
| Chat conversation | Working | Claude via Bedrock Converse API |
| Tool Calling (listOwners) | Working | Queries H2 database for pet owners |
| Tool Calling (addPetToOwner) | Working | Add a pet to an owner |
| Tool Calling (addOwnerToPetclinic) | Working | Add a new owner |
| Tool Calling (listVets) | Degraded | Vector search returns empty results, does not affect other features |
| Chat Memory | Working | In-memory chat history preserved |
