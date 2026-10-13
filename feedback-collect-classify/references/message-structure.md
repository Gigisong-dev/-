# 消息统一结构

## 概述

从飞书群聊拉取到的原始消息结构（平台原生结构）可能因接口版本或字段差异而不稳定。本技能在处理前统一整理为下面的标准结构，后续的收集、分类、上下文补全均基于该结构进行。

## 统一消息结构

```json
{
  "group_name": "本地版答疑群",
  "time_window": "2026-07-01 00:00 to 2026-07-07 23:59 Asia/Shanghai",
  "messages": [
    {
      "message_id": "optional",
      "message_time": "2026-07-01 10:15",
      "sender": "张三",
      "text": "昨天还能正常使用 lark-cli，今天就报错了",
      "reply_to": "optional",
      "thread_id": "optional",
      "images": [
        {
          "image_key": "img_v2_041b-xxxxx",
          "url": "https://open.feishu.cn/open-apis/im/v1/images/img_v2_041b-xxxxx"
        }
      ]
    }
  ]
}
```

## 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `group_name` | string | 群聊名称 |
| `time_window` | string | 本次拉取的时间窗口（含时区） |
| `messages` | array | 消息列表 |
| `message_id` | string | 消息唯一标识（可选，用于去重/追溯） |
| `message_time` | string | 消息发送时间，格式 `yyyy-MM-dd HH:mm` |
| `sender` | string | 发送者姓名；无法获取时用"匿名用户" |
| `text` | string | 消息文本内容 |
| `reply_to` | string | 被回复的消息标识（可选） |
| `thread_id` | string | 所属话题/线程标识（可选） |
| `images` | array | 消息中的图片列表 |
| `image_key` | string | 图片在飞书中的 image_key |
| `url` | string | 图片 URL（用于下载/展示） |

## 整理规则

1. 从原始飞书消息中提取 `message_id`、`create_time`、`sender`、`content.text`、`reply`/`thread` 等字段
2. 图片从消息内容中单独提取为 `images` 数组（`image_key` + `url`）
3. `message_time` 统一转换为 `yyyy-MM-dd HH:mm`（本地时区）
4. 消息按时间升序排列
5. 若原始接口分页拉取，需合并分页并去重（按 `message_id`）

## 上下文补抓

对于依赖前文的短消息（"还是不行""这个怎么处理""回复某人"等），需要补充上下文：

- **同线程补抓**：按 `thread_id` 拉取该线程的消息
- **相邻时间段补抓**：向前拉取同时间段最近 3-5 条消息
- **回复链补抓**：按 `reply_to` 追溯被回复的原始问题

补抓到的上下文合并入统一结构，并在处理时生成 `context_summary`（一句话上下文总结）。
