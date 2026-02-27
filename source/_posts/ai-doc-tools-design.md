---
title: 面向 AI Agent 的 Word 文档结构读取与精准编辑工具设计
date: 2026-02-27 15:45:00
categories:
  - 技术设计
  - AI Agent
tags:
  - Word
  - Aspose.Words
  - 文档自动化
  - 工具设计
---
## read_doc_structure & edit_doc_content

> 两个面向 AI Agent 的 Word 文档操作工具，基于 **Aspose.Words** 实现对 .docx 文件的结构化读取与精准编辑。

---

## 1. 设计目标

AI Agent 需要操作 Word 文档（如根据模板生成投标书、填写报告表格等），但 Word 文档是富格式二进制文件，AI 无法直接理解其内部结构。我们需要解决两个核心问题：

1. **让 AI "看懂" Word 文档** — 将复杂的 Word 结构转译为 AI 可理解的文本索引格式
2. **让 AI "精准编辑" Word 文档** — 通过索引定位 + 操作指令实现无损编辑，保留原始格式

### 1.1 设计约束

- 编辑后生成新文件，**不修改源文件**（不可变性）
- 所有编辑操作必须**保留原始格式**（字体、样式、合并单元格等）
- 工具输出面向 AI 消费，需要**紧凑且结构化**，减少 token 开销
- 支持大文档的**分段读取**，避免一次性输出超出上下文窗口

---

## 2. 核心概念：Body Children 扁平索引

### 2.1 Word 文档的内部结构

Word 文档由多个 Section 组成，每个 Section 包含一个 Body，Body 下的直接子节点只有两种类型：

```
Document
  └── Section[]
        └── Body
              ├── Paragraph  (段落)
              └── Table      (表格)
```

### 2.2 扁平化索引设计

我们将所有 Section 的 Body Children 收集到一个**扁平列表**中，每个元素分配一个全局索引 `[index]`。这个索引是两个工具之间的桥梁——`read_doc_structure` 输出索引，`edit_doc_content` 通过索引定位编辑目标。

```pseudocode
function collectAllBodyChildren(document):
    allChildren = []
    for each section in document.sections:
        body = section.body
        for each child in body.directChildren:
            if child is Paragraph or child is Table:
                allChildren.append(child)
    return allChildren
```

**示例**：一个包含标题、正文、表格的文档被展平为：

```
[0]  Paragraph  "第一章 项目概述"    (Heading1)
[1]  Paragraph  "本项目旨在..."       (Normal)
[2]  Paragraph  ""                    (空段落)
[3]  Table      5×4                   (表格)
[4]  Paragraph  "第二章 技术方案"    (Heading1)
[5]  Paragraph  "采用微服务架构..."   (Normal)
```

AI 看到 `[3]` 是一个 5 行 4 列的表格，就可以通过 `elementIndex=3` 精确定位到它进行编辑。

---

## 3. Tool 1: read_doc_structure

### 3.1 职责

将 Word 文档的物理结构转译为 AI 可理解的带索引文本，支持两种模式：

| 模式 | 用途 | 输出特点 |
|------|------|----------|
| **overview** | 快速了解文档全貌 | 标题完整、正文截断、表格仅显示尺寸、空段落折叠 |
| **detail** | 查看指定区间的完整内容 | 段落全文、表格逐单元格输出、含合并标记 |

### 3.2 参数设计

```
read_doc_structure({
    fileId:     string  (required)  // 文件唯一标识
    mode:       enum    (optional)  // "overview" | "detail", 默认 "overview"
    startIndex: int     (optional)  // detail 模式的起始索引（含）
    endIndex:   int     (optional)  // detail 模式的结束索引（不含）
})
```

### 3.3 返回结构

```
{
    fileId:        string   // 文件ID
    fileName:      string   // 原始文件名
    totalElements: int      // Body Children 总数
    content:       string   // 格式化后的结构文本
}
```

### 3.4 Overview 模式格式化逻辑

**设计原则**：用最少的 token 让 AI 掌握文档骨架。

