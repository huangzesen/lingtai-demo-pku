# LingTai PKU Demo Recipe

LingTai 北大极客中心 30–60 分钟演示技艺的 recipe bundle——包含完整的演出弧（见·知·触）、化身爆发编排、幻灯素材、风险预案与术语速查。

## 应用方式

```bash
git clone https://github.com/huangzesen/lingtai-demo-pku.git
cd lingtai-demo-pku
lingtai-tui
```

首次启动时，wizard 自动检测 `.recipe/` 目录并提示应用此 recipe。应用后，orchestrator 将以叙事钩子「我有太多 token 烧不完怎么办？」开场，并按三幕结构运行演示。

## Bundle 内容

- `.recipe/` — recipe manifest、开场词（greet）、演出指引（comment）
- `lingtai-demo-pku/` — library sibling，含完整 SKILL.md 及四份原始草稿
  - `drafts/arc.md` — 演出弧设计
  - `drafts/vs-pi-mono.md` — 与 pi-mono 的技术对比
  - `drafts/slide-feed.md` — 11 张幻灯素材
  - `drafts/avatar-burst.md` — 化身爆发编排方案
