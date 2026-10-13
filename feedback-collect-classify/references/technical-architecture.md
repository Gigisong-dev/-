# 技术架构文档

## 概述

本文档详细说明 feedback-collect-classify 技能的技术架构、核心组件和实现原理。

## 系统架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                     用户交互层                               │
│  (NIO Chat Agent / 用户请求)                                 │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌───────────────────────────────────────────────────────────────┐
│                   配置管理层                                 │
│  - 多维表格配置                                              │
│  - 群聊配置                                                  │
│  - 字段映射配置                                              │
│  - 图片处理配置                                              │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                   数据采集层                                 │
│  - 群聊搜索                                                  │
│  - 消息拉取                                                  │
│  - 图片提取                                                  │
│  - 上下文获取                                                │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                   数据处理层                                 │
│  - 反馈筛选                                                  │
│  - 上下文补全                                                │
│  - 标准化处理                                                │
│  - 分类判断                                                  │
│  - 置信度评估                                                │
│  - 重复识别                                                  │
│  - 知识库判断                                                │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                   图片处理层                                 │
│  - 图片下载                                                  │
│  - 图片上传                                                  │
│  - 格式转换                                                  │
│  - 文件关联                                                  │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                   数据写入层                                 │
│  - 字段映射                                                  │
│  - 数据验证                                                  │
│  - 批量写入                                                  │
│  - 错误处理                                                  │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                   输出生成层                                 │
│  - 结果统计                                                  │
│  - JSON格式化                                                │
│  - 约束校验                                                  │
└─────────────────────────────────────────────────────────────┘
```

## 核心组件

### 1. 配置管理器 (Configuration Manager)

**职责**:
- 管理用户配置
- 验证配置有效性
- 提供配置访问接口

**主要配置项**:
```typescript
interface SkillConfig {
  // 多维表格配置
  baseConfig: {
    baseUrl?: string;
    baseToken: string;
    tableToken: string;
    tableName?: string;
  };
  
  // 字段映射配置
  fieldMapping: {
    [standardField: string]: string;
  } | null;
  
  // 群聊配置
  groupConfig: {
    groupName?: string;
    groupUrl?: string;
    chatId?: string;
  };
  
  // 时间窗口配置
  timeWindow: {
    start: string;  // ISO 8601格式
    end: string;    // ISO 8601格式
    useDefaultRule: boolean;
  };
  
  // 图片处理配置
  imageConfig: {
    enabled: boolean;
    fieldType: 'attachment' | 'url' | 'mixed';
    fieldName: string;
    uploadFolder: string;
    tempDir: string;
  };
  
  // 知识库配置
  knowledgeBaseConfig: {
    enabled: boolean;
    docUrl?: string;
  };
}
```

### 2. 数据采集器 (Data Collector)

**职责**:
- 搜索群聊
- 拉取消息
- 提取图片信息
- 获取上下文

**主要方法**:
```typescript
class DataCollector {
  // 搜索群聊
  async searchGroup(groupName: string): Promise<string> {
    const result = await execute(
      `lark-cli im +chat-search --query "${groupName}" --as user --json`
    );
    return result.chat_id;
  }
  
  // 拉取消息
  async fetchMessages(
    chatId: string, 
    startTime: string, 
    endTime: string
  ): Promise<Message[]> {
    const result = await execute(
      `lark-cli im +chat-messages-list --chat-id "${chatId}" ` +
      `--as user --start "${startTime}" --end "${endTime}" ` +
      `--order asc --page-size 200 --json`
    );
    return this.parseMessages(result);
  }
  
  // 提取图片
  extractImages(message: Message): ImageInfo[] {
    if (message.msg_type !== 'image') return [];
    
    return [{
      imageKey: message.content.image_key,
      url: message.content.images[0]?.url,
      metadata: this.extractMetadata(message)
    }];
  }
  
