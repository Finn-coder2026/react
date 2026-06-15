明白,你要一份可以直接 review、能照着做的完整实现计划。我把前面所有调研和设计整合成一份 plan,锚定你们的真实代码。

# Plan:Tiptap 编辑器块级精确 diff(节点级 source-mapping)

------

## 一、问题背景

### 1.1 现状

git-backed Space 把 markdown 存在 git 仓库里,用户在 [TiptapEdit.tsx](https://claude.ai/epitaxy/frontend/src/components/ui/TiptapEdit.tsx) 里编辑,改完通过 `onChangeMarkdown`(经 [debouncedOnChangeMarkdown](https://claude.ai/epitaxy/frontend/src/components/ui/TiptapEdit.tsx:1174))把序列化后的 markdown 写回 git。

加载链路:`content` prop → [initialContent](https://claude.ai/epitaxy/frontend/src/components/ui/TiptapEdit.tsx:1159) 经 `ensureImageParagraphs(unwrapLinkedImages(...))` → `@tiptap/markdown` 解析成 ProseMirror 文档。 回写链路:[onUpdate](https://claude.ai/epitaxy/frontend/src/components/ui/TiptapEdit.tsx:1288) → [getEditorMarkdown](https://claude.ai/epitaxy/frontend/src/components/ui/TiptapEdit.tsx:126) → `editor.getMarkdown()`(顶层走自定义 [MarkdownDocument](https://claude.ai/epitaxy/frontend/src/components/ui/TiptapEdit.tsx:175) + [getTopLevelMarkdownSeparator](https://claude.ai/epitaxy/frontend/src/components/ui/TiptapEdit.tsx:160))。

### 1.2 痛点

`editor.getMarkdown()` 是**整篇重新生成**。它不知道哪块改过、哪块没改,对全文套用同一套序列化规则。结果:**用户只改了一个段落,未编辑的块也会因为"原始写法 ≠ 序列化器产出的写法"而产生 diff**(空段落 → 多空行、` `;标题旁空行被 `getTopLevelMarkdownSeparator` 的默认 `\n` 删掉;等等)。git diff 充满与本次编辑无关的噪音。

### 1.3 根本原因

markdown ↔ ProseMirror 是**有损往返**:富文本模型只保留语义,丢弃"原始写法的选择"(`*`/`_`、`-`/`+`、atx/setext、空行数……)。所以"整篇 round-trip 字符级一致"理论上做不到(CommonMark 作者已确认)。`normalize` 清洗只能统一空白,管不了写法重排。

### 1.4 目标

**用户编辑某一个顶层块时,git diff 只显示该块的变化,其他块字节级不变。**

- 段落 / 标题级改动:精确隔离到块。
- 容器块(列表 / 引用 / 表格)内的改动:隔离到整个容器块(见 §3.7 粒度声明)。

------

## 二、解决思路

### 2.1 核心洞察

既然整篇 round-trip 必然有损,就**不重新生成整篇**:**未编辑的块直接复用它在原文里的字节,只有改动过的块才重新序列化**。这样未编辑部分逐字节不变,diff 自然只剩真实改动。

### 2.2 三阶段机制

1. **加载**:用 markdown-it 的 `token.map`(行号区间)把原文切成"顶层块片段",每段保留原始字节;给每个编辑器顶层节点打 `blockId`,建立 `blockId → 原始片段` 映射。
2. **编辑追踪**:监听 transaction,把改动落到的顶层块 `blockId` 记进 `dirtySet`。
3. **回写**:逐块判断——干净块吐原始片段,脏块/新块才 `getMarkdown` + `normalize`,最后拼接。

副产品:整篇没编辑时,所有块都复用原片段 → 输出 == 原文(零 diff)**自然成立**,不需要单独的快速路径。

### 2.3 为什么自研

查证过:`@tiptap/markdown`、`tiptap-markdown`、`prosemirror-markdown`、`prosemirror-remark` 全是整篇 parse/serialize,**没有现成的增量 source-mapping 方案**。`token.map` 是 markdown-it 自带的数据,但"切块+映射+脏追踪+逐块回写"这套机制 tiptap 不提供,需要我们从零实现。

------

## 三、具体解决方案

### 3.1 架构总览

新增一个独立模块(纯逻辑,易测),编辑器组件只做接线:

```
frontend/src/components/ui/markdown/
  SourceMap.ts          // 切块、建映射、脏追踪辅助、逐块拼接(核心,无 React 依赖)
  SourceMap.test.ts     // 单元测试
  BlockIdExtension.ts    // Tiptap extension:给顶层节点加 blockId 全局属性
```

[TiptapEdit.tsx](https://claude.ai/epitaxy/frontend/src/components/ui/TiptapEdit.tsx) 里:加载时调用切块 + 组装、注册 `BlockIdExtension`、`onUpdate` 里把 `getEditorMarkdown` 换成 source-map 版序列化。

### 3.2 数据结构

```typescript
/** One top-level markdown block plus its trailing separator, kept verbatim. */
interface BlockFragment {
  blockId: string;     // stable id for this editing session
  raw: string;         // original bytes, INCLUDING the blank line(s) after it
  startLine: number;   // 0-based, inclusive
  endLine: number;     // 0-based, exclusive
}
```

组件内用 ref 持有会话级状态(不进 React state,避免重渲染):

```typescript
const originalMarkdownRef = useRef<string>("");           // full original, for fallback
const fragmentMapRef = useRef<Map<string, string>>(new Map()); // blockId -> raw fragment
const dirtyBlockIdsRef = useRef<Set<string>>(new Set());
const sourceMapEnabledRef = useRef<boolean>(true);        // false => degrade to whole-doc
```

### 3.3 阶段一:加载切块 + 建映射

**切块**——`@tiptap/markdown` 的 markdown manager 内部已有 markdown-it 实例;若不便复用,自带一个 `MarkdownIt()`(配置要和解析时一致)。顶层块 = `token.level === 0` 的块级 token,用其 `.map` 行号区间从原文切片:

```typescript
function sliceTopLevelBlocks(markdown: string, md: MarkdownIt): BlockFragment[] {
  const lines = markdown.split("\n");
  const tokens = md.parse(markdown, {});
  const fragments: BlockFragment[] = [];
  let cursorLine = 0;
  for (const token of tokens) {
    // Only the opening token of a top-level block carries the full .map range.
    if (token.level !== 0 || !token.map) continue;
    const [start, end] = token.map;
    // Bytes from cursorLine..end include the block AND the blank lines after the
    // previous block, so separators travel with the block (see §3.5).
    const raw = lines.slice(cursorLine, end).join("\n");
    fragments.push({ blockId: nextBlockId(), raw, startLine: cursorLine, endLine: end });
    cursorLine = end;
  }
  // trailing bytes (final newline etc.) appended to the last fragment
  if (cursorLine < lines.length) appendTail(fragments, lines.slice(cursorLine));
  return fragments;
}
```

**节点打 blockId + 组装**(解决"markdown-it token 与 tiptap 节点不对齐"的关键)——**按块独立解析再组装**,保证片段与顶层节点严格一一对应:

```typescript
function buildDocWithBlockIds(fragments: BlockFragment[], editor: Editor): JSONContent {
  const content: JSONContent[] = [];
  for (const frag of fragments) {
    const json = editor.markdown.parse(frag.raw);   // parse this block alone
    for (const node of json.content ?? []) {
      content.push({ ...node, attrs: { ...node.attrs, blockId: frag.blockId } });
    }
    fragmentMapRef.current.set(frag.blockId, frag.raw);
  }
  return { type: "doc", content };
}
```

`blockId` 通过 `BlockIdExtension` 声明为全局属性,且**不渲染进 DOM**(`renderHTML: () => ({})`、`parseHTML: () => null`),只活在 ProseMirror 文档里:

```typescript
export const BlockIdExtension = Extension.create({
  name: "blockId",
  addGlobalAttributes() {
    return [{
      types: ["paragraph", "heading", "bulletList", "orderedList",
              "taskList", "blockquote", "codeBlock", "table", "horizontalRule"],
      attributes: { blockId: { default: null, rendered: false } },
    }];
  },
});
```

### 3.4 阶段二:脏追踪

```typescript
editor.on("transaction", ({ transaction }) => {
  if (!transaction.docChanged) return;
  if (isSyncingContentRef.current || !editorInitializedRef.current) return; // programmatic/setContent
  markDirtyBlocks(transaction, editor.state.doc);
});

function markDirtyBlocks(tr: Transaction, doc: PMNode) {
  // For each changed range (after mapping), resolve to the depth-1 ancestor block.
  tr.mapping.maps.forEach((stepMap, i) => {
    stepMap.forEach((_oldStart, _oldEnd, newStart, newEnd) => {
      const from = tr.mapping.slice(i + 1).map(newStart);
      const to = tr.mapping.slice(i + 1).map(newEnd);
      doc.nodesBetween(clamp(from), clamp(to), (node, pos) => {
        const $pos = doc.resolve(pos);
        const top = $pos.depth === 0 ? node : $pos.node(1);
        if (top?.attrs?.blockId) dirtyBlockIdsRef.current.add(top.attrs.blockId);
        return false; // top-level only
      });
    });
  });
}
```

增删 / 拆分 / 合并的处理:

- **新插入的块**:`attrs.blockId` 为 `null`(独立 parse 没给它) → 回写时当脏块重新序列化。
- **拆分**(回车把一段拆两段):一半继承原 `blockId`(已脏),另一半 `null`(新块)→ 两块都重写,周围干净块不动。
- **合并 / 删除**:结果块标脏;被删块的 id 从映射移除(可选,留着也无害,回写时按当前文档遍历)。

### 3.5 阶段三:回写(逐块复用/重写 + 拼接)

```typescript
function serializeWithSourceMap(editor: Editor): string {
  if (!sourceMapEnabledRef.current) return normalize(getEditorMarkdown(editor) ?? "");
  const parts: string[] = [];
  editor.state.doc.forEach((topNode) => {
    const id = topNode.attrs?.blockId as string | null;
    if (id && !dirtyBlockIdsRef.current.has(id) && fragmentMapRef.current.has(id)) {
      parts.push(fragmentMapRef.current.get(id)!);        // verbatim reuse (separators included)
    } else {
      parts.push(ensureBlockSeparator(normalize(renderSingleBlock(editor, topNode))));
    }
  });
  return stitch(parts);
}
```

**块间分隔策略**:因为 §3.3 切块时把"块后面的空行"一起切进了 `raw`,干净块自带原分隔;脏块/新块经 `normalize` 后用规范 `\n\n` 收尾。`stitch` 负责消除首尾多余换行、保证文件末尾单个 `\n`。

### 3.6 接入编辑器组件

- 加载:在 `useEditor` 之后,markdown 模式下用 `sliceTopLevelBlocks` + `buildDocWithBlockIds` 生成带 `blockId` 的初始文档,`setContent` 进去;同时填 `originalMarkdownRef` / `fragmentMapRef`。
- extensions 数组([TiptapEdit.tsx:1207](https://claude.ai/epitaxy/frontend/src/components/ui/TiptapEdit.tsx:1207))加入 `BlockIdExtension`。
- [onUpdate](https://claude.ai/epitaxy/frontend/src/components/ui/TiptapEdit.tsx:1288):`getEditorMarkdown(editor)` 改为 `serializeWithSourceMap(editor)`。
- 外部内容刷新(切文档/模式)时重置 `fragmentMapRef`、`dirtyBlockIdsRef`,重新切块。

### 3.7 边界与降级(必须实现)

1. **跨块的链接引用定义**(`[id]: https://...` 定义在别处):按块独立 parse 会断。**降级**:加载时检测到原文含引用定义(正则 `/^[ \t]*\[[^\]]+\]:\s+\S+/m`)→ 设 `sourceMapEnabledRef = false`,该文档退回整篇 `getMarkdown` + `normalize`(即现有行为)。
2. **容器块粒度声明**:列表/引用/表格是一个顶层块,**块内改一处 → 整块重写**,块内其他项的写法可能被规范化。本期**不做容器内递归 source-mapping**(那是后续增强)。
3. **跨会话不持久**:`blockId` 只活在本次会话(不写进 markdown)。每次加载重新切块、重建映射 → 所有块初始都是干净的,只有本会话编辑产生脏块。这正确,无需持久化。
4. **解析不一致兜底**:`buildDocWithBlockIds` 里任一块 parse 抛错或块数异常 → 降级整篇。

### 3.8 涉及文件

| 文件                                                         | 改动                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `markdown/SourceMap.ts` (新)                                 | `sliceTopLevelBlocks` / `buildDocWithBlockIds` / `markDirtyBlocks` / `serializeWithSourceMap` / `stitch` / `normalize` |
| `markdown/BlockIdExtension.ts` (新)                          | 顶层节点 `blockId` 全局属性                                  |
| [TiptapEdit.tsx](https://claude.ai/epitaxy/frontend/src/components/ui/TiptapEdit.tsx) | 注册扩展、加载切块、`onUpdate` 改用新序列化、ref 重置        |
| `markdown/SourceMap.test.ts` (新)                            | 单元测试                                                     |

纯逻辑层,**无新增用户可见文本,不涉及 intlayer**。

### 3.9 分步落地(建议拆成几个 commit / 小 PR)

1. **SourceMap 模块骨架 + `normalize` + 单测**(把现有 normalize 逻辑沉淀进来)。
2. **`BlockIdExtension` + 切块 `sliceTopLevelBlocks` + 单测**(各类块、列表、围栏代码、表格的切分)。
3. **`buildDocWithBlockIds` 组装 + 加载接线 + 降级检测**。
4. **脏追踪 `markDirtyBlocks`**(transaction 映射到顶层块)。
5. **`serializeWithSourceMap` + `stitch` + `onUpdate` 接线**。
6. **feature flag + 边界兜底 + 全干净=原文的回归校验**。

### 3.10 测试与验证(遵守项目规则)

- 为 `SourceMap.ts` 写充分的 **vitest 单测**:切块正确性、round-trip(无编辑 → 输出严格等于输入)、改一块只动一块、拆分/合并、含引用定义降级、容器块整块重写。
- 按 CLAUDE.local.md 规则:**实现期间和收尾只跑 `npm run lint` / `npm run build`(tsgo 类型检查),不执行任何测试命令**;测试文件随业务代码写好、保证 lint 通过即可。
- 行为正确性靠手动 / E2E(由你或 CI 跑),不由我执行。

### 3.11 风险与回退

| 风险                 | 应对                                                         |
| -------------------- | ------------------------------------------------------------ |
| token 与节点对齐出错 | 按块独立 parse + 组装(§3.3),从根上保证一一对应               |
| 跨块引用定义         | 检测到即整篇降级(§3.7)                                       |
| 脏追踪漏标/误标      | feature flag 一键回退整篇 `getMarkdown`;充分单测覆盖增删拆合 |
| 性能(高频 onUpdate)  | 保留现有 `debounce`;回写是 O(块数)                           |
| 容器块内噪音         | 明确为已知粒度限制,本期不做递归                              |

------

## 待你拍板的几个点

1. **容器块粒度**:本期"列表/表格/引用内改一处 → 整块重写"可接受吗?(可接受=工程量中等偏大;要容器内也精确=要做递归,工程量大很多。)
2. **行尾两空格硬换行**:`normalize` 要不要清行尾空格?(用硬换行就保留,不用就清掉更干净。)
3. **feature flag**:是否要一个开关(默认开,出问题可关回整篇序列化)?我建议要。
4. **归属**:当前分支是 `fix/JOLLI-1734`(structural ops),这个 markdown 问题建议**开新 ticket / 新分支**,你定。

这四点你定了,我就按这份 plan 开始实现(先建 SourceMap 模块和单测)。如果 plan 里某个环节你想调整设计,也直接说,我改完再动手。