# TrendRadar - 全球金融市场分析

定时推送全球金融市场热点新闻，AI 智能筛选 + 全天汇总报告。

## 推送时间（北京时间）

| 时间 | 内容 |
|------|------|
| **08:40** | 早间汇总（A股盘前 + 美股隔夜总结） |
| **18:45** | 晚间速览（美股盘前准备） |
| **20:40** | 🌟 全天总结（AI 深度汇总） |

## 配置

- **调度模式**: Custom（3 次/天）
- **AI 引擎**: Kimi K2.6 (Moonshot)
- **筛选方式**: AI 智能分类
- **数据源**: 华尔街见闻、财联社、Yahoo Finance、CNBC、MarketWatch

## 本地运行

### 环境要求

- Python 3.10+
- 依赖安装：`uv sync` 或 `pip install -r requirements.txt`

### 配置 API 密钥

**安全提醒：不要在 `config/config.yaml` 中填写 API 密钥，该文件会被提交到 Git。**

1. 复制 `.env.example` 为 `.env`
2. 在 `.env` 中填入你的 API 密钥：

```
AI_API_KEY=你的_API_密钥
```

`run.bat` 会自动加载 `.env` 文件。也可以直接设置系统环境变量。

### 命令行

```bash
# 正常运行
python -m trendradar

# 开启 DEBUG 日志
python -m trendradar --debug

# 环境体检
python -m trendradar --doctor

# 查看调度状态
python -m trendradar --show-schedule

# 测试通知渠道
python -m trendradar --test-notification
```

## 部署

GitHub Actions 定时触发，GitHub Pages 托管报告页面。敏感信息通过 GitHub Secrets（环境变量）配置。

---

*Last updated: 2026-06-10*
