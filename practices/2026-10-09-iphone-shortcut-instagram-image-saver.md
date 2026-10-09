# 实践沉淀：不开发 App，用 iPhone 快捷指令批量下载 Instagram 帖子图片

> **完成时间**：2026-10-09
> **参与方**：我（需求方 + 手机端实测）、ChatGPT（方案设计，前期）、Claude Code on Mac（代码开发与远程调试，后期）
> **最终成果**：iPhone 上 Safari 打开 Instagram 公开帖子 → 分享 → 运行快捷指令 → 轮播图全部图片自动存入「照片」App，全程不开发原生 App、不装 Xcode、不装第三方下载工具
> **性质**：一次完整的「人机协作远程调试」实战——AI 在 Mac 上写代码和分析证据，人在 iPhone 上操作和反馈，两端接力闭环

---

## 一、需求与约束

**需求**：在 iPhone 上给一个网页地址（主要是 Instagram 公开帖子），把帖子里的所有图片（含轮播图）批量下载保存到「照片」App。

**约束**：

- 不开发原生 iOS App，不用 Xcode
- 纯 iPhone 端运行（后期放宽：Mac 可以参与开发调试，但最终使用流程仍在手机上）
- 不绕过登录/反爬，只处理公开可访问的内容

---

## 二、第一阶段：与 ChatGPT 的方案设计（13 轮对话）

原始对话存档：`chatgpt.com/share/6ac8a15a-d448-83e8-9e0d-5297980b5b63`（标题「快捷指令下载图片」，完整重建文本 6 万字符，本机留存于 `/tmp/conversation_clean.md`）

### 2.1 交互流程时间线

| # | 我的输入 | ChatGPT 的输出 | 关键转折 |
|---|---|---|---|
| 1 | 想在 iPhone 上批量下载网页图片，不想开发 App，怎么最快实现？ | 给出 3 方案：①快捷指令+在线提取服务 ②iPhone 本地脚本（Scriptable）③Mac 提取+iPhone 接收；推荐「快捷指令+Scriptable」 | 明确了技术路线的大方向 |
| 2 | 用表单选择了：纯 iPhone、保存到照片、要具体步骤和代码 | 给出**通用版快捷指令**完整配置：`在网页上运行 JavaScript`（提取 img/srcset/懒加载/og:image）→ `重复每一项` → `获取 URL 内容` → `存储到照片相簿` | 核心架构在此定型，后面所有迭代都围绕这套动作链 |
| 3 | 选择了平台：Instagram、范围：单个帖子或图集 | 给出 **Instagram 专版**：只接受 `cdninstagram.com`/`fbcdn.net` CDN 域名、限定 `article`/`main` 区域、过滤 <250px 小图、按面积排序；同时坦承限制：轮播图未必全在 DOM、CDN 链接会过期 | 第一次明确「网页 DOM 能拿到什么」的边界 |
| 4 | Safari 打开 Instagram 链接总是自动跳转到 App 怎么办 | 解释 Universal Links 机制；建议手动在 Safari 地址栏输入网址进入，不走 App 内链接 | 排除环境干扰 |
| 5 | 快捷指令运行了但没有图片 | （此处 ChatGPT 给出的方向较泛） | **问题悬而未决，成为后续 Claude Code 阶段的主战场** |
| 6 | 通过开发微信小程序可以实现吗？ | 可以但有限制：小程序网络请求受域名白名单约束，Instagram CDN 不在白名单内，需要云函数中转 | 探索支线，发现微信生态限制太多 |
| 7 | 云开发云函数是什么？ | 通俗解释：别人帮你准备好的云端电脑 | 支线补课 |
| 8 | 还是回到第一套方案。我有 Mac，装了 Xcode 和 Claude Code，能让 Mac 里的 Claude Code 连接手机自己开发自己调试吗？登录输密码这些我可以手机端接入 | 给出**人机分工模式**：Claude Code 写代码/改文件/跑测试，Mac 执行，iPhone 负责打开网页、触发操作、保存图片；明确 Claude Code 不能自动控制手机镜像；给出三阶段开发计划和可直接粘贴给 Claude Code 的需求 Prompt | **本次实践的核心协作模式在此确立** |
| 9 | 把错误交回 Claude Code 修复，错误怎么得到？我试了之前的脚本拿不到反馈 | 给出**诊断版脚本**：不下载，输出 JSON 报告（页面标题、readyState、图片元素数、去重 URL 数、CDN 计数、样例），后接「快速查看」显示，把报告复制回给 Claude Code 分析；提出 A 提取/B 下载/C 保存 三层独立测试法 | **建立了手机→Mac 的错误反馈通道，调试闭环成型** |
| 10 | 发快捷指令截图求确认 | 确认前两步正确，但看不到后续动作，建议加「快速查看」验证提取数量 | 会话在此结束，接力棒交给 Claude Code |

