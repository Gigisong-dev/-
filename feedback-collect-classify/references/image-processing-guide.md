# 图片处理指南

## 概述

本文档详细说明 feedback-collect-classify 技能中的图片处理流程、配置方法和最佳实践。

## 图片处理流程

### 完整流程图

```
群聊消息
  ↓
提取图片信息
  ├─ image_key
  ├─ url
  └─ 元数据（尺寸、格式等）
  ↓
下载图片到本地
  ↓
上传到云空间
  ↓
获取文件信息
  ├─ file_token
  └─ 访问URL
  ↓
根据配置写入字段
  ├─ 附件类型: file_token
  ├─ URL类型: url
  └─ 混合类型: 两者都保存
  ↓
关联到记录
```

## 详细步骤

### 1. 图片提取

#### 从消息中提取

**飞书消息结构**:
```json
{
  "message_id": "om_xxxxxxxxx",
  "content": {
    "image_key": "img_v2_041b-xxxxx",
    "images": [
      {
        "image_key": "img_v2_041b-xxxxx",
        "url": "https://open.feishu.cn/open-apis/im/v1/images/img_v2_041b-xxxxx"
      }
    ]
  },
  "msg_type": "image"
}
```

**提取逻辑**:
1. 检查 `msg_type` 是否为 `image`
2. 从 `content.image_key` 获取图片 key
3. 从 `content.images[0].url` 获取图片 URL
4. 提取其他元数据（如需要）

#### 支持的图片格式

- **JPG/JPEG**: `.jpg`, `.jpeg`
- **PNG**: `.png`
- **GIF**: `.gif`
- **WebP**: `.webp`
- **BMP**: `.bmp`
- **其他飞书支持的格式**

### 2. 图片下载

#### 下载命令

```bash
lark-cli im +image-download \
  --image-key "img_v2_041b-xxxxx" \
  --output "/tmp/image_20260819_143030.jpg" \
  --as user
```

#### 文件命名规则

```
/tmp/feedback_image_{timestamp}_{index}.{extension}
```

**示例**:
- `/tmp/feedback_image_20260819_143030_0.jpg`
- `/tmp/feedback_image_20260819_143030_1.png`

#### 下载参数

- **超时时间**: 30秒
- **重试次数**: 3次
- **临时目录**: `/tmp/`
- **文件大小限制**: 10MB

#### 错误处理

```python
try:
    # 下载图片
    download_result = download_image(image_key, output_path)
except DownloadError as e:
    # 记录错误
    log_error(f"图片下载失败: {e}")
    # 在字段中填写错误描述
    image_field = f"图片下载失败: {e}"
except TimeoutError:
    log_error("图片下载超时")
    image_field = "图片下载超时"
```

### 3. 图片上传

#### 上传命令

```bash
lark-cli drive +file-upload \
  --file "/tmp/image_20260819_143030_0.jpg" \
  --parent-folder-token "folder_xxxxxxxxx" \
  --as user
```

#### 云空间组织

建议的文件夹结构：

```
反馈图片/
├── 2026/
│   ├── 08/
│   │   ├── 19/
│   │   │   ├── 本地版答疑群/
│   │   │   │   ├── image_001.jpg
│   │   │   │   └── image_002.png
│   │   │   └── 产品反馈群/
│   │   │       └── image_003.jpg
│   │   └── 20/
│   └── 09/
└── 2027/
```

#### 上传参数

- **超时时间**: 60秒
- **重试次数**: 3次
- **并发上传**: 最多3个同时上传
- **文件大小限制**: 10MB