```pseudocode
function formatOverview(bodyChildren):
    output = ""
    i = 0
    while i < bodyChildren.length:
        node = bodyChildren[i]

        if node is Paragraph:
            text = cleanControlChars(node.text)

            if isHeading(node):
                // 标题段落：完整保留，带样式标注
                output += "[{i}] P({styleName}) \"{text}\"\n"
                i++

            else if text is not empty:
                // 普通段落：截断到 50 字符
                preview = text.length > 50 ? text[0:50] + "..." : text
                output += "[{i}] P \"{preview}\"\n"
                i++

            else:
                // 连续空段落：折叠为一行
                start = i
                while i < length AND isEmptyParagraph(bodyChildren[i]):
                    i++
                if i - start == 1:
                    output += "[{start}] P \"\"\n"
                else:
                    output += "[{start}-{i-1}] {i-start}个空段落\n"

        else if node is Table:
            rows = node.rowCount
            cols = maxColumnCount(node)
            tableName = findPrecedingParagraphText(bodyChildren, i)
            output += "[{i}] TABLE {rows}×{cols} \"{tableName}\"\n"
            i++

    return output
```

**输出示例**：

```
[0] P(Heading1) "第一章 项目概述"
[1] P "本项目旨在构建一个智能化的文档处理平台，支持多种..."
[2-4] 3个空段落
[5] TABLE 8×4 "项目基本信息表"
[6] P(Heading1) "第二章 技术方案"
[7] P "采用微服务架构，核心技术栈包括..."
[8] TABLE 12×6 "技术选型对比表"
```

### 3.5 Detail 模式格式化逻辑

**设计原则**：完整暴露内容，为编辑操作提供精确的行列坐标。

```pseudocode
function formatDetail(bodyChildren, start, end):
    output = ""
    for i from start to end:
        node = bodyChildren[i]

        if node is Paragraph:
            style = getStyleName(node)
            text = cleanControlChars(node.text)
            styleStr = (style == "Normal") ? "" : "({style}) "
            output += "[{i}] P {styleStr}\"{text}\"\n"

        else if node is Table:
            output += "[{i}] TABLE {rows}×{cols} \"{tableName}\"\n"

            for r in 0..rowCount:
                output += "  r{r}: "
                for c in 0..cellCount:
                    cell = table[r][c]

                    if cell.verticalMerge == PREVIOUS:
                        output += "(merged)"
                    else:
                        cellText = cleanControlChars(cell.text)
                        if cellText is empty:
                            output += "{r{r},c{c}}"  // 空单元格用坐标占位
                        else:
                            output += "\"{cellText}\""

                        // 标注合并信息
                        if colSpan > 1:  output += "(colspan={colSpan})"
                        if rowSpan > 1:  output += "(rowspan={rowSpan})"

                    output += " | "  // 列分隔符
                output += "\n"

    return output
```

**输出示例**：

```
[5] TABLE 4×3 "项目基本信息表"
  r0: "项目名称" | "XXX智慧平台建设项目"(colspan=2)
  r1: "建设单位" | "XX市信息中心" | "联系人"
  r2: "项目预算" | {r2,c1} | {r2,c2}
  r3: "建设周期" | {r3,c1} | {r3,c2}
```

AI 看到 `r2,c1` 是空的，就知道该往 `elementIndex=5, rowIndex=2, colIndex=1` 写入内容。

### 3.6 关键细节

**表格命名**：向前回溯找最近的非空段落文本作为表格名称，遇到另一个表格则停止。这让 AI 能通过名称而非纯索引识别表格。

```pseudocode
function findTableName(bodyChildren, tableIndex):
    for i from tableIndex-1 down to 0:
        if bodyChildren[i] is Paragraph:
            text = cleanText(bodyChildren[i].text)
            if text is not empty:
                return truncate(text, 30)
        if bodyChildren[i] is Table:
            break  // 遇到另一个表格就停止
    return null
```

**合并单元格处理**：
- 横向合并（colspan）：检测 `HorizontalMerge.FIRST`，向右计数连续的 `PREVIOUS`
- 纵向合并（rowspan）：检测 `VerticalMerge.FIRST`，向下计数连续的 `PREVIOUS`
- 被合并的单元格显示 `(merged)` 标记

