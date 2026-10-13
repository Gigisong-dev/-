# 输出格式

## 概述

技能最终输出**严格 JSON**，不输出解释性段落、Markdown、表格、代码块或列表。输出结构固定为四个顶层字段：`config_summary`、`window_summary`、`dimension_summary`、`collected_items`。

## 完整输出结构

```json
{
  "config_summary": {
    "base_token": "",
    "table_token": "",
    "group_name": "",
    "time_window": "",
    "image_field_enabled": false,
    "custom_field_mapping": false,
    "classification_standard": "默认四分类"
  },
  "window_summary": {
    "group_name": "",
    "time_window": "",
    "total_messages": 0,
    "collected_count": 0,
    "images_processed": 0
  },
  "dimension_summary": {
    "技术问题": 0,
    "产品建议": 0,
    "使用咨询": 0,
    "待划定": 0
  },
  "collected_items": [
    {
      "提交时间": "",
      "standard_title": "",
      "反馈人": "",
      "反馈原文": "",
      "附件": "",
      "一级分类": "",
      "二级分类": "",
      "跟进状态": "",
      "回答人": "",
      "回复内容": "",
      "来自群聊": "",
      "是否命中知识库": ""
    }
  ]
}
```

**说明**：
- `collected_items` 中每条记录的字段顺序与用户配置的多维表格字段一致
- 用户提供自定义分类时，`dimension_summary` 使用用户定义的分类作为键
- 用户提供自定义字段映射时，`collected_items` 使用映射后的实际字段名

## 字段规则（按输出顺序）

### 提交时间
- 类型：datetime
- 格式：`yyyy/MM/dd HH:mm`
- 取值：消息发送时间

### standard_title
- 类型：文本
- 描述：标准化后的问题标题，用于后续归并、去重和知识库检索

### 反馈人
- 类型：user（多选）
- 取值：消息发送者名称；匿名时用"匿名用户"

### 反馈原文
- 类型：文本
- 描述：原始消息文本，保持原样

### 附件
- 类型：附件数组或 URL
- 取值：图片文件 token 数组或 URL 字符串

### 一级分类
- 类型：选择（单选）
- 默认只允许：`技术问题` / `产品建议` / `使用咨询` / `待划定`
- 用户提供自定义分类时：只允许用户定义的值
- 约束：不得为空字符串

### 二级分类
- 类型：文本
- 描述：子分类，根据具体问题类型填写，可为空字符串

### 跟进状态
- 类型：选择（单选）
- 只允许：`待跟进` / `已完成` / `重复问题`
- 规则：有回复则已完成，无回复则待跟进，与已有记录重复则标记为重复问题

### 回答人
- 类型：user（多选）
- `跟进状态` 为 `已完成` 时填写回答者姓名
- `待跟进` 或 `重复问题` 时为空字符串 `""`

### 回复内容
- 类型：文本
- 群聊中对反馈的回复文本
- `跟进状态` 为 `待跟进` 或 `重复问题` 时为空字符串 `""`

### 来自群聊
- 类型：选择（单选）
- 取值：用户配置的群名

### 是否命中知识库
- 类型：选择（单选）
- **用户未提供知识库时**：不填充，空字符串 `""`，且不影响 `一级分类`
- 用户提供知识库时：仅当 `一级分类` 为 `技术问题` / `产品建议` / `使用咨询` 时填写 `是` 或 `否`
- 其他分类（如 `待划定`）为空字符串 `""`

## 校验清单

输出前逐项校验（与 SKILL.md 的 Critical 强约束一致）：
1. 输出格式：仅一个 JSON 对象，无附加文本
2. 顶层字段：仅 `config_summary` / `window_summary` / `dimension_summary` / `collected_items`
3. 配置信息：`config_summary` 含全部配置信息
4. 记录字段：`collected_items` 每项字段与用户配置一致
5. 空值处理：无值字段填空字符串 `""` / 数字 `0` / 布尔 `false` / 空数组 `[]`
6. 分类约束、跟进状态约束、回复字段一致性、知识库字段约束、待划定约束
7. 统计一致：`dimension_summary` 各计数之和 == `window_summary.collected_count` == `collected_items` 数量；`collected_count` ≤ `total_messages`
8. 图片处理约束：`images_processed` 准确、图片字段按配置类型填写