  // 获取上下文
  async getContext(
    chatId: string, 
    messageId: string, 
    range: number = 5
  ): Promise<Message[]> {
    // 向前获取range条消息
    // 返回上下文消息数组
  }
}
```

### 3. 数据处理器 (Data Processor)

**职责**:
- 筛选有效反馈
- 补全上下文
- 标准化处理
- 分类判断
- 置信度评估
- 重复识别

**核心算法**:

#### 反馈筛选算法
```typescript
function isValidFeedback(message: Message): boolean {
  // 排除公告
  if (isAnnouncement(message)) return false;
  
  // 排除群管理动作
  if (isGroupManagement(message)) return false;
  
  // 排除纯回复
  if (isPureReply(message)) return false;
  
  // 排除附和
  if (isAgreement(message)) return false;
  
  // 排除纯情绪表达
  if (isPureEmotion(message)) return false;
  
  // 排除闲聊
  if (isSmallTalk(message)) return false;
  
  // 默认分类标准下，产品建议纳入收集（isProductSuggestion 不再直接排除）；
  // 仅当用户自定义分类标准不含产品建议时才排除
  if (config.classificationStandard && !config.classificationStandard.includes('产品建议')) {
    if (isProductSuggestion(message)) return false;
  }
  
  // 检查是否包含明确提问、报错或产品建议
  return hasQuestionOrError(message) || isProductSuggestion(message);
}
```

#### 上下文补全算法
```typescript
function completeContext(
  message: Message, 
  context: Message[]
): ContextSummary {
  // 检查消息是否依赖前文
  if (!dependsOnContext(message)) {
    return { summary: "", needsContext: false };
  }
  
  // 向前查找完整问题描述
  const fullQuestion = findFullQuestion(context);
  
  // 生成上下文摘要
  const summary = generateSummary(fullQuestion, message);
  
  return {
    summary,
    needsContext: true,
    relatedMessages: extractRelatedMessages(context)
  };
}
```

#### 标准化算法
```typescript
function standardizeTitle(
  originalText: string, 
  context?: string
): string {
  // 去除情绪化表达
  let cleaned = removeEmotion(originalText);
  
  // 提取核心问题
  const core = extractCoreIssue(cleaned);
  
  // 使用标准术语
  const standardized = useStandardTerminology(core);
  
  // 生成简洁标题
  return generateConciseTitle(standardized);
}
```

#### 分类算法
```typescript
function classifyFeedback(
  text: string, 
  context?: string
): ClassificationResult {
  const fullText = context ? `${context}\n${text}` : text;
  
  // 技术问题判断
  if (isTechnicalIssue(fullText)) {
    return {
      category: '技术问题',
      confidence: calculateTechnicalConfidence(fullText)
    };
  }
  
  // 产品建议判断（默认分类标准）
  if (isProductSuggestion(fullText)) {
    return {
      category: '产品建议',
      confidence: calculateSuggestionConfidence(fullText)
    };
  }
  
  // 使用咨询判断
  if (isUsageInquiry(fullText)) {
    return {
      category: '使用咨询',
      confidence: calculateUsageConfidence(fullText)
    };
  }
  
  // 待划定
  return {
    category: '待划定',
    confidence: calculatePendingConfidence(fullText)
  };
}
```

#### 置信度评估算法
```typescript
function calculateConfidence(
  text: string, 
  category: string,
  context?: string
): number {
  let score = 0;
  
  // 信息完整性 (40分)
  score += assessInformationCompleteness(text, context) * 0.4;
  
  // 表达清晰度 (30分)
  score += assessClarity(text) * 0.3;
  
  // 分类确定性 (30分)
  score += assessClassificationCertainty(text, category) * 0.3;
  
  return Math.round(score);
}
```

#### 重复识别算法
```typescript
async function identifyDuplicate(
  newFeedback: Feedback,
  existingRecords: Feedback[]
): Promise<boolean> {
  for (const record of existingRecords) {
    // 语义相似度计算
    const similarity = await calculateSemanticSimilarity(
      newFeedback.standard_title,
      record.standard_title
    );
    
    // 相似度阈值判断
    if (similarity > 0.85) {
      return true;
    }
  }
  
  return false;
}
```

### 4. 图片处理器 (Image Processor)

**职责**:
- 下载图片
- 上传图片
- 格式转换
- 文件关联

**主要方法**:
```typescript
class ImageProcessor {
  // 下载图片
  async downloadImage(
    imageKey: string, 
    outputPath: string
  ): Promise<DownloadResult> {
    return await execute(
      `lark-cli im +image-download ` +
      `--image-key "${imageKey}" ` +
      `--output "${outputPath}" ` +
      `--as user`
    );
  }
  
  // 上传图片
  async uploadImage(
    filePath: string, 
    folderToken: string
  ): Promise<UploadResult> {
    return await execute(
      `lark-cli drive +file-upload ` +
      `--file "${filePath}" ` +
      `--parent-folder-token "${folderToken}" ` +
      `--as user`
    );
  }
  
