# FastGPT 知识库管理功能深度分析

## 概述
FastGPT 的知识库管理系统是整个平台的核心功能之一,负责文档的存储、向量化、检索和管理。系统支持多种数据源类型(本地文件、网页、API、飞书、语雀等),提供灵活的分块策略和强大的检索能力。

## 核心架构

### 数据模型

#### 1. 知识库 (Dataset)
**表名**: `datasets`

**核心字段**:
```typescript
{
  _id: ObjectId,
  parentId: ObjectId,           // 父文件夹ID(支持目录层级)
  teamId: ObjectId,             // 所属团队
  tmbId: ObjectId,              // 创建者
  type: DatasetTypeEnum,        // 类型:普通/文件夹/网页/API/飞书/语雀
  name: string,                 // 名称
  avatar: string,               // 头像
  intro: string,                // 简介
  vectorModel: string,          // 向量模型(如 text-embedding-ada-002)
  agentModel: string,           // Agent模型(如 gpt-4)
  vlmModel: string,             // 视觉语言模型
  status: DatasetStatusEnum,    // 状态:active/syncing/waiting/error
  apiDatasetServer: {...},      // API数据源配置
  inheritPermission: boolean,   // 是否继承权限
  createTime: Date,
  updateTime: Date
}
```

**知识库类型**:
- `folder`: 文件夹(组织层级)
- `dataset`: 普通知识库
- `websiteDataset`: 网站爬取
- `externalFile`: 外部文件
- `apiDataset`: API 数据源
- `feishu`: 飞书文档
- `yuque`: 语雀文档

#### 2. 文档集合 (DatasetCollection)
**表名**: `dataset_collections`

**核心字段**:
```typescript
{
  _id: ObjectId,
  datasetId: ObjectId,          // 所属知识库
  parentId: string,             // 父集合ID(支持文件夹)
  teamId: ObjectId,
  tmbId: ObjectId,
  name: string,
  type: DatasetCollectionTypeEnum, // 类型
  
  // 文件相关
  fileId: string,               // S3文件ID
  rawLink: string,              // 原始链接
  apiFileId: string,            // API文件ID
  externalFileId: string,       // 外部文件ID
  externalFileUrl: string,      // 外部文件URL
  
  // 训练配置
  trainingType: DatasetCollectionDataProcessModeEnum,
  chunkSize: number,            // 分块大小
  chunkSplitter: string,        // 分块分隔符
  chunkSettingMode: ChunkSettingModeEnum,
  chunkSplitMode: DataChunkSplitModeEnum,
  paragraphChunkAIMode: ParagraphChunkAIModeEnum,
  paragraphChunkDeep: number,
  paragraphChunkMinSize: number,
  
  // 索引配置
  indexSize: number,            // 索引块大小
  autoIndexes: boolean,         // 自动索引
  imageIndex: boolean,          // 图片索引
  indexPrefixTitle: boolean,    // 索引前缀标题
  
  // 增强配置
  dataEnhanceCollectionName: boolean,
  chunkTriggerType: ChunkTriggerConfigTypeEnum,
  chunkTriggerMinSize: number,
  
  // 状态信息
  trainingAmount: number,       // 训练数据量
  forbid: boolean,              // 是否禁用
  hashRawText: string,          // 原始文本哈希(用于去重)
  
  metadata: {...},              // 元数据
  tags: [...],                  // 标签
  
  createTime: Date,
  updateTime: Date
}
```

**集合类型**:
- `folder`: 文件夹
- `file`: 文件
- `link`: 网页链接
- `apiCollection`: API 集合

**训练类型**:
- `chunk`: 直接分块(最常用)
- `qa`: 问答对生成
- `backup`: 备份(不训练)
- `template`: 模板
- `auto`: 自动(已废弃,转为chunk)

#### 3. 数据块 (DatasetData)
**表名**: `dataset_datas`

**核心字段**:
```typescript
{
  _id: ObjectId,
  teamId: ObjectId,
  tmbId: ObjectId,
  datasetId: ObjectId,
  collectionId: ObjectId,
  
  q: string,                    // 问题/索引文本(用于检索)
  a: string,                    // 答案/实际内容
  fullTextToken: string,        // 全文分词结果(用于全文检索)
  
  imageId: string,              // 图片ID
  imageDescMap: {...},          // 图片描述映射
  
  chunkIndex: number,           // 块索引(在集合内的顺序)
  
  indexes: [{
    dataId: string,             // 向量数据库中的ID
    defaultIndex: boolean,      // 是否为默认索引
    type: IndexTypeEnum         // 索引类型:embedding/fullText
  }],
  
  updateTime: Date
}
```

