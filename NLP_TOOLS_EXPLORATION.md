# Brand Listener NLP工具和配置全景探索

## 项目概览
- **主目录**: `d:/wmz/Brand Listener/Brand Listener/`
- **内网数据平台**: FOLO（微博RPA导出）
- **监听平台**: 微博、小红书（XHS）、抖音等
- **行业垂直**: 口腔护理产品
- **当前数据量**: 1200+条帖子，277条已补齐评论（21.7%）

---

## 1. 配置管理体系

### 1.1 主配置文件

**位置**: `d:/wmz/Brand Listener/Brand Listener/.env`

```
FOLO_EXPORT_PATH=D:/wmz/FOLO
FOLO_EXPORT_PATTERN=*.db
FOLO_UPDATE_LOOKBACK_HOURS=720  # 30天回溯期

# 小红书API配置
XHS_API_TOKEN=sk-ff654ace-...  # Proxy API令牌
XHS_COOKIES=...                # Cookie（OCR下载图片用）
XHS_MONITOR_TARGETS=[...]      # 固定博主监听列表（10个品牌）
XHS_SEARCH_KEYWORDS=[]         # 可选的关键词搜索列表
XHS_SEARCH_NUM=20              # 每次搜索返回数量
XHS_MIN_LIKES=0                # 最低点赞数筛选
XHS_MIN_FANS=0                 # 最低粉丝数筛选

# Report Engine配置
REPORT_ENGINE_API_KEY=tp-cgym9j3ztb1...
REPORT_ENGINE_BASE_URL=https://token-plan-cn.xiaomimimo.com/v1
REPORT_ENGINE_MODEL_NAME=mimo-v2.5-pro
```

**对应代码管理**:
- 文件: `d:/wmz/Brand Listener/Brand Listener/src/utils/config.py`
- 函数: `get_env_var()`, `get_env_int()`, `get_env_bool()`
- 预定义configs:
  - `get_folo_config()` - FOLO导出配置
  - `get_xhs_agent_config()` - 小红书Agent配置
  - `get_ocr_agent_config()` - OCR Agent配置
  - `get_api_config()` - API服务配置

### 1.2 品牌配置管理

**位置**: `d:/wmz/Brand Listener/Brand Listener/data/brands.json`

**管理类**: `BrandConfigManager` in `src/brand_config.py`
- CRUD操作（增删改查）
- 支持多品牌同时监听
- 存储Weibo UID、平台配置、启用状态等

---

## 2. NLP工具和模型

### 2.1 情感分析系统 ⭐ (核心)

**模型**: WeiboMultilingualSentiment (DistilBERT)
- **支持语言**: 中文、英文、西班牙文、阿拉伯文、日文、韩文等22种语言
- **情感等级**: 5级分类 → 3级简化
  - `非常负面` / `负面` → `negative`
  - `中性` → `neutral`
  - `正面` / `非常正面` → `positive`

**位置**: `d:/wmz/Brand Listener/BettaFish/InsightEngine/tools/sentiment_analyzer.py`

**关键类**:
```python
class WeiboMultilingualSentimentAnalyzer:
    - initialize()           # 加载模型（首次或从本地）
    - analyze_single_text()  # 单文本分析
    - analyze_batch()        # 批量分析（推荐）
    - analyze_query_results()# 查询结果分析
    - disable/enable()       # 运行时开关
```

**性能优化**:
- 自动选择设备: CUDA > MPS (Apple) > CPU
- 模型本地缓存: `BettaFish/SentimentAnalysisModel/WeiboMultilingualSentiment/model/`
- 模型来源: `tabularisai/multilingual-sentiment-analysis`（Hugging Face）
- 国内镜像回退: `https://hf-mirror.com`

**使用示例**:
```python
from src.agents.searcher.comment_analyzer import analyze_sentiment

# 单条分析
result = analyze_sentiment("今天天气特别好！")
# 返回: SentimentResult(sentiment_label='正面', confidence=0.95, ...)

# 批量分析
texts = ["很好用", "很差劲", "还可以"]
batch_result = analyze_sentiment(texts)
# 返回: BatchSentimentResult(results=[...], success_count=3, ...)
```

### 2.2 评论情感分析与卖点提取

**位置**: `d:/wmz/Brand Listener/Brand Listener/src/agents/searcher/comment_analyzer.py`

**功能**:
```python
def analyze_comments(comments: List[Dict]) -> Dict:
    """
    返回:
    {
        "total_comments": int,
        "sentiment": {"positive": int, "negative": int, "neutral": int},
        "positive_ratio": float,
        "selling_points": [
            {"tag": str, "count": int, "sentiment": str},
            ...
        ],
        "details": [
            {
                "content": str,
                "sentiment": str,
                "selling_points": [str],
                "nickname": str,
            },
            ...
        ]
    }
    """
```