  // 批量处理
  async processBatch(
    images: ImageInfo[],
    config: ImageConfig
  ): Promise<ProcessResult[]> {
    const results = [];
    
    for (const image of images) {
      try {
        // 下载
        const downloaded = await this.downloadImage(
          image.imageKey,
          this.generateTempPath(image)
        );
        
        // 上传
        const uploaded = await this.uploadImage(
          downloaded.path,
          config.uploadFolder
        );
        
        results.push({
          success: true,
          imageKey: image.imageKey,
          fileToken: uploaded.file_token,
          url: uploaded.url
        });
      } catch (error) {
        results.push({
          success: false,
          imageKey: image.imageKey,
          error: error.message
        });
      }
    }
    
    return results;
  }
}
```

### 5. 数据写入器 (Data Writer)

**职责**:
- 字段映射
- 数据验证
- 批量写入
- 错误处理

**主要方法**:
```typescript
class DataWriter {
  // 字段映射
  mapFields(
    feedback: ProcessedFeedback, 
    mapping: FieldMapping
  ): MappedRecord {
    const mapped = {};
    
    for (const [standardField, targetField] of Object.entries(mapping)) {
      mapped[targetField] = feedback[standardField];
    }
    
    return mapped;
  }
  
  // 数据验证
  validateRecord(record: MappedRecord): ValidationResult {
    const errors = [];
    
    // 检查必需字段
    for (const field of this.requiredFields) {
      if (!record[field]) {
        errors.push(`缺少必需字段: ${field}`);
      }
    }
    
    // 检查枚举值
    if (record['问题分类'] && !this.validCategories.includes(record['问题分类'])) {
      errors.push(`无效的问题分类: ${record['问题分类']}`);
    }
    
    // 检查字段一致性
    this.validateFieldConsistency(record, errors);
    
    return {
      valid: errors.length === 0,
      errors
    };
  }
  
  // 批量写入
  async batchWrite(
    records: MappedRecord[], 
    config: WriteConfig
  ): Promise<WriteResult> {
    const payload = {
      records: records.map(record => ({
        fields: record
      }))
    };
    
    return await execute(
      `lark-cli base +record-batch-create ` +
      `--base-token "${config.baseToken}" ` +
      `--table-token "${config.tableToken}" ` +
      `--as user ` +
      `--json << 'EOF'
${JSON.stringify(payload)}
EOF`
    );
  }
}
```

### 6. 输出生成器 (Output Generator)

**职责**:
- 生成统计信息
- 格式化输出
- 约束校验

**主要方法**:
```typescript
class OutputGenerator {
  // 生成输出
  generateOutput(
    config: SkillConfig,
    windowSummary: WindowSummary,
    processedItems: ProcessedFeedback[]
  ): Output {
    const output = {
      config_summary: this.generateConfigSummary(config),
      window_summary: windowSummary,
      dimension_summary: this.generateDimensionSummary(processedItems),
      collected_items: this.formatCollectedItems(processedItems)
    };
    
    // 约束校验
    this.validateConstraints(output);
    
    return output;
  }
  
  // 约束校验
  validateConstraints(output: Output): void {
    // 检查顶层字段
    this.validateTopLevelFields(output);
    
    // 检查记录完整性
    this.validateRecordCompleteness(output);
    
    // 检查枚举值
    this.validateEnumValues(output);
    
    // 检查统计一致性
    this.validateStatisticsConsistency(output);
    
    // 检查字段约束
    this.validateFieldConstraints(output);
  }
}
```

## 数据流

### 完整数据流

```
用户请求
  ↓
配置验证
  ↓
群聊搜索 → chat_id
  ↓
消息拉取 → 原始消息列表
  ↓
图片提取 → 图片信息列表
  ↓
反馈筛选 → 候选反馈列表
  ↓
上下文补全 → 补全后的反馈
  ↓
标准化处理 → 标准化标题
  ↓
分类判断 → 问题分类 + 置信度
  ↓
重复识别 → 跟进状态
  ↓
知识库判断 → 是否命中知识库
  ↓
图片处理 → 图片token/URL
  ↓
字段映射 → 映射后的记录
  ↓
数据验证 → 验证后的记录
  ↓
批量写入 → 写入结果
  ↓