**索引说明**:
- `indexes` 数组存储向量索引引用
- 每个数据块可以有多个索引(embedding + fullText)
- `dataId` 指向向量数据库中的实际向量数据

#### 4. 训练队列 (DatasetTraining)
**表名**: `dataset_trainings`

**核心字段**:
```typescript
{
  _id: ObjectId,
  teamId: ObjectId,
  tmbId: ObjectId,
  datasetId: ObjectId,
  collectionId: ObjectId,
  billId: string,               // 账单ID
  
  mode: TrainingModeEnum,       // 训练模式
  prompt: string,               // 提示词(QA模式)
  
  q: string,                    // 问题文本
  a: string,                    // 答案文本
  imageId: string,              // 图片ID
  
  chunkIndex: number,           // 块索引
  indexSize: number,            // 索引大小
  weight: number,               // 权重
  
  indexes: [{...}],             // 索引配置
  
  lockTime: Date,               // 锁定时间(防止重复处理)
  retryCount: number,           // 重试次数
  
  model: string                 // 使用的模型
}
```

**训练模式**:
- `chunk`: 直接分块向量化
- `qa`: 生成问答对
- `auto`: 自动处理
- `image`: 图片索引
- `imageParse`: 图片解析

#### 5. 数据块文本 (DatasetDataText)
**表名**: `dataset_data_texts`

**说明**: 存储数据块的纯文本内容,用于全文检索

**核心字段**:
```typescript
{
  _id: ObjectId,
  datasetId: ObjectId,
  collectionId: ObjectId,
  dataId: ObjectId,             // 关联的 DatasetData ID
  text: string,                 // 纯文本内容
  updateTime: Date
}
```

#### 6. 集合标签 (DatasetCollectionTags)
**表名**: `dataset_collection_tags`

**核心字段**:
```typescript
{
  _id: ObjectId,
  teamId: ObjectId,
  datasetId: ObjectId,
  tag: string,                  // 标签名称
  createTime: Date
}
```

#### 7. 数据同步 (DatasetSync)
**说明**: 用于外部数据源的定时同步任务

**队列**: `datasetSyncQueue` (BullMQ)

**任务数据**:
```typescript
{
  datasetId: string,
  type: DatasetTypeEnum,
  repeatConfig: {
    pattern: string,            // Cron 表达式
    enabled: boolean
  }
}
```

### 核心功能模块

#### 1. 知识库创建

**API**: `POST /api/core/dataset/create`

**流程**:
1. 权限验证(团队创建权限/父文件夹写权限)
2. 模型配置验证(vectorModel, agentModel)
3. 团队配额检查
4. 创建知识库记录
5. 创建权限记录
6. 记录审计日志

**关键代码**:
```typescript
// packages/service/core/dataset/controller.ts
const [dataset] = await MongoDataset.create([{
  ...parseParentIdInMongo(parentId),
  name,
  intro,
  teamId,
  tmbId,
  vectorModel,
  agentModel,
  vlmModel,
  avatar,
  type,
  apiDatasetServer
}], { session });

await MongoResourcePermission.insertOne({
  teamId,
  tmbId,
  resourceId: dataset._id,
  permission: OwnerRoleVal
}, { session });
```

#### 2. 文档上传与处理

**API**: `POST /api/core/dataset/collection/create/localFile`

**流程**:
1. 接收文件上传(Multer)
2. 权限验证(知识库写权限)
3. 上传文件到 S3
4. 创建 Collection 记录
5. 解析文件内容
6. 文本分块
7. 推送到训练队列
8. 异步向量化

**关键代码**:
```typescript
// projects/app/src/pages/api/core/dataset/collection/create/localFile.ts
const fileId = await getS3DatasetSource().upload({
  datasetId: dataset._id,
  stream: result.getReadStream(),
  size: result.fileMetadata.size,
  filename: result.fileMetadata.originalname
});

const { collectionId, insertResults } = await createCollectionAndInsertData({
  dataset,
  createCollectionParams: {
    datasetId: dataset._id,
    name: collectionName,
    teamId,
    tmbId,
    type: DatasetCollectionTypeEnum.file,
    fileId,
    metadata: { relatedImgId: fileId }
  }
});
```