---

## 4. Tool 2: edit_doc_content

### 4.1 职责

基于 `read_doc_structure` 返回的索引，对 Word 文档执行精确编辑操作，生成新文件。

### 4.2 参数设计

```
edit_doc_content({
    sourceFileId: string  (required)  // 源文件ID

    extractRange: {                   // 可选：先提取子文档再编辑
        startIndex: int               // 起始元素索引（含）
        endIndex:   int               // 结束元素索引（不含）
    }

    operations: [                     // 编辑操作列表
        {
            type:         enum        // "set_cell" | "fill_table_rows" | "replace_text"
            elementIndex: int         // 目标元素的 body child 索引
            // ... 各操作类型的专属参数（见下文）
        }
    ]
})
```

### 4.3 返回结构

```
{
    fileId:           string    // 新生成文件的ID
    fileName:         string    // 输出文件名（原名_edited.docx）
    successCount:     int       // 成功操作数
    failCount:        int       // 失败操作数
    failedOperations: string[]  // 失败操作的描述
}
```

### 4.4 操作类型详解

#### 4.4.1 set_cell — 填充单个表格单元格

**用途**：向指定表格的指定单元格写入文本值。

```
{
    type:         "set_cell"
    elementIndex: int       // 表格在 body children 中的索引
    rowIndex:     int       // 行号（从 0 开始）
    colIndex:     int       // 列号（从 0 开始）
    value:        string    // 要写入的文本
}
```

```pseudocode
function executeSetCell(bodyChildren, operation):
    table = bodyChildren[operation.elementIndex]  // 必须是 Table
    row   = table.rows[operation.rowIndex]
    cell  = row.cells[operation.colIndex]

    writeCellValue(cell, operation.value)
```

#### 4.4.2 fill_table_rows — 批量填充表格行

**用途**：克隆模板行并批量填充数据，适合填写重复结构的表格（如人员清单、设备列表）。

```
{
    type:             "fill_table_rows"
    elementIndex:     int            // 表格索引
    templateRowIndex: int            // 模板行号（将被克隆后删除）
    rows:             string[][]     // 数据行，每行是一个字符串数组
}
```

```pseudocode
function executeFillTableRows(bodyChildren, operation):
    table       = bodyChildren[operation.elementIndex]
    templateRow = table.rows[operation.templateRowIndex]
    insertAfter = templateRow

    for each rowData in operation.rows:
        // 深拷贝模板行（继承全部格式：边框、底色、字体等）
        newRow = deepClone(templateRow)

        // 重置纵向合并属性，避免与模板行错误关联
        for each cell in newRow.cells:
            cell.verticalMerge = NONE

        // 按列填入数据
        for c in 0..min(rowData.length, newRow.cellCount):
            writeCellValue(newRow.cells[c], rowData[c])

        // 插入到模板行之后
        table.insertAfter(newRow, insertAfter)
        insertAfter = newRow

    // 删除模板行本身
    templateRow.remove()
```

**关键设计决策**：
- 克隆而非新建行，确保继承模板行的全部格式
- 必须重置 `verticalMerge`，否则克隆出的行会与原模板行产生错误的纵向合并
- 模板行最终被删除，结果文件中只保留数据行

#### 4.4.3 replace_text — 替换段落文本

**用途**：替换或设置段落中的文本内容。

```
{
    type:         "replace_text"
    elementIndex: int       // 段落索引
    oldText:      string    // 要替换的文本（空字符串 = 整段替换）
    newText:      string    // 新文本（支持 \n 创建多段落）
}
```

```pseudocode
function executeReplaceText(document, bodyChildren, operation):
    paragraph = bodyChildren[operation.elementIndex]  // 必须是 Paragraph

    if operation.oldText is empty:
        // 模式A：整段替换
        setParaText(paragraph, operation.newText)
    else:
        // 模式B：局部文本替换
        // 先合并碎片化的 Run，再执行查找替换
        mergeAdjacentRuns(paragraph)
        paragraph.range.replace(operation.oldText, operation.newText, matchCase=true)
```

**为什么需要合并 Run？**

