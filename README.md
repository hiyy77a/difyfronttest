# 竞赛推荐卡片前端（适配 Dify）

一个纯静态、单文件的竞赛推荐卡片 UI，100% 复刻原设计图。数据全部由 **JSON 驱动**，
方便直接对接 Dify 工作流 / Agent 的输出。

- 文件：`index.html`（HTML + CSS + JS 全部内联，零依赖，双击即可打开）
- 悬停任意**卡片或按钮**都会放大到 `1.05`，移开恢复原样（纯 CSS `transform`）
- 所有按钮和卡片都预留了跳转链接，真实链接给出前**统一跳转到百度**（`https://www.baidu.com`）

---

## 一、数据结构（Dify 需要输出的 JSON）

前端读取 `window.DIFY_DATA`。你的 Dify 工作流只需要输出下面这个结构的 JSON：

```json
{
  "title": "为您推荐 5 项竞赛",
  "description": "基于您的画像（计算机·大三·GPA3.6·数模省一）……综合排序：",
  "filters": [
    { "label": "特级", "count": 1, "level": "special", "url": "" },
    { "label": "一级", "count": 2, "level": "first",   "url": "" },
    { "label": "二级", "count": 2, "level": "second",  "url": "" }
  ],
  "totalLabel": "全部 5 项",
  "totalUrl": "",
  "competitions": [
    {
      "icon": "🏅",
      "name": "中国高校计算机大赛 - 人工智能创意赛",
      "badge": "特级",
      "badgeStyle": "purple",
      "matchLabel": "专业匹配",
      "matchPercent": 95,
      "deadline": "2026-06-30",
      "period": "2-3 个月",
      "team": "1-3 人",
      "difficulty": 4,
      "bonus": "6 分 (国一)",
      "organizer": "全国高等学校计算机教育研究会",
      "detailUrl": "",
      "registerUrl": "",
      "cardUrl": ""
    }
  ]
}
```

### 字段说明

| 字段 | 说明 |
| --- | --- |
| `title` | 顶部紫色标题栏文字 |
| `description` | 标题下方的画像说明文字 |
| `filters[].level` | 决定筛选胶囊的配色：`special`（金）/ `first`（蓝）/ `second`（绿） |
| `filters[].count` | 括号里的数量 |
| `badgeStyle` | 卡片右上角徽章配色：`purple`（蓝紫）/ `gold`（金） |
| `matchPercent` | 专业匹配进度条百分比（0–100） |
| `difficulty` | 难度系数，整数 0–5，渲染为对应数量的实心/空心星 |
| `*Url` / `cardUrl` | 各跳转链接；**留空 `""` 会自动回退到百度** |

> 链接回退逻辑：任何一个 URL 字段为空，前端就用 `FALLBACK_URL`（百度）。
> 拿到真实报名/详情链接后，把对应的 `detailUrl` / `registerUrl` / `cardUrl` 填上即可，无需改前端代码。

---

## 二、如何把 Dify 和前端连接起来

三种方式，按你的部署场景任选其一。

### 方式 A：Dify 直接吐 HTML（最简单，推荐用于聊天气泡内展示）

在 Dify 工作流最后加一个 **代码执行（Code）节点**，把结构化数据拼进 HTML 模板，
让节点直接返回一段带 `<script>window.DIFY_DATA = {...}</script>` 的完整 HTML，
Dify 的 Markdown/HTML 消息就能渲染。

Code 节点（Python）示例：

```python
import json

def main(competitions: list, profile: str) -> dict:
    data = {
        "title": "为您推荐 %d 项竞赛" % len(competitions),
        "description": profile,
        "filters": [
            {"label": "特级", "count": 1, "level": "special"},
            {"label": "一级", "count": 2, "level": "first"},
            {"label": "二级", "count": 2, "level": "second"},
        ],
        "totalLabel": "全部 %d 项" % len(competitions),
        "competitions": competitions,   # 直接透传 LLM 产出的竞赛数组
    }
    # 把 index.html 里 <script> 的 SAMPLE_DATA 换成注入版本即可
    html = TEMPLATE.replace("/*__DIFY_DATA__*/", "window.DIFY_DATA = %s;" % json.dumps(data, ensure_ascii=False))
    return {"html": html}
```

然后把 `index.html` 顶部脚本改成从注入点读取（见下方“方式 A 的接入点”）。

### 方式 B：独立部署前端 + 调 Dify API（推荐用于独立网页）

1. 把 `index.html` 部署到任意静态托管（Nginx / Vercel / OSS 等）。
2. 页面加载时调用 Dify 的 **工作流运行 API**，拿到结构化输出后赋值给 `window.DIFY_DATA` 再渲染：

```html
<script>
  fetch("https://<你的-dify-域名>/v1/workflows/run", {
    method: "POST",
    headers: {
      "Authorization": "Bearer <你的-API-KEY>",
      "Content-Type": "application/json"
    },
    body: JSON.stringify({
      inputs: { profile: "计算机·大三·GPA3.6·数模省一" },
      response_mode: "blocking",
      user: "student-001"
    })
  })
  .then(r => r.json())
  .then(res => {
    // Dify 工作流的结构化输出在 res.data.outputs 里
    window.DIFY_DATA = res.data.outputs.result;   // result 为上文定义的 JSON
    render();   // index.html 里已暴露 render()
  });
</script>
```

> 要点：Dify 工作流的“结束/End 节点”输出一个变量（比如 `result`），
> 内容就是本文档第一节的 JSON。API 返回后从 `res.data.outputs.result` 取出即可。

### 方式 C：手动测试

直接双击 `index.html`，它会用内置的 `SAMPLE_DATA` 渲染示例，方便本地对样式。

---

## 三、方式 A 的接入点（可选）

如果走方式 A（服务端注入），把 `index.html` 中这段：

```js
const SAMPLE_DATA = { ... };
```

前面加一行占位注释，供替换：

```js
/*__DIFY_DATA__*/            // Dify 会把 `window.DIFY_DATA = {...};` 注入到这里
const SAMPLE_DATA = { ... };
```

由于代码里已有 `const data = window.DIFY_DATA || SAMPLE_DATA;`，
只要注入了 `window.DIFY_DATA`，就会自动优先使用真实数据。

---

## 四、常见改动

- **换真实链接**：填 JSON 里的 `detailUrl` / `registerUrl` / `cardUrl`，无需动前端。
- **加更多竞赛**：往 `competitions` 数组里加对象即可，卡片自动生成。
- **改配色**：见 `index.html` `<style>` 顶部的 `:root` CSS 变量。