#### 3. 文本分块策略

**分块模式** (`ChunkSplitMode`):
- `auto`: 自动(使用默认分隔符)
- `paragraph`: 段落分块
- `custom`: 自定义分隔符

**段落分块配置**:
```typescript
{
  paragraphChunkAIMode: 'simple' | 'deep',  // 简单/深度分析
  paragraphChunkDeep: 2,                    // 深度层级
  paragraphChunkMinSize: 100,               // 最小块大小
  chunkSize: 512,                           // 目标块大小
  chunkSplitter: '\n'                       // 分隔符
}
```

**分块算法**:
```typescript
// packages/service/core/dataset/read.ts
export const rawText2Chunks = async ({
  text,
  chunkSize,
  overlapRatio = 0.2,
  minChunkSize = 100
}: {
  text: string;
  chunkSize: number;
  overlapRatio?: number;
  minChunkSize?: number;
}): Promise<string[]> => {
  // 1. 按段落分割
  const paragraphs = text.split(/\n\n+/);
  
  // 2. 合并小段落
  const chunks: string[] = [];
  let currentChunk = '';
  
  for (const para of paragraphs) {
    if (currentChunk.length + para.length < chunkSize) {
      currentChunk += (currentChunk ? '\n\n' : '') + para;
    } else {
      if (currentChunk) chunks.push(currentChunk);
      currentChunk = para;
    }
  }
  if (currentChunk) chunks.push(currentChunk);
  
  // 3. 处理重叠(提升检索召回率)
  const overlapSize = Math.floor(chunkSize * overlapRatio);
  const resultChunks = [];
  
  for (let i = 0; i < chunks.length; i++) {
    let chunk = chunks[i];
    
    // 添加前文重叠
    if (i > 0 && overlapSize > 0) {
      const prevOverlap = chunks[i-1].slice(-overlapSize);
      chunk = prevOverlap + chunk;
    }
    
    resultChunks.push(chunk);
  }
  
  return resultChunks.filter(c => c.length >= minChunkSize);
};
```

#### 4. 训练队列处理

**队列定义**:
```typescript
// packages/service/core/dataset/training/controller.ts
const trainingQueue = new Queue('datasetTraining', {
  connection: redis,
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 5000
    },
    removeOnComplete: 100,
    removeOnFail: 100
  }
});
```

**批量推送**:
```typescript
export async function pushDataListToTrainingQueue({
  teamId,
  tmbId,
  datasetId,
  collectionId,
  data,
  mode = TrainingModeEnum.chunk,
  session
}: PushDataToTrainingQueueProps) {
  // 1. 验证模型配置
  const vectorModelData = getEmbeddingModel(vectorModel);
  const agentModelData = getLLMModel(agentModel);
  
  // 2. 过滤数据(去除空内容、超长内容)
  data = data.filter(item => {
    const text = (item.q || '') + (item.a || '');
    return text.length > 0 && text.length <= maxToken;
  });
  
  // 3. 批量插入(500条/批次,避免事务超时)
  const batchSize = 500;
  const maxBatchesPerTransaction = 20;
  
  for (let i = 0; i < data.length; i += batchSize) {
    const batch = data.slice(i, i + batchSize);
    
    await MongoDatasetTraining.insertMany(
      batch.map(item => ({
        teamId,
        tmbId,
        datasetId,
        collectionId,
        mode,
        q: item.q,
        a: item.a,
        imageId: item.imageId,
        chunkIndex: item.chunkIndex ?? 0,
        weight: vectorModelData.weight,
        indexes: item.indexes,
        retryCount: 5
      })),
      { session }
    );
  }
  
  return { insertLen: data.length };
}
```