### 2.2 ChatGPT 阶段小结

**做对了什么**：

- 方案选型果断：没有陷入「要不要学 Swift」的泥潭，直接给出快捷指令 + Safari JavaScript 这条最轻路径
- 诚实标注限制：反复强调懒加载、登录墙、CDN 签名过期、轮播图 DOM 问题，不吹「万能下载」
- 诊断思维：第 9 轮给出的「JSON 诊断报告 + 三层测试法（A 提取/B 下载/C 保存）」是整个调试期的骨架

**没做到什么**：

- 通用版代码有隐患（srcset 的 `&amp;` 问题埋雷，见后文）
- 「运行了没图片」的问题没有实际解决——它没有 Mac 端的验证手段，只能给方向不能给证据
- 用户拿到的反馈始终隔着一层：无法把手机上的真实错误精确传回去

---

## 三、第二阶段：与 Claude Code 的远程调试实战（9 轮交互）

### 3.0 前置：把 ChatGPT 分享页内容完整取回来

**我的请求**：「获取这个分享页面的内容」。

**Claude Code 的做法**（这段本身就是一个值得沉淀的技巧）：

1. `curl` 拉取页面 → 1MB HTML，但 ChatGPT 分享页是纯前端渲染，正文不在 HTML 里
2. 定位到 `window.__reactRouterContext.streamController.enqueue("...")` —— 对话数据以 **turbo-stream 序列化格式**嵌在 JS 里
3. 用 `json.raw_decode` 解出 JS 字符串字面量（第一版脚本手工解转义踩了坑：把 `\"` 原样保留导致 JSON 解析失败，改用 `raw_decode` 从原始文本直接解码才通过）
4. 手写 turbo-stream 解码器：平铺数组中，对象形如 `{"_266": 267}` 表示「键 = payload[266]，值 = payload[267]」，负数是 null/undefined 等特殊值；递归 resolve 出完整对象树
5. 沿 `loaderData → serverResponse → data → mapping` 找到对话树，按 children 关系重建消息顺序，去掉流式重复版本，得到 39 条有效消息（13 user / 22 assistant / 4 tool）

**沉淀**：前端渲染页面的数据往往藏在 `<script>` 的序列化流里（`__NEXT_DATA__`、`__reactRouterContext`、`__remixContext` 等），curl + 解码器是通用取回手段，不必依赖浏览器。

### 3.1 交互全流程复盘

#### 轮 1：提交诊断报告

**我**：把手机上运行诊断脚本得到的 JSON 报告粘贴过来（页面正确、readyState complete、4 个图片元素、4 个 Instagram CDN URL、2 张 928×1232 大图 + 2 张 150×150 小图）。

**Claude Code 的分析**：

- 结论：**测试 A（提取）通过**——页面正确、脚本能拿到 CDN 真图，「没图片」的问题在下载/保存环节
- 150×150 是头像/图标，属应过滤噪音
- 指出两个隐患：①轮播图只有已加载的在 DOM 里；②诊断报告里的样例 URL 被截掉了 `?` 后参数，**实际下载必须用完整 URL**（scontent 链接是带签名的）
- 给出正式提取版 JS（过滤 <250px、完整保留 href）+ 动作链配置（重复→获取 URL 内容→快速查看→存储照片）

**沉淀**：拿到实测数据后先做「证据判读」再动手——报告里每一行都指向一个结论，不要跳过。

#### 轮 2：Quick Look 弹出了 URL 而不是图片

**我**：把 Quick Look 显示的完整 URL 粘贴过来。

**Claude Code 的动作**：

- 先在 Mac 上 `curl -I` 该 URL → **HTTP 200, image/jpeg, 254KB** → 链接本身完全有效，且不需要 Cookie
- 由此判定：问题 100% 在快捷指令的**内部接线**——Quick Look 拿到的是「重复项目」（URL 字符串）而不是「获取 URL 内容」的结果
- 给出动作顺序图 + 逐个变量的检查清单

**沉淀**：**能在 Mac 端验证的事实绝不靠猜**。一条 curl 就把「URL 坏了」和「快捷指令接线错了」两种假设区分开了。