Word 内部将一个段落的文本拆分为多个 Run（文本片段），即使连续文本格式相同也可能被拆成多个 Run（编辑历史导致）。例如 "项目名称" 可能被存储为 `["项", "目名", "称"]` 三个 Run。直接对单个 Run 做文本替换会匹配失败，因此需要先合并相邻同格式 Run。

```pseudocode
function mergeAdjacentRuns(paragraph):
    runs = paragraph.runs
    for i from runs.length-1 down to 1:
        current  = runs[i]
        previous = runs[i-1]
        if isSameFormat(current.font, previous.font):
            previous.text = previous.text + current.text
            current.remove()
```

**多段落支持**：当 `newText` 包含 `\n` 时，会在当前段落后插入新段落：

```pseudocode
function setParaText(paragraph, text):
    referenceFont = paragraph.runs[0].font  // 保存格式参考

    if text contains "\n":
        parts = text.split("\n")

        // 第一段写入当前段落
        paragraph.runs.clear()
        paragraph.runs.add(new Run(parts[0], referenceFont))

        // 后续段落逐个插入
        insertAfter = paragraph
        for i from 1 to parts.length:
            if parts[i] is empty: continue
            newPara = shallowClone(paragraph)  // 继承段落格式（对齐、缩进等）
            newPara.runs.add(new Run(parts[i], referenceFont))
            paragraph.parent.insertAfter(newPara, insertAfter)
            insertAfter = newPara
    else:
        paragraph.runs.clear()
        paragraph.runs.add(new Run(text, referenceFont))
```

### 4.5 操作执行顺序 — 逆序排列防索引漂移

**这是一个关键的设计决策。**

`fill_table_rows` 会插入新行，`replace_text` 的 `\n` 会插入新段落——这些操作会改变后续元素的实际位置。如果按正序执行，后面的 `elementIndex` 就不再准确。

**解决方案**：按 `elementIndex` 从大到小排序后执行，确保前面的元素不受影响。

```pseudocode
function editDocument(request):
    document = loadDocument(request.sourceFileId)

    // Step 1: 可选的范围提取
    if request.extractRange is not null:
        document = extractRange(document, request.extractRange)

    // Step 2: 按 elementIndex 降序排列
    sortedOps = request.operations.sortDescendingBy(op.elementIndex)

    // Step 3: 逆序执行，避免索引漂移
    for each op in sortedOps:
        try:
            executeOperation(document, bodyChildren, op)
            successCount++
        catch error:
            failCount++
            failedOperations.add(describe(op) + " -> " + error.message)

    // Step 4: 保存为新文件
    outputBytes = document.save(DOCX)
    newFileId = fileService.upload(outputBytes, buildOutputFileName(originalName))

    return { fileId: newFileId, successCount, failCount, failedOperations }
```

### 4.6 范围提取 — extractRange

**用途**：从大文档中提取指定区间作为独立子文档，再对子文档执行编辑。典型场景：从完整模板中提取某一章节生成独立文件。

```pseudocode
function extractRange(sourceDocument, range):
    bodyChildren = collectAllBodyChildren(sourceDocument)
    start = range.startIndex
    end   = range.endIndex

    newDocument = createEmptyDocument()
    currentSourceSection = bodyChildren[start].parentSection
    currentTargetSection = newDocument.firstSection

    // 复制页面设置（纸张大小、边距、方向等）
    copyPageSetup(currentSourceSection, currentTargetSection)
    // 复制页眉页脚
    copyHeadersFooters(currentSourceSection, currentTargetSection, newDocument)

    for i from start to end:
        elementSection = bodyChildren[i].parentSection

        // 检测 Section 边界变化，创建新 Section 保留分页
        if elementSection != currentSourceSection:
            currentSourceSection = elementSection
            newSection = createSection(newDocument)
            copyPageSetup(currentSourceSection, newSection)
            copyHeadersFooters(currentSourceSection, newSection, newDocument)
            currentTargetSection = newSection

        // 深拷贝节点到目标文档
        importedNode = newDocument.importNode(bodyChildren[i], deep=true)
        currentTargetSection.body.append(importedNode)

    return newDocument
```