**Worker 处理**:
```typescript
// packages/service/worker/dataset/index.ts
const trainingWorker = new Worker('datasetTraining', async (job) => {
  const { teamId, datasetId, collectionId, trainingId } = job.data;
  
  // 1. 获取训练任务
  const training = await MongoDatasetTraining.findById(trainingId);
  
  // 2. 生成向量
  const vector = await getVectorsByText({
    model: training.model,
    input: training.q
  });
  
  // 3. 存储向量到向量数据库
  const dataId = await insertDataToVectorDB({
    datasetId,
    collectionId,
    vector,
    text: training.q
  });
  
  // 4. 创建 DatasetData 记录
  await MongoDatasetData.create({
    teamId,
    datasetId,
    collectionId,
    q: training.q,
    a: training.a,
    chunkIndex: training.chunkIndex,
    indexes: [{
      dataId,
      type: DatasetDataIndexTypeEnum.embedding,
      defaultIndex: true
    }]
  });
  
  // 5. 删除训练任务
  await MongoDatasetTraining.deleteOne({ _id: trainingId });
  
  // 6. 计费
  await createTrainingUsage({
    teamId,
    tokens: vector.tokens,
    model: training.model
  });
}, { connection: redis });
```

#### 5. 知识库检索

**检索模式** (`DatasetSearchMode`):
- `embedding`: 向量检索(语义相似度)
- `fullTextRecall`: 全文检索(关键词匹配)
- `mixedRecall`: 混合检索(向量+全文)

**核心检索流程**:
```typescript
// packages/service/core/dataset/search/controller.ts
export async function searchDatasetData({
  teamId,
  datasetIds,
  queries,
  reRankQuery,
  searchMode = 'embedding',
  similarity = 0.5,
  limit: maxTokens = 3000,
  usingReRank = false,
  rerankModel,
  collectionFilterMatch
}: SearchDatasetDataProps): Promise<SearchDatasetDataResponse> {
  
  // 1. 初始化参数
  const { embeddingLimit, fullTextLimit } = countRecallLimit(searchMode);
  
  // 2. 集合元数据过滤(标签、时间等)
  const validCollectionIds = await filterCollectionByMetadata({
    datasetIds,
    collectionFilterMatch
  });
  
  // 3. 向量检索
  let embeddingRecallResults = [];
  if (embeddingLimit > 0) {
    // 3.1 生成查询向量
    const queryVectors = await Promise.all(
      queries.map(q => getVectorsByText({ input: q }))
    );
    
    // 3.2 向量数据库检索
    embeddingRecallResults = await Promise.all(
      queryVectors.map(async (vec) => {
        return recallFromVectorStore({
          datasetIds,
          vector: vec.vector,
          limit: embeddingLimit,
          similarity
        });
      })
    );
  }
  
  // 4. 全文检索
  let fullTextRecallResults = [];
  if (fullTextLimit > 0) {
    // 4.1 分词
    const queryTokens = queries.map(q => jiebaSplit(q));
    
    // 4.2 MongoDB 全文检索
    fullTextRecallResults = await Promise.all(
      queryTokens.map(tokens => {
        return MongoDatasetData.aggregate([
          {
            $match: {
              teamId,
              datasetId: { $in: datasetIds },
              $text: { $search: tokens.join(' ') }
            }
          },
          {
            $addFields: {
              score: { $meta: 'textScore' }
            }
          },
          {
            $sort: { score: -1 }
          },
          {
            $limit: fullTextLimit
          }
        ]);
      })
    );
  }
  
  // 5. 合并结果
  const mergedResults = datasetSearchResultConcat({
    embeddingRecallResults,
    fullTextRecallResults,
    embeddingWeight: 0.5
  });
  
  // 6. 重排序(ReRank)
  let reRankedResults = mergedResults;
  if (usingReRank && rerankModel) {
    reRankedResults = await datasetDataReRank({
      rerankModel,
      data: mergedResults,
      query: reRankQuery
    });
  }
  
  // 7. 获取完整数据
  const dataIds = reRankedResults.map(r => r.dataId);
  const dataList = await MongoDatasetData.find({
    _id: { $in: dataIds }
  }).populate('collectionId');
  
  // 8. Token 过滤(控制上下文长度)
  const filteredResults = filterByMaxTokens({
    data: dataList,
    maxTokens
  });
  
  return {
    searchRes: filteredResults,
    searchMode,
    limit: maxTokens,
    similarity,
    usingReRank
  };
}
```