#### 轮 3~4：截图确认快捷指令现状

**我**：发了 2 张快捷指令编辑器截图（后又重发 2 张同内容）。

**Claude Code 的动作**：

- 会话内 Read 工具读不了图片（Unsupported Image），转格式、换路径都无效
- **改用 macOS 自带 Vision 框架写 Swift OCR 脚本**（`/tmp/ocr.swift`，`VNRecognizeTextRequest`，语言 `zh-Hans`），成功读出截图内容
- OCR 结果显示：快捷指令只有「接收输入 → 运行 JavaScript → 快速查看(JavaScript结果)」三个动作——**「重复每一项」「获取 URL 内容」「存储到照片相簿」从来没加进去**！这就是「运行了没图片」的真正原因：下载和保存动作压根不存在
- 给出逐动作的添加指引（搜索什么关键词、变量选什么）

**沉淀**：这条是本次最重要的协作教训之一——**「我以为用户配好了」和「用户实际配了什么」之间隔着巨大的鸿沟，一张截图胜过十句描述**。而当工具链读不了图时，Swift + Vision 的十几行代码就能补上眼睛。

#### 轮 5：配置完成，请求确认

**我**：发 2 张新截图求确认。

**Claude Code**：OCR 读出结构——`重复每一项 → 获取 URL 内容(重复项目) → 快速查看(URL内容) → 存储到照片相簿(最近项目) → 结束重复`，三处关键变量全对，放行实测。

**沉淀**：让用户实操前先做一次「结构确认」，能省掉一轮失败往返。

#### 轮 6：权限弹窗之后，弹出 Bad URL hash

**我**：选了「始终允许」后，Quick Look 显示 `Bad URL hash`（附 2 张截图）。

**Claude Code 的排障过程**（本案例技术含量最高的一环）：

1. OCR 读截图：确认是 CDN 返回的错误文案，请求确实发出去了
2. **系统性变异测试**（curl）：对那条已验证有效的 URL 逐项做破坏，观察服务器的不同反应：

| 破坏方式 | CDN 返回 | 错误文案 |
|---|---|---|
| 原始 URL | 200 image/jpeg | — |
| `%3D` 解码/双重编码 | 200 | 服务端容忍 |
| 改 `oh` 一位 | 403 | **Bad URL timestamp** |
| 改 `oe` / `stp` / `_nc_ht` | 403 | **URL signature mismatch** |
| **删 `oh` 参数** | 403 | **Bad URL hash** ← 就是用户遇到的 |
| 删 `ig_cache_key` / `efg` / `ccb` / `se` | 200 | 不在签名覆盖范围 |

3. **错误文案指纹**定位：用户遇到的 "Bad URL hash" = URL 丢失了 `oh` 参数
4. 第一层归因：代码里 `img.getAttribute("srcset")` 拿到的是 **HTML 原文**，Instagram 把 `&` 写成了 `&amp;`，于是 `&oh=...` 被解析成名为 `amp;oh` 的参数 → `oh` 丢失。而 `img.currentSrc`/`img.src` 是浏览器解析过的干净 URL（这也解释了为什么手动复制出来的 URL 能用）
5. 顺带修了第二个 bug：原代码把 srcset 所有分辨率档（640/750/1080/1440）都加进来，同一张图会下好几份
6. 给出修复版：`textarea` 解码 HTML 实体 + srcset 只取最大宽度一档

**沉淀**：**用错误文案做指纹反推损坏类型**——服务器不同的报错文案是免费的调试探针，一个 curl 循环就能把「黑盒损坏」映射成「具体哪个参数出了问题」。

#### 轮 7：修复后仍然 Bad URL hash

**我**：「不行，还是 bad url hash」。

**Claude Code 的决策**（关键的思路转折）：

- srcset 修复后错误依旧 → 说明坏的不是（只是）srcset 的 URL，**iOS Shortcuts 在把 URL 文本转成 HTTP 请求时本身就会弄坏这种长签名 URL**（生态已知坑，具体机制黑盒）
- 尝试搜索社区方案：g-search 的 Playwright 浏览器未装、WebSearch 报错、DuckDuckGo 判定 botnet、Bing 忽略引号短语返回垃圾结果——**公司网络下搜索引擎全部不可用**
- 果断放弃「搞清楚 Shortcuts 怎么弄坏 URL」这条黑盒路，**改为绕过**：
  - 早在轮 2 的 curl 就看到响应头 `access-control-allow-origin: *` → **允许网页内跨域 fetch**
  - 新方案：**JavaScript 在网页里直接 fetch 图片 → FileReader 转 base64 → completion 传回 → 快捷指令 Base64 解码后存照片**。快捷指令从头到尾不再碰图片 URL
  - 同时在 catch 分支里返回含出错 URL 的 JSON，作为万一失败时的诊断通道