**卖点标签库** (100+个标签):

| 正面卖点 | 对应关键词 |
|---------|----------|
| 刷毛柔软 | 刷毛软、刷毛细腻、柔软 |
| 清洁力强 | 清洁力、刷干净、洁净 |
| 美白效果 | 美白、变白、亮白、去黄 |
| 性价比高 | 性价比、划算、便宜、实惠 |
| 护龈 | 护龈、温和、不伤牙龈 |
| 颜值高 | 好看、精致、高级感 |
| 噪音小 | 静音、声音小 |
| 防水好 | 防水、不漏水 |

| 负面卖点 | 对应关键词 |
|---------|----------|
| 牙龈出血 | 出血、流血、牙龈疼 |
| 味道不好 | 难闻、味道奇怪、臭味 |
| 噪音大 | 吵、声音大、很吵 |
| 质量差 | 掉毛、断了、坏了、劣质 |

### 2.3 内容打标系统

**位置**: `d:/wmz/Brand Listener/Brand Listener/src/agents/searcher/content_tagging_agent.py`

**功能**: 每条帖子自动提取 2-5 个内容标签

**标签体系** (3大类):

#### 宣发手法 (14个)
- 明星代言、明星互动、新品发布、预热造势
- 抽奖送礼、限时优惠、直播带货、联名合作
- 线下活动、互动话题、KOL种草、教程科普
- 场景营销、品牌故事、网感互动、品牌调性
- 品牌资讯、开箱种草、粉丝运营、产品玩梗、内容共创

#### 文案关键词 (9个)
- 限量限定、独家专利、成分功效、美白概念
- 口腔护理、冲牙器、电动牙刷、小光环、颜值外观、口碑背书、育儿亲子

#### 热点元素 (8个)
- 女神节、春节、情人节、618、双十一、世界读书日
- 动漫IP、春季节气、夏季

**品牌联动分类** (6类):
- 渠道合作 (屈臣氏、丝芙兰、天猫、京东等)
- 展会合作 (礼品展、漫展、美博会等)
- IP联名 (罗小黑、迪士尼、宝可梦等)
- 明星合作 (肖战、李宏毅、李佳琦等)
- 媒体合作 (央视、综艺等)
- 公益合作

### 2.4 关键词字典系统

**位置**: `d:/wmz/Brand Listener/Brand Listener/src/agents/searcher/keyword_dicts.py`

#### 品牌关键词 (12个品牌)
```python
BRAND_KEYWORDS = {
    "usmile": ["usmile", "笑容加"],
    "参半": ["参半"],
    "倍至": ["倍至", "bixdo"],
    "佳洁士": ["佳洁士", "Crest"],
    "高露洁": ["高露洁", "Colgate"],
    "BOP": ["BOP", "波普专研"],
    "冷酸灵": ["冷酸灵"],
    "舒客": ["舒客", "Saky"],
    "云南白药": ["云南白药"],
    "黑人": ["黑人牙膏"],
    "狮王": ["狮王", "Lion"],
    "欧乐B": ["欧乐B", "Oral-B"],
    "飞利浦": ["飞利浦", "Sonicare"],
    "松下": ["松下", "Panasonic"],
}
```

#### 产品关键词 (11个类别)
- 电动牙刷、冲牙器、美白牙膏、抗敏牙膏、儿童牙膏
- 漱口水、牙线、牙贴、小光环、台式冲牙器、Y1 Pro等

#### 成分关键词 (15个成分)
- 氟化物、氨基酸、小苏打、益生菌、酵素
- 竹盐、蜂胶、茶多酚、木糖醇、薄荷
- 活性炭、过氧化氢、羟基磷灰石、柠檬酸锌、生物活性玻璃、无氟

---

## 3. 数据流和缓存策略

### 3.1 数据存储结构

**主存储**: `d:/wmz/Brand Listener/Brand Listener/data/entries_store.json`
- **大小**: 14.5MB（1200+条帖子）
- **格式**: JSON数组，每条条目包含:
  ```json
  {
    "id": "271433303730424832",
    "platform": "weibo|xhs|douyin|...",
    "source_url": "https://weibo.com/...",
    "title": "...",
    "content": "...",
    "published_at": "2026-06-01T12:34:56Z",
    "engagement_metrics": {
      "nickname": "...",
      "author": "...",
      "liked_count": "1.5万",
      "comment_count": "128",
      "share_count": "45",
      "ai_tags": ["新品发布", "明星代言"],
      "collab_partners": ["渠道合作"],
      "collab_entity_names": ["屈臣氏"],
      "comments": [...],           // 拉取的原始评论
      "comment_analysis": {         // 情感分析结果
        "total_comments": 128,
        "sentiment": {"positive": 89, "negative": 5, "neutral": 34},
        "positive_ratio": 0.69,
        "selling_points": [
          {"tag": "刷毛柔软", "count": 24, "sentiment": "positive"},
          ...
        ],
        "details": [...]
      }
    }
  }
  ```

