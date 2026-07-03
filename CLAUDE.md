# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

单文件纯 Web 3DXML 模型查看器，部署于 GitHub Pages。支持手机 PWA 安装，可在浏览器中直接查看 CATIA/Tecnomatix 导出的 3DXML 三维模型。

- **在线地址**: https://huitailang7.github.io/3dxml-viewer/
- **部署平台**: GitHub Pages (huitailang7/3dxml-viewer)

## 构建与部署

没有构建步骤——`index.html` 是唯一应用文件。`manifest.json`、`sw.js`、`icon-*.png` 为 PWA 辅助文件。

```bash
# 本地测试
cd C:/Users/Administrator/Desktop/3dxml-viewer-deploy && python -m http.server 8080

# 部署
cd C:/Users/Administrator/Desktop/3dxml-viewer-deploy
git add -A && git commit -m "描述" && git push

# 桌面同步（部署后更新桌面副本）
cp index.html C:/Users/Administrator/Desktop/3dxml-viewer.html
```

修改时直接在 `Desktop\3dxml-viewer.html` 上测试，确认没问题后同步到 deploy 文件夹并推送。

## 架构

### 核心流程

```
用户选择文件 → ThreeDXMLParser.load()
  ├── 文件分类（.3dxml / .3mf / .3DRep）
  ├── ZIP 检测 (PK 头 0x50 0x4B) → JSZip 解压
  ├── parseXML()     → 两遍扫描建立 treeDict + repDict
  ├── buildTree()    → 返回装配根节点
  └── buildModelFromTree() → 递归创建 THREE.Object3D 层级
        ├── 叶子节点 → parse3DRep() → THREE.BufferGeometry + Mesh
        └── 非叶子   → 递归子节点，应用 RelativeMatrix
```

### 3DXML 格式支持

**文件格式**：ZIP 包（内部含 .3dxml + .3DRep）和纯 XML + 独立 .3DRep 均支持。

**几何格式**：

| 格式 | format 属性 | 3DRep 内容 | 渲染 |
|------|-----------|-----------|------|
| TESSELLATED | `"TESSELLATED"` | XML 文本 | 完整解析 |
| UVR | `"UVR"` | CATIA V5 CFV3 二进制 | 仅红色球体占位 |
| 3MF | — | ZIP+XML（独立格式） | parse3MF() 完整解析 |

### 3DXML XML 元素解析（两遍扫描，非常重要）

**必须两遍扫描**——CATIA/Tecnomatix 导出的 XML 中 `InstanceRep` 出现在 `ReferenceRep` **之前**：

```
第一遍：Reference3D → treeDict[id]   /   ReferenceRep → repDict[id]
第二遍：Instance3D  → 父子关系+变换矩阵  /  InstanceRep → 叶子绑定几何
```

四种元素：
- `Reference3D` (id, name) — 装配节点定义
- `ReferenceRep` (id, associatedFile, format) — 几何体引用，`format` 决定解析路径
- `InstanceRep` (IsAggregatedBy→IsInstanceOf) — 叶子节点绑定 ReferenceRep
- `Instance3D` (IsAggregatedBy→IsInstanceOf, RelativeMatrix) — 父子关系+12值变换矩阵

**不改两遍顺序会导致 153 个零件绑定全部静默失败。**

### 3DRep XML 结构（深层嵌套，容易漏）

CATIA 导出的 TESSELLATED 3DRep 有多层 `BagRepType` 嵌套：

```
XMLRepresentation
  └── Root (BagRepType)
        └── Rep (BagRepType)
              └── Rep (BagRepType)        ← 可能嵌套多级
                    └── Rep (PolygonalRepType)
                          ├── PolygonalLOD (accuracy=16.5) → Faces → Face
                          ├── PolygonalLOD (accuracy=21.0) → Faces → Face  ← 多个LOD
                          ├── Faces                              ← 直接Faces（无LOD）
                          └── VertexBuffer → Positions / Normals
```

关键点：
- **必须递归/迭代遍历**多层 BagRepType 才能找到 PolygonalRepType
- VertexBuffer 和 Faces 在 PolygonalRepType 同级，Faces 可能在 PolygonalLOD 内
- 选 LOD 策略：取 accuracy **最小**的 PolygonalLOD（精度最高）
- 每 Face 支持 `triangles`、`strips`（逗号分隔多条带，奇偶翻转绕序）、`fans`（逗号分隔多扇面）三种属性

### ⚠️ JavaScript 栈溢出防范（已踩过的坑）

**绝对禁止 `push(...大数组)`**——展开运算符把每个元素当函数参数压栈，几万元素直接爆栈：

```javascript
// ❌ 错误：indices 有几万条目 → Maximum call stack size exceeded
allTriangles.push(...indices);

// ✅ 正确：逐元素循环
const _pushAll = (target, source) => {
  for (let i = 0; i < source.length; i++) target.push(source[i]);
};
_pushAll(allTriangles, indices);
```

**DOM 树遍历也避免递归**——使用显式栈（数组 push/pop）迭代遍历，防止深层嵌套 XML 溢出。

### 矩阵解析

RelativeMatrix 为 12 个空格分隔浮点数，行主序 4×3：
```
Row0: m0  m1  m2    → right   (X轴方向)
Row1: m3  m4  m5    → up      (Y轴方向)
Row2: m6  m7  m8    → forward (Z轴方向)
Pos:  m9  m10 m11   → 平移
```

Three.js 右手系不做 X 取反。Z-up→Y-up 仅在根节点做一次 `R_x(-90°)`。每层用 `matrix.copy()` + `matrixAutoUpdate = false` 避免 decompose/recompose 精度损失。

### 关键数据结构

- **ThreeDXMLParser**: 持有 treeDict、repDict、repFileMap、isUVR、is3MF、mf3Data
- **sharedGeomCache**: 全局 `Map<fileName, [{geometry, materials[]}]>`，避免重复解析同一 3DRep
- **entityMap**: `Map<treeNodeId, THREE.Object3D>`，用于结构树选中→高亮映射

### CDN 依赖

- `cdn.jsdelivr.net/npm/jszip@3.10.1/dist/jszip.min.js`
- `cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js`
- `cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/controls/OrbitControls.js`

### 已知限制

- UVR 二进制格式仅显示球体占位，无法解析真实几何
- 不支持纹理贴图、EXACT 格式
- 3MF 仅解析三角面片，不支持颜色/材质
- 仅支持前端文件选择/拖放，不能直接输入文件路径（浏览器安全限制）