**ReRank 重排序**:
```typescript
export const datasetDataReRank = async ({
  rerankModel,
  data,
  query
}: {
  rerankModel: RerankModelItemType;
  data: SearchDataResponseItemType[];
  query: string;
}) => {
  // 调用 ReRank 模型API
  const { results, inputTokens } = await reRankRecall({
    model: rerankModel.model,
    query,
    documents: data.map(item => item.q + '\n' + item.a)
  });
  
  // 合并原始分数和 ReRank 分数
  const mergeResult = data.map((item, index) => ({
    ...item,
    score: item.score * (1 - rerankWeight) + results[index].score * rerankWeight
  }));
  
  // 重新排序
  mergeResult.sort((a, b) => b.score - a.score);
  
  return mergeResult;
};
```

#### 6. 查询扩展(Query Extension)

**功能**: 使用 LLM 扩展用户查询,生成多个语义相关的查询词,提升召回率

```typescript
// packages/service/core/dataset/search/utils.ts
export const datasetSearchQueryExtension = async ({
  query,
  llmModel,
  embeddingModel,
  extensionBg,
  histories
}: {
  query: string;
  llmModel: string;
  embeddingModel: string;
  extensionBg: string;
  histories: ChatItemType[];
}) => {
  // 1. 去重相同查询(避免重复扩展)
  const filterSamQuery = (queries: string[]) => {
    return queries.filter(q => 
      hashStr(q.toLowerCase()) !== hashStr(query.toLowerCase())
    );
  };
  
  // 2. 调用 LLM 生成扩展查询
  const aiExtensionResult = await chatCompletion({
    model: llmModel,
    temperature: 0.1,
    messages: [
      {
        role: 'system',
        content: extensionBg || `根据用户问题,生成 2-3 个语义相关的搜索词,用于知识库检索。
要求:
1. 每个搜索词应该从不同角度描述用户意图
2. 包含同义词、相关概念
3. 保持简洁,不要生成完整句子`
      },
      ...histories.slice(-4),
      {
        role: 'user',
        content: query
      }
    ],
    response_format: {
      type: 'json_schema',
      json_schema: {
        name: 'search_queries',
        schema: {
          type: 'object',
          properties: {
            queries: {
              type: 'array',
              items: { type: 'string' },
              minItems: 2,
              maxItems: 3
            }
          }
        }
      }
    }
  });
  
  // 3. 解析结果
  const { queries = [], reRankQuery = query } = json5.parse(
    aiExtensionResult.content
  );
  
  // 4. 去重并返回
  const searchQueries = [query, ...filterSamQuery(queries)];
  
  return {
    searchQueries,
    reRankQuery,
    aiExtensionResult: {
      llmModel,
      embeddingModel,
      inputTokens: aiExtensionResult.usage.prompt_tokens,
      outputTokens: aiExtensionResult.usage.completion_tokens,
      query
    }
  };
};
```

#### 7. 集合标签过滤

**支持的过滤条件**:
```typescript
{
  tags: {
    $and: ['标签1', '标签2'],      // 必须同时包含
    $or: ['标签3', '标签4', null]  // 包含任一标签或无标签
  },
  createTime: {
    $gte: '2024-01-01',           // 创建时间大于等于
    $lte: '2024-12-31'            // 创建时间小于等于
  }
}
```