**沉淀**：**黑盒修复到黑盒绕过的切换时机**——当 (a) 已确认输入有效 (b) 损坏发生在不可观测的中间层 (c) 外部资料不可得时，继续猜中间层的成本高于直接换一条不经过它的路径。工程师的直觉常是「必须搞清楚为什么」，但「绕过」往往是更优解。

#### 轮 8：能下载了，但 3 张只下到 2 张

**我**：「可以了，牛逼，但是明明有 3 张图，为什么只下载了 2 张」。

**Claude Code**：

- 不是 bug：**Instagram 轮播图只把「看过/加载过」的图放进 DOM**——轮 1 的诊断报告早就显示了这一点（当时 928×1232 正好 2 张，第 3 张从未进过页面）
- 零改动解决：运行前把轮播图**从头滑到尾**（每张停 1 秒等加载）再滑回来，然后运行
- 预留了 Plan B：若 Instagram 会回收滑走的 DOM 节点，则改用「解析页面内嵌 JSON 抓全部轮播图」的版本（副作用：可能误抓推荐流图片，需要按 URL 路径前缀过滤——已探明：帖子图 `/v/t51.82787-15/`，头像 `/v/t51.89012-19/`）

#### 轮 9：3 张全部下载成功 ✅

---

## 四、最终方案（可直接复用）

### 4.1 快捷指令结构

```
快捷指令名：Instagram 贴子图片保存
├─ 接收：共享表单输入（App 和 Safari 浏览器网页）
├─ 运行 JavaScript（见 4.2，返回 base64 数组）
└─ 重复每一项
   ├─ Base64 编码（模式 = 解码）     输入 = 重复项目
   ├─ 存储到照片相簿（最近项目）      输入 = Base64 的结果
   └─ （调试期可临时插入「快速查看」验证，用完删除）
```

### 4.2 最终版 JavaScript（粘贴进「在网页上运行 JavaScript」）

```javascript
(() => {
  const found = new Map();

  // 解码 HTML 实体（&amp; → &）
  // getAttribute 拿到的是 HTML 原文，srcset 里的 & 是转义过的
  function decodeEntities(s) {
    const t = document.createElement("textarea");
    t.innerHTML = s;
    return t.value;
  }

  function add(raw, w, h) {
    if (!raw) return;
    try {
      const u = new URL(raw, document.baseURI);
      if (u.protocol !== "https:") return;
      // 过滤头像/图标：短边 < 250 的不要
      if (w && h && Math.min(w, h) < 250) return;
      // 完整保留 href（含签名参数），绝不截断
      if (!found.has(u.href)) found.set(u.href, 1);
    } catch (_) {}
  }

  const scope = document.querySelector("article")
             || document.querySelector("main")
             || document;

  for (const img of scope.querySelectorAll("img")) {
    const w = img.naturalWidth || img.width || 0;
    const h = img.naturalHeight || img.height || 0;

    // currentSrc / src 是浏览器解析后的干净 URL，直接用
    add(img.currentSrc, w, h);
    add(img.src, w, h);

    // srcset 先解码实体；只取宽度最大的一档，避免同一张图下载多个尺寸
    const srcset = decodeEntities(img.getAttribute("srcset") || "");
    let best = null, bestW = -1;
    for (const item of srcset.split(",")) {
      const parts = item.trim().split(/\s+/);
      const mw = parts[1] && parts[1].endsWith("w") ? parseInt(parts[1], 10) : 0;
      if (parts[0] && mw > bestW) { bestW = mw; best = parts[0]; }
    }
    if (best) add(best, w, h);
  }

  const urls = Array.from(found.keys());

  // 关键：在网页里直接 fetch 图片转 base64，
  // 不把 URL 交给快捷指令（iOS Shortcuts 会弄坏带长签名参数的 URL）
  Promise.all(urls.map(u =>
    fetch(u, { mode: "cors", credentials: "omit" })
      .then(r => {
        if (!r.ok) throw new Error(u + " -> HTTP " + r.status);
        return r.blob();
      })
      .then(b => new Promise((res, rej) => {
        const fr = new FileReader();
        fr.onload = () => res(fr.result.split(",")[1]);
        fr.onerror = () => rej(new Error("read error"));
        fr.readAsDataURL(b);
      }))
  ))
  .then(b64s => completion(b64s))
  .catch(e => completion(JSON.stringify({
    error: String(e),
    urlCount: urls.length,
    urls: urls
  }, null, 2)));
})();
```