**注意**：提取后操作的 `elementIndex` 是相对于提取后子文档的索引（从 0 开始），而非原文档索引。

---

## 5. 格式保留策略

这是工具设计中最关键的部分之一——编辑后的文档必须与原文档视觉一致。

### 5.1 单元格写入的格式继承

写入单元格时，需要继承原有字体格式。查找策略按优先级：

```pseudocode
function writeCellValue(cell, value):
    firstParagraph = cell.firstParagraph ?? createParagraph()

    // 三级字体查找策略
    referenceFont = findReferenceFont(cell)

    // 清空内容但保留段落格式（对齐方式、缩进等）
    firstParagraph.runs.clear()
    removeExtraParagraphs(cell)  // 只保留第一个段落

    // 用参考字体创建新 Run
    newRun = new Run(value)
    if referenceFont is not null:
        newRun.font.name       = referenceFont.name
        newRun.font.nameFarEast = referenceFont.nameFarEast  // 中文字体
        newRun.font.size       = referenceFont.size
        newRun.font.bold       = referenceFont.bold
        newRun.font.italic     = referenceFont.italic
        newRun.font.color      = referenceFont.color
    firstParagraph.runs.add(newRun)

function findReferenceFont(cell):
    // 优先级 1：当前单元格内已有的 Run
    run = findFirstRun(cell)
    if run: return run.font

    // 优先级 2：同行其他单元格的 Run
    for each siblingCell in cell.parentRow.cells:
        run = findFirstRun(siblingCell)
        if run: return run.font

    // 优先级 3：表格第一行（表头）的 Run
    headerRow = cell.parentTable.rows[0]
    for each headerCell in headerRow.cells:
        run = findFirstRun(headerCell)
        if run: return run.font

    return null  // 使用默认字体
```

### 5.2 行克隆的格式继承

`fill_table_rows` 通过深拷贝模板行实现，自动继承：
- 单元格边框样式
- 背景色 / 底纹
- 字体格式
- 单元格宽度
- 对齐方式

唯一需要重置的是 `verticalMerge` 属性，防止新行与模板行产生意外的纵向合并。

### 5.3 段落替换的格式继承

`replace_text` 和 `setParaText` 通过以下方式保持格式：
- 保存第一个 Run 的字体作为参考
- 新创建的 Run 复制该字体属性
- 新插入的段落通过 `shallowClone` 继承原段落的段落格式（对齐、缩进、行距等）

---

## 6. 协作工作流

两个工具设计为配合使用，典型流程：

```
AI Agent 工作流：

Step 1: 概览 → 了解文档结构
   read_doc_structure(fileId, mode="overview")
   → 获得文档骨架，识别关键元素位置

Step 2: 详情 → 查看目标区域
   read_doc_structure(fileId, mode="detail", startIndex=5, endIndex=10)
   → 获得表格的完整单元格内容，确认哪些需要填写

Step 3: 编辑 → 精确写入
   edit_doc_content(sourceFileId, operations=[
       { type: "replace_text", elementIndex: 1, oldText: "", newText: "..." },
       { type: "set_cell", elementIndex: 5, rowIndex: 2, colIndex: 1, value: "100万元" },
       { type: "fill_table_rows", elementIndex: 8, templateRowIndex: 1, rows: [...] }
   ])
   → 获得新文件ID

Step 4: 验证 → 检查编辑结果
   read_doc_structure(newFileId, mode="detail", startIndex=5, endIndex=10)
   → 确认内容已正确写入
```

---

## 7. 技术实现：Aspose.Words

底层文档操作完全基于 **Aspose.Words** 库，选择它的理由：

| 能力 | 说明 |
|------|------|
| 无需 Office 环境 | 服务端纯代码操作，不依赖 Microsoft Office 安装 |
| 完整的 DOM 模型 | Document → Section → Body → Paragraph/Table → Run/Cell 的完整对象模型 |
| 格式保真度 | 合并单元格、嵌套表格、复杂样式、页眉页脚等均能准确处理 |
| 跨 Section 操作 | 支持 `importNode` 在文档间迁移节点并保留格式 |
| 查找替换 | 内置 `Range.replace()` 支持精确文本替换 |