**过滤逻辑**:
```typescript
const filterCollectionByMetadata = async (
  collectionFilterMatch: string
): Promise<string[]> => {
  const filter = json5.parse(collectionFilterMatch);
  
  // 1. AND 标签过滤
  let validCollectionIds: Set<string>;
  if (filter.tags?.$and) {
    const andTags = await MongoDatasetCollectionTags.find({
      teamId,
      datasetId: { $in: datasetIds },
      tag: { $in: filter.tags.$and }
    });
    
    // 构建标签到集合的映射
    const tagToCollections = new Map<string, Set<string>>();
    andTags.forEach(item => {
      const tag = item.tag;
      if (!tagToCollections.has(tag)) {
        tagToCollections.set(tag, new Set());
      }
      item.collectionIds.forEach(id => 
        tagToCollections.get(tag).add(id)
      );
    });
    
    // 求交集(必须包含所有AND标签)
    const tagCollectionSets = Array.from(tagToCollections.values());
    validCollectionIds = tagCollectionSets.reduce((acc, set) => {
      return new Set([...acc].filter(x => set.has(x)));
    });
  }
  
  // 2. OR 标签过滤
  if (filter.tags?.$or) {
    const hasNull = filter.tags.$or.includes(null);
    const orTagNames = filter.tags.$or.filter(t => t !== null);
    
    const orCollections = await MongoDatasetCollectionTags.find({
      teamId,
      datasetId: { $in: datasetIds },
      tag: { $in: orTagNames }
    });
    
    const orCollectionIds = new Set<string>();
    orCollections.forEach(item => {
      item.collectionIds.forEach(id => orCollectionIds.add(id));
    });
    
    // 如果包含 null,添加无标签的集合
    if (hasNull) {
      const allCollections = await MongoDatasetCollection.find({
        teamId,
        datasetId: { $in: datasetIds }
      });
      
      allCollections.forEach(coll => {
        if (!coll.tags || coll.tags.length === 0) {
          orCollectionIds.add(String(coll._id));
        }
      });
    }
    
    // 与 AND 结果求交集
    if (validCollectionIds) {
      validCollectionIds = new Set(
        [...validCollectionIds].filter(x => orCollectionIds.has(x))
      );
    } else {
      validCollectionIds = orCollectionIds;
    }
  }
  
  // 3. 时间过滤
  if (filter.createTime) {
    const timeFilter: any = {};
    if (filter.createTime.$gte) {
      timeFilter.$gte = new Date(filter.createTime.$gte);
    }
    if (filter.createTime.$lte) {
      timeFilter.$lte = new Date(filter.createTime.$lte);
    }
    
    const timeFilteredCollections = await MongoDatasetCollection.find({
      teamId,
      datasetId: { $in: datasetIds },
      ...(validCollectionIds && {
        _id: { $in: Array.from(validCollectionIds) }
      }),
      createTime: timeFilter
    });
    
    validCollectionIds = new Set(
      timeFilteredCollections.map(c => String(c._id))
    );
  }
  
  return Array.from(validCollectionIds);
};
```

#### 8. 知识库删除

**级联删除流程**:
```typescript
// packages/service/core/dataset/controller.ts
export async function delDatasetRelevantData({
  datasets
}: {
  datasets: DatasetSchemaType[];
}): Promise<void> {
  const datasetIds = datasets.map(d => String(d._id));
  
  // 1. 查找所有集合
  const collections = await MongoDatasetCollection.find({
    datasetId: { $in: datasetIds }
  });
  
  // 2. 删除集合相关数据
  await Promise.all(
    collections.map(coll => delCollectionRelatedSource(coll))
  );
  
  // 3. 删除知识库记录
  await MongoDataset.deleteMany({
    _id: { $in: datasetIds }
  });
  
  // 4. 删除训练队列
  await MongoDatasetTraining.deleteMany({
    datasetId: { $in: datasetIds }
  });
  
  // 5. 删除数据块
  await MongoDatasetData.deleteMany({
    datasetId: { $in: datasetIds }
  });
  
  // 6. 删除数据块文本
  await MongoDatasetDataText.deleteMany({
    datasetId: { $in: datasetIds }
  });
  
  // 7. 删除向量数据
  await deleteDatasetDataVector({
    datasetIds,
    retry: 3
  });
  
  // 8. 删除 S3 文件
  const s3Source = getS3DatasetSource();
  await Promise.all(
    datasetIds.map(id => s3Source.deleteFolder(`dataset/${id}/`))
  );
  
  // 9. 删除图片
  await delImgByRelatedId({
    relatedIds: datasetIds,
    type: 'dataset'
  });
  
  // 10. 删除权限记录
  await MongoResourcePermission.deleteMany({
    resourceId: { $in: datasetIds }
  });
}
```

**集合删除**:
```typescript
export async function delCollectionRelatedSource(
  collection: DatasetCollectionSchemaType
): Promise<void> {
  const collectionId = String(collection._id);
  
  // 1. 删除数据块
  await MongoDatasetData.deleteMany({ collectionId });
  
  // 2. 删除数据块文本
  await MongoDatasetDataText.deleteMany({ collectionId });
  
  // 3. 删除训练队列
  await MongoDatasetTraining.deleteMany({ collectionId });
  
  // 4. 删除向量
  const dataList = await MongoDatasetData.find(
    { collectionId },
    'indexes'
  );
  const vectorIds = dataList.flatMap(d => 
    d.indexes.map(idx => idx.dataId)
  );
  await deleteVectorsByIds({ ids: vectorIds });
  
  // 5. 删除 S3 文件
  if (collection.fileId) {
    await getS3DatasetSource().delete(collection.fileId);
  }
  
  // 6. 删除集合记录
  await MongoDatasetCollection.deleteOne({ _id: collectionId });
}
```