### 4.3 日常使用流程

1. Safari 打开 Instagram 公开帖子（地址栏手动输入网址进入，避免 Universal Links 跳转到 App）
2. **轮播图从头滑到尾**，每张停留 1 秒等加载完成
3. 分享按钮 → 运行「Instagram 贴子图片保存」
4. 打开「照片」App 检查最近项目

### 4.4 已知边界

| 边界 | 说明 |
|---|---|
| 仅公开内容 | 不绕过登录墙，私密账号无解（也不应该有解） |
| 轮播图需预滑动 | 未加载的图不在 DOM（未来可用页面内嵌 JSON 解析解决） |
| 缩略图 vs 原图 | 拿到的是页面实际加载的最高分辨率（srcset 最大档），不保证是服务器存档原图 |
| 推荐流干扰 | 长页面下滑太多可能把「更多推荐」的图也扫进来，article/main 限定可缓解大部分 |
| 平台变化 | Instagram 改版（CDN 域名、DOM 结构、CORS 策略）可能导致失效，需按本文方法论重新诊断 |

---

## 五、踩坑清单（按发现顺序）

| # | 坑 | 现象 | 根因 | 解法 |
|---|---|---|---|---|
| 1 | ChatGPT 分享页无正文 | curl 拿到 1MB HTML 但没有对话内容 | 纯前端渲染，数据在 `__reactRouterContext` 的 turbo-stream 里 | 手写解码器：raw_decode 解 JS 字符串字面量 + 平铺数组递归 resolve |
| 2 | Safari 打开帖子跳 App | 链接自动唤起 Instagram App | Universal Links | 地址栏手动输入网址 |
| 3 | 「运行了但没图片」 | 快捷指令跑完照片没变化 | **下载/保存动作根本没加**（截图 OCR 才发现） | 补齐 重复/获取URL内容/存储照片 动作链 |
| 4 | Quick Look 显示 URL 文本 | 弹出的是网址不是图 | Quick Look 输入接了「重复项目」而非「获取 URL 内容的结果」 | 理顺动作顺序与变量 |
| 5 | Bad URL hash（第一次） | CDN 返回 403 Bad URL hash | srcset 经 `getAttribute` 拿到 `&amp;` 转义，`oh` 参数名变成 `amp;oh` | textarea 解码 HTML 实体 |
| 6 | srcset 多档全下 | 同一张图存了好几个尺寸 | 所有 srcset 候选都通过了尺寸过滤 | 只取最大宽度一档 |
| 7 | Bad URL hash（第二次） | 修复 #5 后依旧 | **iOS Shortcuts 本身会弄坏长签名 URL**（黑盒） | 绕过：网页内 fetch → base64 → 快捷指令解码保存 |
| 8 | 3 张只下 2 张 | 少一张 | Instagram 轮播图只把加载过的图放进 DOM | 运行前滑完整组轮播图 |
| 9 | 会话内读不了图片 | Read 工具对截图返回 Unsupported Image | 会话能力限制 | Swift + Vision 框架 OCR（十几行代码） |
| 10 | 搜索引擎全灭 | 需要查社区方案时 g-search/WebSearch/DDG/Bing 全部不可用 | 公司网络环境 | 靠 curl 实测自证，不依赖外部资料 |

---

## 六、方法论总结

### 6.1 人机协作的分工模型（本次实践的最核心产出）

```
┌─────────────── Mac (Claude Code) ───────────────┐
│  写代码 / 改代码 / curl 验证 URL / OCR 读截图      │
│  设计变异实验 / 分析错误指纹 / 决定修复还是绕过      │
└──────────────┬───────────────────────────────────┘
               │ 需求 Prompt / 代码 / 操作指引 ↓
               │ 诊断报告 / 错误文案 / 截图 / 实测结果 ↑
┌─────────────── 我 (iPhone) ─────────────────────┐
│  粘贴代码到快捷指令 / 实际运行 / 复制报告回传        │
│  截图配置界面 / 截图报错弹窗 / 授权 / 最终验收       │
└──────────────────────────────────────────────────┘
```

**关键设计**：