### 3.2 评论拉取缓存策略

**位置**: `d:/wmz/Brand Listener/Brand Listener/server.py` - `_fetch_and_store_comments()`

**省钱策略**:
```python
def _fetch_and_store_comments(entries: list, token: str, max_per_run: int = 50):
    """
    - 仅拉取近30天帖子（旧帖子评论不再变化）
    - 跳过低互动帖子（点赞<10）
    - 每篇最多1页（10条评论）
    - 每次运行最多处理max_per_run条（默认50）
    - 已有评论的帖子直接跳过
    - 评论完成后自动存入entries_store
    """
```

**缓存逻辑**:
1. 新增帖子进入 `new_entries` 列表
2. 后台线程遍历，跳过已有评论的
3. 调用 `XHSNoteComments.get_all_comments()`
4. 管道式执行: `analyze_comments()` → 情感分析 + 卖点提取
5. 结果存入 `entry["engagement_metrics"]["comments"]` 和 `["comment_analysis"]`
6. 原子写入 `entries_store.json`（带文件锁保护）

**API获取端点**:
```
GET /api/xhs/comments/{note_id}
  ?sort=latest
  &force=1  # 强制重新拉取，否则优先使用缓存
```

### 3.3 小红书API调用

**位置**: `d:/wmz/Brand Listener/Brand Listener/src/agents/searcher/xhs_api/note_comments.py`

**API服务**: 内网Proxy API
- **Base URL**: `https://proxy-api.ainm.store`
- **端点**: `/p/xiaohongshu/get-note-comment/v2`
- **认证**: Bearer token (在.env中配置)

**关键参数**:
```python
{
  "noteId": note_id,           # 笔记ID
  "sort": "latest|normal",     # 排序方式
  "lastCursor": "...",         # 分页游标
}
```

**错误处理**:
- 301错误: 自动重试（指数退避）
- 响应解析: 支持gzip压缩
- Cursor提取: 处理嵌套JSON格式

---

## 4. Agent流水线架构

### 4.1 LangGraph工作流

**位置**: `d:/wmz/Brand Listener/Brand Listener/langgraph/workflow.py`

**当前运行配置**:
```
官方更新Agent (FOLO + 小红书博主)
    ↓
内容分类Agent
    ↓
内容打标Agent (加评论分析)
    ↓
END
```

**支持但未启用的Agent**:
- BrandCultureListeningAgent (品牌文化)
- SocialMediaFeedbackAgent (社媒反馈)
- ShoppingFeedbackAgent (购物反馈)
- CompetitorInsightAgent (竞对分析)
- UserFeedbackAnalystAgent (用户反馈分析)
- TemplateReportAgent (报告生成)
- TaskDispatcherAgent (任务调度)

### 4.2 数据转换层

**位置**: `d:/wmz/Brand Listener/Brand Listener/src/report_engine/data_converter.py`

**功能**: 将entries_store转换为3份Markdown报告
- `report_query`: 品牌声量统计、平台分布、趋势分析
- `report_media`: KOL合作、内容类型、热门帖子
- `report_insight`: 标签分析、关键词、用户反馈

---

## 5. 行业垂直过滤系统

### 5.1 口腔护理行业识别

**位置**: `d:/wmz/Brand Listener/Brand Listener/server.py`

**口腔关键词** (60+个):
- 品牌: 参半、倍至、佳洁士、高露洁、BOP等
- 产品: 牙膏、牙刷、电动牙刷、冲牙器、漱口水等
- 症状: 口臭、蛀牙、龋齿、牙龈出血等
- 服务: 洗牙、补牙、拔牙、正畸等

**黑名单过滤**:
```python
_BLOCKED_SOURCES = [
    "1x.com", "macrumors.com", "github.blog",  # 技术网站
    "x.com/elonmusk", "x.com/OpenAI",          # 无关账户
    "youtube.com/channel/...",                  # 无关频道
]
```

**诊所/医院广告过滤**:
- 昵称包含: "口腔医院、诊所、牙科诊所、齿科"等
- 内容包含: "种植牙、隐形矫正、义诊、公益口腔"等

### 5.2 内容去重系统

**函数**: `_dedup_entries_store()`
- 对title+content计算MD5 hash
- 去掉完全重复的帖子
- 记录去重日志

---

## 6. 关键性能指标和优化

### 6.1 批处理优化