### 7.1 关键 API 映射

| 本文伪代码 | Aspose.Words 实际 API |
|------------|----------------------|
| `new Document(stream)` | `com.aspose.words.Document(InputStream)` |
| `document.save(DOCX)` | `Document.save(OutputStream, SaveFormat.DOCX)` |
| `node.getNodeType()` | `Node.getNodeType()` → `NodeType.PARAGRAPH` / `NodeType.TABLE` |
| `body.directChildren` | `Body.getChildNodes(NodeType.ANY, false)` |
| `paragraph.runs` | `Paragraph.getRuns()` → `RunCollection` |
| `cell.verticalMerge` | `Cell.getCellFormat().getVerticalMerge()` → `CellMerge.FIRST/PREVIOUS/NONE` |
| `cell.horizontalMerge` | `Cell.getCellFormat().getHorizontalMerge()` → `CellMerge.FIRST/PREVIOUS/NONE` |
| `isHeading(para)` | `para.getParagraphFormat().getStyleIdentifier()` in `HEADING_1..HEADING_9` |
| `deepClone(row)` | `Row.deepClone(true)` |
| `document.importNode(node, true)` | `Document.importNode(Node, boolean isImportChildren)` |
| `range.replace(old, new)` | `Range.replace(String, String, FindReplaceOptions)` |

### 7.2 移植说明

如需移植到其他平台，核心操作可替换为其他 Word 处理库：

| 替代方案 | 语言 | 说明 |
|---------|------|------|
| Apache POI (XWPF) | Java | 开源免费，但合并单元格处理和格式保真度较弱 |
| python-docx | Python | 轻量，适合简单场景，复杂表格合并支持有限 |
| Aspose.Words for .NET | C# | .NET 版本，API 几乎一致 |
| Aspose.Words for Python | Python | Python 版本，API 风格与 Java 版类似 |
| OpenXML SDK | C# | 微软官方，底层操作，需要自行处理高层抽象 |

**移植时需重点关注**：
1. 合并单元格的检测方式（不同库的 API 差异较大）
2. 深拷贝节点时的格式保留（尤其是跨文档导入）
3. Run 碎片化问题的处理

---

## 8. 控制字符清理

Word 文档内部的文本包含各种控制字符，必须在输出给 AI 之前清理：

```pseudocode
function cleanText(text):
    if text is null: return ""
    // 移除末尾控制字符（段落标记等）
    text = removeTrailing(text, range 0x00-0x1F)
    // 移除内嵌控制字符
    text = remove(text, [0x07, 0x0B, 0x0C, 0x0D])  // BEL, VT, FF, CR
    return trim(text)
```

| 字符 | 含义 | 处理 |
|------|------|------|
| `0x07` (BEL) | Aspose 表格单元格结束标记 | 移除 |
| `0x0B` (VT) | 垂直制表符 / 软换行 | 移除 |
| `0x0C` (FF) | 分页符 | 移除 |
| `0x0D` (CR) | 段落标记 | 移除 |

---

## 9. 错误处理设计

### 9.1 部分成功机制

`edit_doc_content` 采用"尽可能执行"策略——单个操作失败不阻断后续操作：

```pseudocode
for each operation in sortedOperations:
    try:
        execute(operation)
        successCount++
    catch error:
        failCount++
        failedOperations.add(describe(operation) + " -> " + error.message)

// 即使有失败，仍然保存并返回结果
save(document)
return { successCount, failCount, failedOperations }
```

AI Agent 可以根据 `failedOperations` 决定是否重试或调整策略。

### 9.2 常见校验

| 校验项 | 触发条件 | 错误信息 |
|--------|---------|---------|
| 文件不存在 | fileId 无效 | "文件不存在" |
| 类型不匹配 | set_cell 指向段落 | "elementIndex N 不是表格" |
| 索引越界 | rowIndex 超出行数 | "rowIndex N 超出表格行数 M" |
| 缺少参数 | set_cell 缺少 value | "set_cell 需要 value" |
| 范围无效 | startIndex >= endIndex | "endIndex 超出范围" |