1. **反馈通道要显式建设**——「拿不到反馈」曾经是卡点，解法是让手机端产出结构化证据（JSON 诊断报告、截图、错误文案原文），每一轮交互都必须带回新信息
2. **AI 给的每个操作指引要精确到「搜索什么词、变量选什么」**，用户侧的操作成本越低，往返越少
3. **结构确认先于实测**——让用户跑之前先用截图核对配置，一轮确认省一轮失败
4. 登录、授权、实机验证只能由人完成，AI 不假装自己测过（这也是当初给 Claude Code 的需求 Prompt 里明确写的约束）

### 6.2 调试方法论

1. **分层测试**：A 提取 → B 下载 → C 保存，三个环节独立验证，一次只怀疑一层
2. **证据驱动**：每一轮先固化事实（curl 200、OCR 截图、错误文案），再提假设；Mac 端能验证的绝不靠手机端猜
3. **错误文案指纹**：服务器的差异化报错（Bad URL hash / Bad URL timestamp / URL signature mismatch）是免费的探针，用变异测试把「症状」映射到「具体损坏」
4. **知道何时绕过**：确认输入有效、损坏在黑盒中间层、外部资料不可得 → 三条全中就换路径，不跟黑盒死磕
5. **每轮都留诊断后门**：正式方案里依然保留 catch 分支的 JSON 输出，失败了也能带回定位信息

### 6.3 可复用的工具技巧

| 技巧 | 触发场景 | 要点 |
|---|---|---|
| turbo-stream/`__NEXT_DATA__` 解码 | 抓取前端渲染页面的真实内容 | 数据在 `streamController.enqueue()` 的 JS 字符串里，`json.raw_decode` 直接解字面量；平铺数组中 `{"_N": M}` = 键取 payload[N]、值取 payload[M] |
| Swift + Vision OCR | 会话工具读不了图片时 | `VNRecognizeTextRequest` + `recognitionLanguages: ["zh-Hans"]`，swift 直接跑无需编译 |
| curl 变异测试 | 黑盒服务返回含糊错误时 | 逐参数改/删/编码变换，把错误文案差异当指纹 |
| curl -I 快速验证 | 判断「URL 坏了」还是「链路坏了」 | 一条 HEAD 请求区分两种假设 |

---

## 七、遗留与展望

1. **轮播图免滑动**：解析页面内嵌 JSON（`script[type="application/json"]`，注意 `&` 转义）可拿到全部轮播图地址；过滤靠 URL 路径前缀（帖子图 `t51.82787-15` vs 头像 `t51.89012-19`）+ 可能需要按帖子 ID 过滤排除推荐流——待有需求时再做
2. **多平台扩展**：小红书/Pinterest 的 DOM 结构和懒加载策略不同，提取逻辑需按平台定制，但「网页内 fetch + base64 回传」的架构可直接复用
3. **iOS Shortcuts 弄坏 URL 的确切机制**：本次未深究（绕过了）。若未来要给别的平台做下载器且该平台 CDN 不开 CORS，就得回来正面解决——备选：URL 双重编码、拆分请求、或经 Mac/服务器中转
4. **快捷指令收尾**：调试用的「快速查看」用完可删，让保存流程完全无感

---

## 附：本实践的完整交互链条

```
[ChatGPT 13 轮] 需求 → 3 方案 → 通用版 → Instagram 版 → 跳转问题 → 无反馈问题
      → 微信小程序支线 → 回归主方案 + 确立 Claude Code 协作模式 → 诊断脚本 → 截图确认
[Claude Code 9 轮]
  R0 取回 ChatGPT 分享页（turbo-stream 解码）
  R1 诊断报告分析 → 测试 A 通过，给正式提取版
  R2 Quick Look 显示 URL → curl 验证 200 → 定位接线问题
  R3 截图 OCR → 发现下载动作缺失 → 补动作指引
  R4 截图 OCR → 结构确认，放行实测
  R5 权限弹窗 + Bad URL hash → 变异测试定位「丢 oh」→ srcset &amp; 修复
  R6 仍 Bad URL hash → 搜索不可用 → 判定 Shortcuts 黑盒损坏 → base64 绕过方案
  R7 成功，3 张只下 2 张 → 轮播 DOM 机制 → 滑动预加载
  R8 3 张全部下载成功 ✅
```

**一句话总结**：这次实践真正的产出不是那个快捷指令，而是验证了一套「AI 写码 + 人实操 + 结构化证据回传 + Mac 端验证兜底」的远程调试协作模式——手机端黑盒问题，靠 Mac 端白盒实验逐层攻破，攻不破的黑盒就绕过去。
