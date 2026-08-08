# 把审阅搬到产物上,而不是留在对话里

> **出处** — [@petergyang](https://x.com/petergyang/status/2085776743642898847) · 工具 [petergyang/human-review](https://github.com/petergyang/human-review)(MIT,607 星) · 采集于 2026-08-09
> **原作者说法**:agent 写完 HTML/Markdown 后,用浏览器打开给人审;人像用 Google Docs 一样**直接改字 + 划词批注**,一次性把整批反馈发回 agent。他自己用它改 AI 生成的方案、落地页、localhost 应用,以及「删掉 AI 爱加的多余文案」。
> 个人实践,未经第三方验证。下文的机制拆解与安装坑是本仓库实测补充。

## 问题

给 AI 改稿的标准姿势是在 chat 里描述:

> 第三段把 X 改成 Y。第三张卡片和第一张重复了,删掉。CTA 再重写一遍。

这句话要经过四道转换:你心里想的 → 你写出的描述 → agent 的理解 → agent 改出的结果。然后你还得**逐条回去核对它有没有理解对**。稿子越长、页面越多,核对成本涨得越快,最后你花在验收上的时间超过自己动手改。

## 核心机制:把反馈从「描述」降级成「已完成的编辑」

这个回环里最关键的一条设计,在它 SKILL.md 的规则里写得很直白:

> **`edits` 是用户已经改好的东西。`after` 是他们的原话——原样搬过去,永远不要回退。**

也就是说,人不再**描述**要改什么,而是**直接改好**;agent 拿到的不是指令,是既成事实,它唯一的活是把这个事实**搬运**到源文件里(渲染的 HTML → MDX/Markdown 源)。

「理解」这一环被整个删掉了,失真也就没了。

批注(`comments`)才走原来的老路——那是真正需要 agent 判断的部分。**能自己一秒改完的直接改,需要商量的才留言**,两类反馈走两条通道,这是它比「截图丢给 AI」高明的地方。

配套的几条设计同样值得抄:

- **锚定用 prefix/quote/suffix 三段文本,不用行号。** 审的是渲染后的产物,没有行号可言,而且文件随时在变。三段锚定在文本漂移后仍能定位。
- **`kind: "url"` 必须改源,绝不把 HTTP 响应写回项目。** 审 localhost 页面时,agent 得自己找到对应的 MDX/TSX/模板去改。这条规则不写死,agent 迟早会把渲染结果覆盖进源码。
- **`kind: "moved"` 只搬位置不改内容**,用 `moved_after`/`moved_before` 两个邻居描述新位置——比传坐标或索引稳。
- **「不要回复。这里没有聊天。」** 人看到的是保存后自动刷新的页面。省掉了 agent 那段「我已为您修改了以下 5 处……」的复述——那段话除了消耗上下文和你的注意力之外没有任何作用。

## 为什么它是流程改动,不只是个工具

真正让「人在回路」落地的是这条:

```sh
human-review poll <target> --timeout 600
```

**这是阻塞的,而且 SKILL.md 明确要求 agent 不许在等待期间结束回合。**

差别很大。指望人「事后想起来去看一眼」,回路基本不会闭合;而把阻塞式等待嵌进 agent 的执行流,人的审阅就成了任务完成的**必经步骤**。反馈到达 → 应用 → 再 `--ack` 继续等,直到人说结束。

反馈状态是落盘的,poll 进程死了也不丢;`status` 子命令可以非阻塞地问「有没有攒着的反馈」,适合放在新回合开头。

## 怎么用

装(**别照 README 用 npx,见下面的坑**):

```bash
npm i -g "github:petergyang/human-review#v0.6.0" && human-review setup --global
```

`setup --global` 只做一件事:往 `~/.claude/`、`~/.codex/`、`~/.agents/` 三处的 `skills/human-review/SKILL.md` 写同一份说明书,教 agent 这套回环怎么走。没有账号、没有云服务、没有 API key,server 跑在本机。

用:

```bash
human-review path/to/file.html        # 或 .md,或 http://localhost:3000/xxx
```

浏览器打开后直接改字、划词批注、拖块换位、⌘K 加链接、粘贴图片(自动存到同级 `assets/`),点 Send 一次性发回。

**写产物时预埋区块名**,反馈回来就是有意义的标签而不是 DOM 猜出来的名字:

```html
<p data-block="问题陈述">…</p>
<div data-container="指标卡片">…</div>
```

`data-container` 还会让整块变成可点击的批注目标。这一步花几秒,回收的是每轮反馈的可读性。

## 安装坑(实测)

**npm 上的版本落后 GitHub 三周。** 实测 2026-08-09:npm `latest` 停在 **0.3.0**(2026-08-01),GitHub 已发到 **v0.6.0**(2026-08-08)。发布流水线是 `on: release: published` 触发的,v0.5.0/v0.6.0 的 Release 都发了但 npm 没跟上——发布 workflow 挂了。

后果是 README、SKILL.md 里那些 `npx -y human-review` 全部拉到 0.3.0,而**推文里新宣传的功能一个都不在里面**:实测 0.3.0 的 `src/` 里搜不到 ⌘K 链接、列表快捷键、拖拽、块移动(`moved_after`)的任何实现。0.3.0 的包里甚至还是小写的 `skill.md`,而 `setup.js` 读的是 `SKILL.md`——macOS 默认大小写不敏感才侥幸没崩,换到大小写敏感的文件系统上直接 ENOENT。

**修法是装 GitHub tag,而且顺序不能反:**

```bash
npm i -g "github:petergyang/human-review#v0.6.0"   # 先让真 binary 进 PATH
human-review setup --global                        # 再写 SKILL.md
```

`setup.js` 里的 `invocation()` 会探测 PATH 上有没有 `human-review`,据此决定写进 SKILL.md 的是裸命令还是 `npx -y human-review`——它还专门做了排除,不把 npx 自己的临时缓存路径当数(注释里写了这个坑:`npx ... setup --global` 期间 `which` 会误报成功,写出的裸命令在 npx 退出后就失效)。所以**先全局装、再 setup**,SKILL.md 里落的才是裸 `human-review`,agent 每次调用少一次 npx 解析,也彻底绕开 npm 的旧版本。

装反了就再跑一次 `setup --global` 覆盖。

## 适用与不适用

**适合**:方案/规划/报告/newsletter 草稿、落地页、幻灯片、localhost 应用——凡是**产物本身是给人读的、且你会想动手改几个字**的东西。文档越长、页面越多,相对 chat 描述的优势越大。

**不适合**:后端代码、配置、数据管线。这类产物没有「渲染后的样子」可审,该走 diff 和测试。也不适合你**根本不想自己动手**、只想说一句「你看着办」的场景——那还是直接在 chat 里说更快。

**注意**:Markdown 文件是渲染后审的,工具本身**从不写 Markdown 源**,所有改动都得 agent 翻译回 Markdown 语法落到源文件。如果 agent 偷懒直接写渲染后的 HTML,源文件就毁了——这条依赖 agent 守规矩,SKILL.md 写了,但值得你第一次用的时候盯一眼。