#### 返回结果

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "file_token": "file_xxxxxxxxx",
    "url": "https://xxx.feishu.cn/file/file_xxxxxxxxx",
    "name": "image_20260819_143030_0.jpg",
    "size": 1024000,
    "type": "image/jpeg"
  }
}
```

### 4. 字段写入

#### 附件类型

**配置**:
```json
{
  "image_field_type": "attachment",
  "image_field_name": "反馈图片"
}
```

**写入格式**:
```json
{
  "反馈图片": ["file_token_xxxxxxxxx"]
}
```

**特点**:
- 直接使用文件 token
- 支持在多维表格中预览
- 支持下载
- 占用存储空间

#### URL类型

**配置**:
```json
{
  "image_field_type": "url",
  "image_field_name": "反馈图片"
}
```

**写入格式**:
```json
{
  "反馈图片": "https://xxx.feishu.cn/file/file_xxxxxxxxx"
}
```

**特点**:
- 使用图片 URL
- 点击可跳转查看
- 不占用存储空间
- 依赖外部链接

#### 混合类型

**配置**:
```json
{
  "image_field_type": "mixed",
  "image_field_name": "反馈图片",
  "attachment_field": "反馈图片附件",
  "url_field": "反馈图片链接"
}
```

**写入格式**:
```json
{
  "反馈图片附件": ["file_token_xxxxxxxxx"],
  "反馈图片链接": "https://xxx.feishu.cn/file/file_xxxxxxxxx"
}
```

**特点**:
- 同时保存 token 和 URL
- 提供最大的灵活性
- 占用两个字段

## 配置选项

### 基本配置

```json
{
  "image_processing": {
    "enabled": true,
    "field_type": "attachment",
    "field_name": "反馈图片",
    "upload_folder": "folder_xxxxxxxxx",
    "temp_dir": "/tmp/",
    "max_size": 10485760,
    "timeout": 60,
    "retry": 3
  }
}
```

### 高级配置

```json
{
  "image_processing": {
    "enabled": true,
    "field_type": "mixed",
    "field_name": "反馈图片",
    "attachment_field": "反馈图片附件",
    "url_field": "反馈图片链接",
    "upload_folder": "folder_xxxxxxxxx",
    "folder_structure": "by_date_and_group",
    "temp_dir": "/tmp/",
    "max_size": 10485760,
    "timeout": 60,
    "retry": 3,
    "concurrent_uploads": 3,
    "supported_formats": ["jpg", "jpeg", "png", "gif", "webp", "bmp"],
    "resize_options": {
      "enabled": false,
      "max_width": 1920,
      "max_height": 1080,
      "quality": 85
    }
  }
}
```

## 错误处理

### 常见错误及处理

#### 1. 下载失败

**错误类型**: `DownloadError`

**处理方式**:
```json
{
  "反馈图片": "图片下载失败: 网络连接超时"
}
```

**恢复策略**:
- 重试3次
- 记录详细错误日志
- 不影响其他字段处理

#### 2. 上传失败

**错误类型**: `UploadError`

**处理方式**:
```json
{
  "反馈图片": "图片上传失败: 存储空间不足"
}
```

**恢复策略**:
- 重试3次
- 检查云空间权限
- 检查存储空间

#### 3. 格式不支持

**错误类型**: `FormatError`

**处理方式**:
```json
{
  "反馈图片": "图片格式不支持: .tiff"
}
```

**恢复策略**:
- 跳过该图片
- 记录警告日志
- 继续处理其他图片

#### 4. 文件过大

**错误类型**: `SizeError`

**处理方式**:
```json
{
  "反馈图片": "图片文件过大: 15MB (限制: 10MB)"
}
```

**恢复策略**:
- 尝试压缩（如果启用）
- 跳过该图片
- 记录警告日志

### 错误日志格式

```json
{
  "timestamp": "2026-08-19T14:30:30Z",
  "level": "ERROR",
  "message": "图片处理失败",
  "details": {
    "image_key": "img_v2_041b-xxxxx",
    "error_type": "DownloadError",
    "error_message": "网络连接超时",
    "retry_count": 3,
    "feedback_id": "om_xxxxxxxxx"
  }
}
```

## 性能优化

### 并发处理

**下载并发**:
```python
# 并发下载多个图片
with ThreadPoolExecutor(max_workers=3) as executor:
    futures = [
        executor.submit(download_image, img) 
        for img in images
    ]
    results = [f.result() for f in futures]
```

**上传并发**:
```python
# 并发上传多个图片
with ThreadPoolExecutor(max_workers=3) as executor:
    futures = [
        executor.submit(upload_image, img) 
        for img in downloaded_images
    ]
    results = [f.result() for f in futures]
```

### 缓存策略

**下载缓存**:
- 使用本地临时目录
- 设置合理的过期时间
- 定期清理缓存文件

**上传缓存**:
- 缓存上传结果
- 避免重复上传相同图片
- 使用文件 hash 作为缓存键

### 批量操作

**批量下载**:
```bash
# 一次性下载多个图片
for img_key in image_keys:
    lark-cli im +image-download \
      --image-key "$img_key" \
      --output "/tmp/${img_key}.jpg" \
      --as user
```

**批量上传**:
```bash
# 一次性上传多个图片
for file in /tmp/feedback_image_*; do
    lark-cli drive +file-upload \
      --file "$file" \
      --parent-folder-token "folder_xxxxxxxxx" \
      --as user
done
```

## 最佳实践

### 1. 文件夹组织

**按日期和群聊组织**:
```
反馈图片/
├── 2026/08/19/
│   ├── 本地版答疑群/
│   └── 产品反馈群/
```

**优点**:
- 便于查找和管理
- 避免单个文件夹文件过多
- 便于定期清理

### 2. 文件命名

**使用时间戳和序号**:
```
image_20260819_143030_0.jpg
image_20260819_143030_1.png
```

**优点**:
- 避免文件名冲突
- 便于按时间排序
- 包含时间信息

### 3. 错误处理

**优雅降级**:
- 图片处理失败不影响其他字段
- 提供详细的错误信息
- 记录完整的错误日志

**重试机制**:
- 网络错误自动重试
- 设置合理的重试次数
- 指数退避策略

### 4. 资源管理

**及时清理**:
- 处理完成后删除临时文件
- 定期清理过期缓存
- 监控存储空间使用

**��发控制**:
- 限制并发下载数量
- 限制并发上传数量
- 避免资源耗尽

### 5. 监控和日志

**详细日志**:
- 记录每个处理步骤
- 记录错误和警告
- 记录性能指标

**监控指标**:
- 图片处理成功率
- 平均处理时间
- 错误类型分布

## 安全考虑

### 1. 权限控制

**下载权限**:
- 确保有群聊图片访问权限
- 验证用户身份

**上传权限**:
- 确保有云空间写入权限
- 验证目标文件夹权限

### 2. 数据安全

**临时文件**:
- 使用安全临时目录
- 设置适当的文件权限
- 及时删除敏感文件

**传输安全**:
- 使用 HTTPS 协议
- 验证 SSL 证书
- 加密敏感数据

### 3. 访问控制

**文件访问**:
- 设置合理的访问权限
- 避免公开敏感图片
- 定期审计访问日志

---

**最后更新**: 2026-08-19