## 性能优化

### 1. 向量检索优化
- **HNSW 索引**: 高性能近似最近邻搜索
- **批量检索**: 多个查询并行处理
- **相似度阈值**: 过滤低相关结果

### 2. 全文检索优化
- **Jieba 分词**: 中文智能分词
- **MongoDB 文本索引**: 快速关键词匹配
- **分词缓存**: 减少重复分词开销

### 3. 训练队列优化
- **批量插入**: 500条/批次,减少事务开销
- **分段事务**: 大数据量分多个事务,避免超时
- **重试机制**: 失败自动重试,指数退避

### 4. 检索缓存
- **查询结果缓存**: Redis 缓存热门查询
- **向量缓存**: 缓存常用查询向量
- **TTL 策略**: 1小时过期

## 监控与日志

### 关键指标
- **训练队列长度**: 监控待处理任务数
- **向量化耗时**: 平均处理时间
- **检索耗时**: P50/P95/P99 延迟
- **检索准确率**: 用户反馈统计

### 日志记录
```typescript
// 训练日志
addLog.info({
  type: 'dataset_training',
  datasetId,
  collectionId,
  mode: 'chunk',
  dataCount: 100,
  duration: 5000,
  tokens: 50000
});

// 检索日志
addLog.info({
  type: 'dataset_search',
  datasetIds,
  query,
  searchMode: 'embedding',
  resultCount: 10,
  duration: 200,
  embeddingTokens: 500,
  reRankTokens: 1000
});
```

## 最佳实践

### 1. 分块策略选择
- **通用文档**: `chunk` 模式 + 512 tokens 块大小
- **长文档**: 增大块大小到 1024,启用重叠
- **问答知识**: `qa` 模式,自动生成问答对
- **代码文档**: 自定义分隔符(函数/类边界)

### 2. 检索参数调优
- **高精度场景**: `embedding` + reRank + 相似度 0.7
- **高召回场景**: `mixedRecall` + 相似度 0.3
- **长上下文**: 增大 maxTokens 到 5000+

### 3. 性能优化
- **批量导入**: 使用批量 API,减少请求次数
- **异步处理**: 大文件后台训练,不阻塞用户
- **标签管理**: 合理使用标签,加速过滤

### 4. 成本控制
- **向量模型**: 平衡精度与成本(text-embedding-ada-002 vs text-embedding-3-small)
- **ReRank**: 仅在高价值场景使用
- **查询扩展**: 控制扩展数量(2-3个)

## 扩展功能

### 1. 外部数据源同步
- **飞书文档**: OAuth 授权 + Webhook 实时同步
- **语雀文档**: API 轮询 + 增量更新
- **API 数据源**: 自定义接口 + Cron 定时同步

### 2. 图片索引
- **VLM 解析**: 使用视觉语言模型提取图片描述
- **OCR**: 提取图片中的文字
- **图片向量化**: 多模态向量检索

### 3. 数据增强
- **标题增强**: 自动为每个块添加标题
- **同义词扩展**: 增加同义词提升召回
- **摘要生成**: LLM 生成块摘要

## 故障排查

### 常见问题

**1. 训练卡住不动**
- 检查队列积压: `MongoDatasetTraining.countDocuments({ lockTime: { $lt: new Date() } })`
- 检查 Worker 状态: 查看日志是否有异常
- 手动重置锁: `lockTime` 设为过去时间

**2. 检索结果不准确**
- 调整相似度阈值
- 启用 ReRank
- 检查向量模型是否匹配
- 查看查询扩展效果

**3. 向量数据库连接失败**
- 检查连接配置(PGVector/Milvus URL)
- 查看数据库日志
- 验证网络连通性

## 相关文档
- [数据持久化层](./05-data-persistence-layer.md)
- [外部服务集成层](./06-external-services-layer.md)
- [工作流引擎](./07-workflow-engine.md)
