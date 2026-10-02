# 历史人物互动测试合集

一套基于「随机路径 + 口令加密」的付费互动测试页面，托管在 GitHub Pages 上。每个测试都是一个独立的小游戏，用户付款后获得专属口令，输入口令即可进入作答并生成个性化结果。

> 本仓库为创作者自用，口令由创作者在付款后私下发给用户，**不在仓库中公开**。

## 玩法

1. 创作者把某款测试的链接发给已付款用户
2. 用户打开链接，输入创作者提供的 6 位口令
3. 口令正确 → 进入答题；口令错误 → 被拦截
4. 答完所有题目，生成一份专属的图文结果报告，可截图分享

## 安全机制

- **随机不可猜路径**：每个测试放在一个 10 位随机字符串目录下（如 `e6t4pihgks`），无法从一个测试推导出另一个测试的链接
- **题库 AES-GCM 加密**：题库明文（`content.source.json`）不会出现在网页里，只有加密后的 `data.enc.json` 被加载
- **口令 PBKDF2 派生密钥**：每个口令经过 20 万次 PBKDF2 迭代派生密钥，再包裹内容密钥；想作废某个口令，重新打包时不带它即可
- **本机凭证隔离**：通过后的凭证按目录存在 `localStorage`，不同测试互不干扰；想清除可访问 `?forget=1`

## 现有测试（13 款）

| # | 主题 | 链接 |
|---|---|---|
| 1 | 你的天赋变现报告 | https://gavinwang668.github.io/talent-report/jx9guxq8m9/ |
| 2 | 历史人物版 MBTI 对照表 | https://gavinwang668.github.io/talent-report/f6rt3evt5p/ |
| 3 | 古人朋友圈 | https://gavinwang668.github.io/talent-report/b07mwt99ep/ |
| 4 | 你的死法生成器 | https://gavinwang668.github.io/talent-report/n22x65p6vv/ |
| 5 | 历史人物吵架模拟器 | https://gavinwang668.github.io/talent-report/m46m1gs243/ |
| 6 | 你的灵魂成分表 | https://gavinwang668.github.io/talent-report/wjiwkl2evc/ |
| 7 | 古代官职任命书 | https://gavinwang668.github.io/talent-report/w74vkjmuaj/ |
| 8 | 你的本命朝代 | https://gavinwang668.github.io/talent-report/qtuspa7brl/ |
| 9 | 古代恋爱人格测试 | https://gavinwang668.github.io/talent-report/baum12il61/ |
| 10 | 你的江湖门派 | https://gavinwang668.github.io/talent-report/e6t4pihgks/ |
| 11 | 你的古代谥号 | https://gavinwang668.github.io/talent-report/2uxznqn6hc/ |
| 12 | 你的神兽人格 | https://gavinwang668.github.io/talent-report/df23qgevkw/ |
| 13 | 你的古代名讳 | https://gavinwang668.github.io/talent-report/uxbipnurmc/ |

> 口令由创作者在用户付款后私信发送，不在此处公开。

## 项目结构

```
talent-report/
├── jx9guxq8m9/            # 每个测试一个随机目录
│   ├── index.html         # 页面 + 答题引擎 + 主题
│   └── data.enc.json      # 加密后的题库（线上加载）
├── ...
├── pack.local.js          # 本地加密打包工具（不入库）
└── .gitignore             # 忽略 content.source.json 和 pack.local.js
```

每个测试目录下实际有 3 个文件，其中 `content.source.json`（明文题库）被 `.gitignore` 忽略，不会推送到仓库：

- `index.html` — 页面，包含通用答题引擎和该测试的主题配色/文案
- `content.source.json` — 明文题库，仅本地存在（不入库）
- `data.enc.json` — 加密题库，由 `pack.local.js` 生成，随 `index.html` 一起部署

## 新增一款测试

### 1. 生成随机路径和口令

```bash
node -e '
const crypto=require("crypto");
const A="abcdefghjkmnpqrstuvwxyz23456789";
const genPath=()=>Array.from({length:10},()=>(crypto.randomInt(36)).toString(36)).join("");
const genCode=()=>Array.from({length:6},()=>A[crypto.randomInt(A.length)]).join("");
console.log("path:", genPath());
console.log("code1:", genCode(), "code2:", genCode());
'
```

### 2. 复制模板并定制

```bash
mkdir <随机路径>
cp w74vkjmuaj/index.html <随机路径>/index.html
# 修改 index.html 中的 THEME 对象（标题、配色、文案）
```

### 3. 编写题库 `content.source.json`

题库结构：

```json
{
  "QUESTIONS": [
    {
      "q": "题目文本",
      "opts": [
        { "t": "选项文本", "p": { "类型key": 权重 } }
      ]
    }
  ],
  "TYPES": {
    "类型key": {
      "emoji": "🦁",
      "name": "结果标题",
      "tag": "副标题",
      "blocks": [
        { "k": "para", "label": "段落标题", "text": "段落正文" },
        { "k": "list", "label": "列表标题", "items": [{"t":"项","d":"描述","chips":["标签"]}] },
        { "k": "quote", "label": "引言标题", "text": "引语文本" },
        { "k": "action", "label": "行动框标题", "text": "行动框正文" },
        { "k": "table", "label": "表格标题", "head":["列1","列2"], "rows":[["a","b"]] }
      ]
    }
  }
}
```

**权重设计要点**（保证结果无平局且分布均衡）：

- 每个选项的权重用 `8192 + 2^题序号`（题序号从 0 开始）
- 这样每道题的权重都包含一个唯一的二进制位，任意答题路径的得分都不会出现平局
- 尽量让每个类型出现在相同数量的题目中，保证各结果出现概率均衡

支持 `AXES` 轴模式（如 MBTI，4 个轴各 3 题，组合出 16 型）和 `COMPONENTS` 成分模式（如灵魂成分表，算各成分占比）。

### 4. 加密打包

```bash
node pack.local.js <随机路径> <口令1> <口令2>
# 生成 data.enc.json
```

### 5. 本地验证

```bash
cd /workspace/_tests
node validate.js                       # 结构 + 计分校验
CHROME_BIN=<chrome路径> node check-codes.js   # 口令有效性
CHROME_BIN=<chrome路径> node run.js 8 12       # 完整流程（参数为起始/结束下标）
```

### 6. 部署

```bash
# 提交 main
git add <随机路径>
git commit -m "feat: 新增测试 xxx"
git push origin main

# 同步到 gh-pages
git fetch origin gh-pages
git worktree add --detach /tmp/ghp origin/gh-pages
mkdir -p /tmp/ghp/<随机路径>
cp <随机路径>/index.html <随机路径>/data.enc.json /tmp/ghp/<随机路径>/
cd /tmp/ghp && git add -A && git commit -m "deploy: 上线测试 xxx"
git push origin HEAD:gh-pages
git worktree remove /tmp/ghp --force
```

## 技术栈

- 纯前端：HTML + CSS + 原生 JS，无构建步骤
- 加密：Web Crypto API（AES-GCM + PBKDF2）
- 部署：GitHub Pages（`gh-pages` 分支）
- 测试：Playwright + 自定义校验脚本

## 注意事项

- `content.source.json` 和 `pack.local.js` 已被 `.gitignore` 忽略，不会推送到仓库
- 明文题库只存在于本地，线上只加载加密后的 `data.enc.json`
- 口令作废只需重新打包（`pack.local.js` 不带该口令）并重新部署
- 每个测试的口令相互独立，更换一个测试的口令不影响其他测试
