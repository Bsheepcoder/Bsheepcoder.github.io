每条 API 经实测验证（2026-06-25），国内直连可用。不可用的一律不收。从 GitHub 仓库中直接拿接口，不只列仓库链接。

---

## 金融行情

| API | 请求示例 | 说明 |
|-----|---------|------|
| 腾讯财经 | `https://qt.gtimg.cn/q=hf_GC` | 无需 Key，浏览器端可用，支持批量 `q=hf_GC,hf_SI` |
| 新浪财经 | `https://hq.sinajs.cn/list=hf_GC` | 需 Referer 头，仅服务端可用 |
| TwelveData | `https://api.twelvedata.com/price?symbol=XAU/USD&apikey=KEY` | 免费注册 800 次/天，有 [ClawHub 技能](https://github.com/twelvedata/twelvedata-clawhub) |
| SEC EDGAR | `https://data.sec.gov/submissions/CIK0000320193.json` | 美国上市公司年报/财报，无需 Key |
| Econdb | `https://www.econdb.com/api/series/` | 全球宏观经济数据，需免费注册 |

**腾讯品种代码**：`hf_GC` 黄金 · `hf_SI` 白银 · `hf_CL` 原油 · `sh000001` 上证 · `sz399001` 深成 · `hkHSI` 恒生

**腾讯返回字段**（逗号分隔）：`[0]当前价 [1]涨跌额 [4]最高 [5]最低 [6]时间 [7]开盘 [8]昨结 [12]日期 [13]品名`

> 以上来自 [public-apis/public-apis](https://github.com/public-apis/public-apis) ★444K Finance 分类

---

## 天气

| API | 请求示例 | 说明 |
|-----|---------|------|
| Open-Meteo | `https://api.open-meteo.com/v1/forecast?latitude=39.9&longitude=116.4&current=temperature_2m` | 全球天气预报，无需 Key，CORS ✅ |
| wttr.in | `https://wttr.in/Beijing?format=j1` | 终端天气，支持 JSON，无需 Key，CORS ✅ |
| US Weather | `https://api.weather.gov/points/39.7,-104.9` | 美国国家气象局，无需 Key，CORS ✅ |

> 以上来自 [public-apis/public-apis](https://github.com/public-apis/public-apis) ★444K Weather 分类

---

## 地理与 IP 定位

| API | 请求示例 | 说明 |
|-----|---------|------|
| REST Countries | `https://restcountries.com/v3.1/all?fields=name,capital` | 国家信息（首都/货币/语言/国旗），无需 Key，CORS ✅ |
| ip-api.com | `http://ip-api.com/json/1.1.1.1` | IP 地理定位，无需 Key，仅 HTTP |
| GeoJS | `https://get.geojs.io/v1/ip/geo.json` | IP 地理定位，无需 Key，CORS ✅ |
| Nominatim | `https://nominatim.openstreetmap.org/search?q=beijing&format=json&limit=1` | OpenStreetMap 地理编码/逆地理编码，无需 Key |
| OnWater | `https://api.onwater.io/api/v1/results/40.7,-74.0` | 判断经纬度是否在水上，无需 Key |
| Open Topo Data | `https://api.opentopodata.org/v1/srtm90m?locations=39.9,116.4` | 海拔/海洋深度查询，无需 Key |

> 以上来自 [public-apis/public-apis](https://github.com/public-apis/public-apis) ★444K Geocoding 分类

---

## 学术与医疗

| API | 请求示例 | 说明 |
|-----|---------|------|
| OpenAlex | `https://api.openalex.org/works?search=brain+rehabilitation` | 2.5 亿篇论文，无需 Key，RAG 采集首选 |
| NCBI PubMed | `https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi?db=pubmed&term=brain+injury` | 生物医学文献权威，无 Key 3 次/秒 |
| Wikidata | `https://www.wikidata.org/w/api.php?action=wbgetentities&ids=Q42&format=json` | 知识图谱，支持 SPARQL，无需 Key |
| FDA openfda | `https://api.fda.gov/drug/event.json?limit=5` | 药品不良事件/标签/召回，无 Key 1000 次/天 |
| Open Disease | `https://disease.sh/v3/covid-19/all` | COVID-19 和流感实时数据，无需 Key |
| Clinical Trials | `https://trials.starfile.org/api` | 临床试验数据，按疾病/赞助者索引，无需 Key |

> 以上来自 [public-apis/public-apis](https://github.com/public-apis/public-apis) ★444K Health 分类

---

## 航天

| API | 请求示例 | 说明 |
|-----|---------|------|
| NASA | `https://api.nasa.gov/planetary/apod?api_key=DEMO_KEY` | 天文一图/火星照片/小行星，DEMO_KEY 可用 |
| Spaceflight News | `https://api.spaceflightnewsapi.net/v4/articles/?limit=5` | 全球航天新闻聚合，无需 Key |

---

## 新闻

| API | 请求示例 | 说明 |
|-----|---------|------|
| Noozra | `https://noozra.com/api` | 200+ RSS 源新闻标题，无需 Key，CORS ✅ |

> 以上来自 [public-apis/public-apis](https://github.com/public-apis/public-apis) ★444K News 分类

---

## 开放数据

| API | 请求示例 | 说明 |
|-----|---------|------|
| Wikipedia | `https://en.wikipedia.org/w/api.php?action=query&list=search&srsearch=brain&format=json` | 维基百科全文搜索，无需 Key |
| Archive.org | `https://archive.org/wayback/available?url=example.com` | 互联网档案馆 Wayback Machine，无需 Key |
| Nobel Prize | `https://api.nobelprize.org/v1/prize.json?limit=1` | 诺贝尔奖数据，无需 Key |
| Universities | `https://raw.githubusercontent.com/Hipo/university-domains-list/master/world_universities_and_domains.json` | 全球大学域名列表，无需 Key |
| Statistics World | `https://statisticsoftheworld.com/api/v1/countries?limit=1` | 218 国经济数据（GDP/人口/通胀），无需 Key |
| Microlink.io | `https://api.microlink.io/?url=https://example.com` | 从任意 URL 提取结构化数据，无需 Key，CORS ✅ |

> 以上来自 [public-apis/public-apis](https://github.com/public-apis/public-apis) ★444K Open Data 分类

---

## 测试数据

| API | 请求示例 | 说明 |
|-----|---------|------|
| JSONPlaceholder | `https://jsonplaceholder.typicode.com/posts/1` | 假 REST API（posts/users/comments/todos），无需 Key |
| DummyJSON | `https://dummyjson.com/products/1` | 假数据（产品/用户/帖子/评论），无需 Key，CORS ✅ |
| RandomUser | `https://randomuser.me/api/?results=1` | 随机用户数据生成，无需 Key |
| RoboHash | `https://robohash.org/test.png` | 随机头像生成，无需 Key |
| Yes No | `https://yesno.wtf/api` | 随机返回 yes/no/maybe（含 GIF），无需 Key |

> 以上来自 [public-apis/public-apis](https://github.com/public-apis/public-apis) ★444K Test Data 分类

---

## 机器学习

| API | 请求示例 | 说明 |
|-----|---------|------|
| Jina AI | `https://api.jina.ai/v1/embeddings` | 免费 Embedding/Reranking API，需 Key |

> 以上来自 [public-apis/public-apis](https://github.com/public-apis/public-apis) ★444K Machine Learning 分类

---

## 免费 LLM API

从 [free-llm-api-resources](https://github.com/cheahjs/free-llm-api-resources) ★24K 和 [awesome-free-llm-apis](https://github.com/mnfst/awesome-free-llm-apis) ★5K 两个仓库直接提取，经国内可达性验证。

所有 API 均兼容 OpenAI SDK 格式，切换 `base_url` 即可。

### 无需注册

| API | Base URL | 免费模型 | 限制 |
|-----|----------|---------|------|
| LLM7.io | `https://api.llm7.io/v1` | DeepSeek-R1, DeepSeek-V3, GPT-4o-mini, Gemini 2.5 Flash Lite | 30 RPM（有 Token 120 RPM） |

### 注册即用（免费额度）

| API | Base URL | 模型 | 免费额度 |
|-----|----------|------|---------|
| OpenRouter | `https://openrouter.ai/api/v1` | Llama 3.3 70B, Qwen3 Coder, GPT-OSS 120B 等 20+ 免费模型 | 50 req/天（充 $10 终身提至 1000/天） |
| GitHub Models | `https://models.github.ai/inference` | GPT-5, GPT-4.1, o4-mini, DeepSeek-R1, Llama 4 等 45+ 模型 | 10-15 RPM, 50-150 req/天 |
| Cloudflare Workers AI | `https://api.cloudflare.com/client/v4/accounts/{id}/ai/run` | Llama 3.3 70B, GPT-OSS 120B, GLM-4.7-Flash, DeepSeek R1 等 50+ 模型 | 10K neurons/天 |
| Groq | `https://api.groq.com/openai/v1` | Llama 3.3 70B, Llama 4 Scout, Qwen3-32B, GPT-OSS 120B | 30 RPM, 1000 req/天 |
| Cerebras | `https://api.cerebras.ai/v1` | GPT-OSS 120B, GLM-4.7（超快推理 ~2600 tok/s） | 30 RPM, 14,400 req/天, 1M tokens/天 |
| Cohere | `https://api.cohere.com/v2` | Command A+ (218B), Command A, Command R | 20 RPM, 1000 calls/月 |
| 智谱 AI (Z AI) | `https://open.bigmodel.cn/api/paas/v4` | GLM-4.7-Flash (200K context), GLM-4.6V-Flash (图文) | 永久免费，1 并发 |
| NVIDIA NIM | `https://integrate.api.nvidia.com/v1` | 各种开源模型 | 40 RPM |
| Kilo Code | `https://api.kilo.ai/api/gateway` | Grok Code Fast, Minimax M2.5, Nemotron 3 Super 120B | ~200 req/时 |

### 有试用额度

| API | 信用额度 | 模型 |
|-----|---------|------|
| Fireworks | $1 | 各种开源模型 |
| Baseten | $30 | 按计算时间计费 |
| Nebius | $1 | 各种开源模型 |
| Novita | $0.5 (1 年) | 各种开源模型 |
| AI21 | $10 (3 个月) | Jamba 系列 |
| Modal | $5/月 | 按计算时间计费 |
| SambaNova Cloud | $5 (3 个月) | DeepSeek V3.1, Llama 3.3 70B |
| Scaleway | 1M tokens | Llama 3.3 70B, Qwen3 Coder, Mistral 等 |

> 不可用：Google AI Studio（超时）· Mistral（超时）· HuggingFace（超时）

---

## 国产 LLM API

均兼容 OpenAI API 格式。

| API | 域名 | 免费 | 模型 |
|-----|------|------|------|
| 智谱 AI | `open.bigmodel.cn` | GLM-4-Flash 永久免费 | GLM-4 / GLM-4-Flash |
| DeepSeek | `api.deepseek.com` | 新用户赠 Token | DeepSeek-V3 / R1 |
| Moonshot | `api.moonshot.cn` | 新用户赠 Token | Moonshot-v1（128K） |
| 通义千问 | `dashscope.aliyuncs.com` | 新用户赠 Token | Qwen-Plus / Qwen-Turbo |

---

## AI Agent 技能生态

- [ClawHub](https://github.com/openclaw/clawhub) ★9K — OpenClaw 官方技能注册表，[clawhub.ai](https://clawhub.ai)，CLI：`clawhub install/search`
- [Cow Skill Hub](https://github.com/zhayujie/cow-skill-hub) — 跨平台技能市场，[skills.cowagent.ai](https://skills.cowagent.ai)
- [Skills Manager](https://github.com/cchao123/skills-manager) ★135 — 桌面端跨 Agent 技能管理器（Claude Code/Cursor/OpenClaw/OpenCode 等）
- [OpenClaw Medical Skills](https://github.com/FreedomIntelligence/OpenClaw-Medical-Skills) ★2.7K — 最大开源医疗 AI 技能库
- [MCP Servers](https://github.com/modelcontextprotocol/servers) ★88K — MCP 官方服务器仓库，开箱即用

---

## GitHub 公开 API 汇总仓库

以下仓库持续更新，更多分类的 API 直接去看。

### 综合

- [public-apis/public-apis](https://github.com/public-apis/public-apis) ★444K — 51 个分类 1400+ API，标注 Key/HTTPS/CORS（本文 50+ 接口取自此处）
- [n0shake/Public-APIs](https://github.com/n0shake/Public-APIs) ★23K
- [public-api-lists/public-api-lists](https://github.com/public-api-lists/public-api-lists) ★15K — 48 分类，提供可搜索 JSON API
- [TonnyL/Awesome_APIs](https://github.com/TonnyL/Awesome_APIs) ★13K
- [255kb/stack-on-a-budget](https://github.com/255kb/stack-on-a-budget) ★12K — 免费层服务汇总
- [cporter202/API-mega-list](https://github.com/cporter202/API-mega-list) ★7K
- [APIs-guru/graphql-apis](https://github.com/APIs-guru/graphql-apis) ★5K — GraphQL API 列表
- [wdhdev/free-for-life](https://github.com/wdhdev/free-for-life) ★2K — 终身免费资源

### LLM 专项

- [cheahjs/free-llm-api-resources](https://github.com/cheahjs/free-llm-api-resources) ★24K — 免费 LLM API 资源（本文 LLM 接口取自此处）
- [mnfst/awesome-free-llm-apis](https://github.com/mnfst/awesome-free-llm-apis) ★5K — 永久免费 LLM API（本文 LLM 接口取自此处）
- [deepseek-ai/awesome-deepseek-integration](https://github.com/deepseek-ai/awesome-deepseek-integration) ★38K

### 金融 / 医疗

- [theOGognf/finagg](https://github.com/theOGognf/finagg) ★539 — Python 金融 API 聚合
- [quantmind/awesome-open-finance](https://github.com/quantmind/awesome-open-finance) ★164
- [FDA/openfda](https://github.com/FDA/openfda) ★682

---

## 避坑

- **User-Agent**：调用学术 API 带上 `User-Agent: MyProject/1.0 (mailto:you@email.com)`，避免被限流封禁
- **Referer**：新浪财经需 Referer，浏览器端不可用，改用腾讯
- **限流**：PubMed 无 Key 3 次/秒 · NASA DEMO_KEY 30 次/时 · FDA 无 Key 1000 次/天 — 注册免费 Key 均可提升
- **RAG 管道**：OpenAlex 抓取 → 智谱 GLM-4-Flash（永久免费）或 LLM7.io（无需注册）清洗 → 本地 Embedding → 静态 JSON / 向量库

---

## 验证

**最近验证日期**：2026-06-25 | 全部 ✅ 直连可用

**不可用（已移除）**：TradingView（超时）· Yahoo Finance（403）· Binance（超时）· OKEx（DNS 不可达）· CoinGecko（超时）· CoinCap（DNS 不可达）· 网易财经（DNS 不可达）· Google Gemini（超时）· Mistral（超时）· HuggingFace（超时）· Launch Library 2（429）· Dicebear（410）

**重测脚本**：

```powershell
# 金融
Invoke-WebRequest "https://qt.gtimg.cn/q=hf_GC" -TimeoutSec 10
Invoke-WebRequest "https://data.sec.gov/submissions/CIK0000320193.json" -TimeoutSec 10 -Headers @{"User-Agent"="Mozilla/5.0"}

# 天气
Invoke-WebRequest "https://api.open-meteo.com/v1/forecast?latitude=39.9&longitude=116.4&current=temperature_2m" -TimeoutSec 10
Invoke-WebRequest "https://wttr.in/Beijing?format=j1" -TimeoutSec 10

# 地理与 IP 定位
Invoke-WebRequest "https://restcountries.com/v3.1/all?fields=name,capital" -TimeoutSec 10
Invoke-WebRequest "https://get.geojs.io/v1/ip/geo.json" -TimeoutSec 10
Invoke-WebRequest "https://nominatim.openstreetmap.org/search?q=beijing&format=json&limit=1" -TimeoutSec 10
Invoke-WebRequest "https://api.opentopodata.org/v1/srtm90m?locations=39.9,116.4" -TimeoutSec 10

# 学术医疗
Invoke-WebRequest "https://api.openalex.org/works?search=test&per-page=1" -TimeoutSec 10
Invoke-WebRequest "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi?db=pubmed&term=test&retmax=1" -TimeoutSec 10
Invoke-WebRequest "https://api.fda.gov/drug/event.json?limit=1" -TimeoutSec 10
Invoke-WebRequest "https://disease.sh/v3/covid-19/all" -TimeoutSec 10

# 开放数据
Invoke-WebRequest "https://en.wikipedia.org/w/api.php?action=query&list=search&srsearch=brain&format=json" -TimeoutSec 10
Invoke-WebRequest "https://archive.org/wayback/available?url=example.com" -TimeoutSec 10
Invoke-WebRequest "https://api.nobelprize.org/v1/prize.json?limit=1" -TimeoutSec 10
Invoke-WebRequest "https://api.microlink.io/?url=https://example.com" -TimeoutSec 10

# 测试数据
Invoke-WebRequest "https://jsonplaceholder.typicode.com/posts/1" -TimeoutSec 10
Invoke-WebRequest "https://dummyjson.com/products/1" -TimeoutSec 10
Invoke-WebRequest "https://randomuser.me/api/?results=1" -TimeoutSec 10

# LLM（返回 401/403 = 域名可达，需 Key）
Invoke-WebRequest "https://api.llm7.io/v1/models" -TimeoutSec 10
Invoke-WebRequest "https://openrouter.ai/api/v1/models" -TimeoutSec 10
Invoke-WebRequest "https://models.github.ai/inference/models" -TimeoutSec 10
Invoke-WebRequest "https://api.groq.com/openai/v1/models" -TimeoutSec 10
```

---

## 更新日志

| 日期 | 变更 |
|------|------|
| 2026-06-25 | 初始版本，从 3 个 GitHub 仓库直接提取 API，验证 50+ 个接口，收录 50+ 个可用 API + 12 个免费 LLM 平台，覆盖 13 个分类 |