输出生成 → 最终JSON输出
```

## 错误处理

### 错误分类

1. **配置错误**
   - 缺少必需配置
   - 配置格式错误
   - 配置值无效

2. **权限错误**
   - 群聊访问权限不足
   - 表格写入权限不足
   - 云空间操作权限不足

3. **数据错误**
   - 消息格式异常
   - 字段类型不匹配
   - 数据验证失败

4. **网络错误**
   - 连接超时
   - 网络不可达
   - API调用失败

5. **处理错误**
   - 分类失败
   - 图片处理失败
   - 重复识别失败

### 错误处理策略

```typescript
async function executeWithErrorHandling(
  operation: () => Promise<any>
): Promise<Result> {
  try {
    return await operation();
  } catch (error) {
    switch (error.type) {
      case 'ConfigurationError':
        return handleConfigurationError(error);
      case 'PermissionError':
        return handlePermissionError(error);
      case 'DataError':
        return handleDataError(error);
      case 'NetworkError':
        return handleNetworkError(error);
      case 'ProcessingError':
        return handleProcessingError(error);
      default:
        return handleUnknownError(error);
    }
  }
}
```

## 性能优化

### 1. 并发处理

**消息拉取并发**:
```typescript
// 分页并发拉取
const pages = Math.ceil(totalMessages / pageSize);
const results = await Promise.all(
  Array.from({ length: pages }, (_, i) => 
    fetchMessagesPage(chatId, i * pageSize, pageSize)
  )
);
```

**图片处理并发**:
```typescript
// 并发下载和上传
const downloadPool = new ThreadPool(3);
const uploadPool = new ThreadPool(3);

const downloaded = await downloadPool.processAll(images);
const uploaded = await uploadPool.processAll(downloaded);
```

### 2. 缓存策略

**群聊信息缓存**:
```typescript
const groupCache = new Map<string, string>();

async function getGroupId(groupName: string): Promise<string> {
  if (groupCache.has(groupName)) {
    return groupCache.get(groupName);
  }
  
  const chatId = await searchGroup(groupName);
  groupCache.set(groupName, chatId);
  return chatId;
}
```

**知识库缓存**:
```typescript
const knowledgeCache = new LRUCache<string, boolean>(100);

async function checkKnowledgeBase(query: string): Promise<boolean> {
  if (knowledgeCache.has(query)) {
    return knowledgeCache.get(query);
  }
  
  const result = await searchKnowledgeBase(query);
  knowledgeCache.set(query, result);
  return result;
}
```

### 3. 批量操作

**批量写入**:
```typescript
// 分批写入，避免单次请求过大
const batchSize = 100;
for (let i = 0; i < records.length; i += batchSize) {
  const batch = records.slice(i, i + batchSize);
  await batchWrite(batch);
}
```

## 监控和日志

### 日志级别

- **DEBUG**: 详细的调试信息
- **INFO**: 一般信息输出
- **WARN**: 警告信息
- **ERROR**: 错误信息

### 监控指标

- **处理时间**: 各阶段处理耗时
- **成功率**: 各操作成功率
- **错误分布**: 错误类型分布
- **资源使用**: CPU、内存、网络使用

### 日志格式

```json
{
  "timestamp": "2026-08-19T14:30:30Z",
  "level": "INFO",
  "component": "DataProcessor",
  "message": "处理完成",
  "metrics": {
    "total_messages": 150,
    "collected_count": 12,
    "processing_time_ms": 2340
  }
}
```

## 扩展性设计

### 1. 插件化架构

支持自定义分类器、重复识别算法等：

```typescript
interface Plugin {
  name: string;
  version: string;
  process(input: any): Promise<any>;
}

class PluginManager {
  private plugins: Map<string, Plugin> = new Map();
  
  register(plugin: Plugin): void {
    this.plugins.set(plugin.name, plugin);
  }
  
  async execute(name: string, input: any): Promise<any> {
    const plugin = this.plugins.get(name);
    if (!plugin) {
      throw new Error(`Plugin not found: ${name}`);
    }
    return await plugin.process(input);
  }
}
```

### 2. 配置驱动的处理流程

通过配置文件定义处理流程：

```yaml
pipeline:
  - name: "collect"
    type: "data-collector"
    config:
      max_messages: 1000
      
  - name: "filter"
    type: "feedback-filter"
    config:
      exclude_types: ["announcement", "small_talk"]
      
  - name: "classify"
    type: "classifier"
    config:
      model: "custom-classifier"
      confidence_threshold: 70
```

---

**最后更新**: 2026-08-19