| 任务 | 批大小 | 超时 | 说明 |
|------|-------|------|------|
| 情感分析 | N条 | - | 支持批处理 |
| 评论拉取 | 50条/次 | 60s | max_per_run可调 |
| OCR处理 | 3张 | 60s | OCR_MAX_IMAGES |
| 日期回溯 | - | - | 30天/720小时 |

### 6.2 缓存策略

| 缓存对象 | 策略 | 更新方式 |
|---------|------|--------|
| 评论数据 | 持久化到entries_store | 新增帖子触发拉取 |
| 情感分析结果 | 随评论存储 | 与评论同生命周期 |
| 已打标帖子 | 标记ai_tags，跳过 | 幂等处理，避免重复 |
| 模型权重 | 本地磁盘 | HuggingFace或国内镜像下载 |

### 6.3 成本节约

1. **API Token节约**:
   - 仅对新增帖子拉取评论
   - 跳过低互动帖（<10赞）
   - 仅拉1页评论（≈10条）
   - 30天后不再拉取

2. **计算优化**:
   - 模型本地缓存，无需重复下载
   - 批量情感分析，减少前向传播
   - GPU加速（自动选择CUDA/MPS/CPU）

3. **存储优化**:
   - 去重存储（MD5 hash）
   - 只保留30天内数据
   - 诊所广告过滤（减少噪声）

---

## 7. 使用指南

### 7.1 启用/禁用情感分析

```python
# 在sentiment_analyzer.py中全局开关
SENTIMENT_ANALYSIS_ENABLED = False  # 禁用

# 或运行时切换
from BettaFish.InsightEngine.tools.sentiment_analyzer import (
    enable_sentiment_analysis,
    disable_sentiment_analysis
)

enable_sentiment_analysis()
disable_sentiment_analysis(reason="...", drop_state=True)
```

### 7.2 执行评论补拉

```bash
# 自动触发（后台）
curl -X POST http://localhost:8000/api/pipeline/run

# 手动补拉特定帖子（调大max_per_run）
_fetch_and_store_comments(
    entries=selected_entries,
    token=api_token,
    max_per_run=200  # 补拉200条
)
```

### 7.3 查询缓存的评论和分析

```python
# API查询
GET /api/xhs/comments/{note_id}?force=0  # 优先用缓存

# 本地查询
from src.utils.config import get_api_config
import json

with open("data/entries_store.json") as f:
    entries_store = json.load(f)
    
for entry in entries_store:
    if entry.get("id") == note_id:
        comments = entry["engagement_metrics"].get("comments", [])
        analysis = entry["engagement_metrics"].get("comment_analysis", {})
        break
```

---

## 8. 文件清单

### 核心NLP文件
| 文件 | 功能 |
|------|------|
| `src/agents/searcher/comment_analyzer.py` | 评论情感分析 + 卖点提取 |
| `src/agents/searcher/content_tagging_agent.py` | 内容打标（宣发手法、热点元素等） |
| `src/agents/searcher/keyword_dicts.py` | 品牌/产品/成分关键词库 |
| `BettaFish/InsightEngine/tools/sentiment_analyzer.py` | 多语言情感分析（核心模型） |
| `src/agents/searcher/xhs_api/note_comments.py` | 小红书评论采集 |

### 配置和工具文件
| 文件 | 功能 |
|------|------|
| `src/utils/config.py` | 配置管理（环境变量加载） |
| `src/brand_config.py` | 品牌配置CRUD |
| `server.py` | FastAPI主服务（含_fetch_and_store_comments等函数） |
| `langgraph/workflow.py` | 多Agent流水线 |
| `src/report_engine/data_converter.py` | 数据转换到报告 |

### 数据文件
| 文件 | 说明 |
|------|------|
| `.env` | 环境变量（API Token、Cookie等） |
| `data/entries_store.json` | 核心数据存储（14.5MB） |
| `data/brands.json` | 品牌配置列表 |
| `data/latest_result.json` | 最新管道运行结果 |

---

## 9. 待优化方向

1. **模型选择**: 当前DistilBERT可考虑换成ERNIE-ViL（中文特化）
2. **卖点标签扩展**: 当前100+个标签可考虑引入用户反馈自动补充
3. **评论去重**: 当前无重复评论检测机制
4. **增量学习**: 模型权重未随新评论自适应调整
5. **多语言支持**: 当前重点中文，非中文评论的准确率待验证

---

## 总结

Brand Listener已实现**完整的NLP处理链**:
- ✅ 情感分析（5级→3级）
- ✅ 卖点提取（100+标签库）
- ✅ 内容打标（宣发手法、热点等）
- ✅ 关键词匹配（品牌、产品、成分）
- ✅ 评论缓存（避免重复API调用）
- ✅ 行业垂直过滤（口腔护理）

**核心配置**: 环境变量 + JSON文件 + Python类
**关键性能**: 批处理 + 本地缓存 + 成本控制
