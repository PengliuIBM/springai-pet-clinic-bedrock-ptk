# Spring PetClinic AI — 从 OpenAI 迁移到 AWS Bedrock (Claude) 变更记录

## 背景

原项目使用 OpenAI (GPT-4o) 作为 LLM 后端。本次改造将其迁移为 AWS Bedrock (Claude Sonnet 4.6)，使用 Bedrock Converse API。

## 变更清单

### 1. pom.xml — 依赖替换

| 变更 | 原始 | 修改后 |
| ---- | ---- | ------ |
| LLM Starter | `spring-ai-starter-model-openai` | `spring-ai-starter-model-bedrock-converse` |

移除了注释掉的 `spring-ai-starter-model-azure-openai`。

### 2. application.properties — 配置替换

**移除：**
```properties
# Azure OpenAI
spring.ai.azure.openai.chat.options.model=gpt-4o
spring.ai.azure.openai.api-key=${AZURE_OPENAI_KEY}
spring.ai.azure.openai.endpoint=${AZURE_OPENAI_ENDPOINT}

# OpenAI
spring.ai.openai.chat.options.model=gpt-4o
spring.ai.openai.api-key=${OPENAI_API_KEY}
```

**新增：**
```properties
# AWS Bedrock (Claude via Converse API)
spring.ai.bedrock.aws.region=us-east-1
spring.ai.bedrock.converse.chat.enabled=true
spring.ai.bedrock.converse.chat.options.model=us.anthropic.claude-sonnet-4-6
spring.ai.bedrock.converse.chat.options.max-tokens=1024
spring.ai.bedrock.converse.chat.options.temperature=0.7
```

注意：Bedrock 需要使用 inference profile ID（`us.anthropic.claude-sonnet-4-6`），而非裸 model ID。

### 3. AIBeanConfiguration.java — VectorStore 改为 No-Op 实现

**原因：** 原项目使用 `SimpleVectorStore` + OpenAI Embedding Model。迁移到 Bedrock 后，Titan Embedding 的 AutoConfiguration 与 Spring Boot 4.0 存在兼容性问题（缺少 ObjectMapper bean 注入顺序问题）。

**解决方案：** 用 no-op VectorStore 实现替代，跳过 embedding 依赖。对核心聊天 + tool calling 功能无影响，仅影响兽医信息的向量搜索（该功能退化为返回空结果）。

```java
@Bean
VectorStore vectorStore() {
    return new VectorStore() {
        // no-op implementation
        // add/delete/similaritySearch 均为空操作
    };
}
```

### 4. VectorStoreController.java — 添加类型检查

**原因：** 原代码在 `loadVetDataToVectorStoreOnStartup` 中直接将 `VectorStore` 强转为 `SimpleVectorStore`。使用 no-op 实现后会抛出 ClassCastException。

**修改：** 在方法开头添加 instanceof 检查，非 SimpleVectorStore 时直接跳过数据加载。

```java
if (!(vectorStore instanceof SimpleVectorStore)) {
    logger.info("Vector store is not a SimpleVectorStore, skipping pre-load");
    return;
}
```

## 认证方式对比

原项目（OpenAI/Azure）通过 API Key 认证，key 直接配在 application.properties 或环境变量中：

```properties
# OpenAI 方式
spring.ai.openai.api-key=${OPENAI_API_KEY}

# Azure OpenAI 方式
spring.ai.azure.openai.api-key=${AZURE_OPENAI_KEY}
spring.ai.azure.openai.endpoint=${AZURE_OPENAI_ENDPOINT}
```

迁移到 Bedrock 后，**不再使用 API Key**，改为 AWS IAM 认证。Spring AI Bedrock starter 会自动从以下位置（按优先级）获取 AWS credentials：

1. **环境变量** — `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` + `AWS_SESSION_TOKEN`（可选）
2. **AWS credentials 文件** — `~/.aws/credentials`（通过 `aws configure` 配置）
3. **IAM Role** — EC2 Instance Profile / ECS Task Role / Lambda Execution Role（生产环境推荐）

因此 application.properties 中**只需配置 region 和模型参数**，无需存放任何密钥：

```properties
spring.ai.bedrock.aws.region=us-east-1
spring.ai.bedrock.converse.chat.enabled=true
spring.ai.bedrock.converse.chat.options.model=us.anthropic.claude-sonnet-4-6
```

### 关于 model ID

Bedrock 调用模型需要使用 **inference profile ID**（带 `us.` 前缀），而非裸 model ID：

| 类型 | 格式 | 示例 |
| ---- | ---- | ---- |
| Model ID（不可直接调用） | `anthropic.claude-sonnet-4-6` | 仅用于列出模型 |
| Inference Profile ID（用于调用） | `us.anthropic.claude-sonnet-4-6` | 配置在 application.properties |

查看可用的 inference profile：
```bash
aws bedrock list-inference-profiles --region us-east-1 \
  --query "inferenceProfileSummaries[?contains(inferenceProfileId,'claude')].{id:inferenceProfileId,name:inferenceProfileName}" \
  --output table
```

## 前置条件

1. **AWS Credentials** — 本地需要有 Bedrock 访问权限的 AWS credentials（通过 `aws configure` 配置或环境变量设置）
2. **IAM 权限** — credentials 对应的 IAM 用户/角色需要 `bedrock:InvokeModel` 和 `bedrock:InvokeModelWithResponseStream` 权限
3. **Bedrock Model Access** — AWS 账户需在对应 region 启用 Claude 模型访问（在 AWS Console > Bedrock > Model access 中开启）
4. **JDK 17+** — 项目使用 Spring Boot 4.0，编译需要 JDK 17+

### AWS credentials 配置示例

```bash
# 方式一：aws configure（交互式）
aws configure
# 输入 Access Key ID, Secret Access Key, Region (us-east-1)

# 方式二：环境变量
export AWS_ACCESS_KEY_ID=AKIA...
export AWS_SECRET_ACCESS_KEY=...
export AWS_DEFAULT_REGION=us-east-1

# 验证配置是否有效
aws bedrock list-inference-profiles --region us-east-1 --output table
```

## 运行

```bash
export JAVA_HOME=~/java
./mvnw spring-boot:run -DskipCheckstyle
```

## 验证

```bash
# 健康检查
curl http://localhost:8080/actuator/health

# 聊天（使用 tool calling 查询数据库）
curl -X POST http://localhost:8080/chat -H "Content-Type: text/plain" -d "List the owners"

# Web UI
open http://localhost:8080/
```

## 功能影响

| 功能 | 状态 | 说明 |
| ---- | ---- | ---- |
| 聊天对话 | 正常 | Claude via Bedrock Converse API |
| Tool Calling (listOwners) | 正常 | 查询 H2 数据库返回宠物主人 |
| Tool Calling (addPetToOwner) | 正常 | 添加宠物到主人 |
| Tool Calling (addOwnerToPetclinic) | 正常 | 添加新主人 |
| Tool Calling (listVets) | 退化 | 向量搜索返回空结果，但不影响其他功能 |
| Chat Memory | 正常 | 内存聊天历史保持 |
