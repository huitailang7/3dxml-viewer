# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

单文件纯 Web 3DXML 模型查看器，部署于 GitHub Pages。支持手机 PWA 安装，可在浏览器中直接查看 CATIA 导出的 3DXML 三维模型。

- **在线地址**: https://huitailang7.github.io/3dxml-viewer/
- **部署平台**: GitHub Pages (huitailang7/3dxml-viewer)

## 构建与部署

没有构建步骤——`index.html` 是唯一应用文件。`manifest.json`、`sw.js`、`icon-*.png` 为 PWA 辅助文件。

```bash
# 本地测试
cd C:/Users/Administrator/Desktop/3dxml-viewer-deploy && python -m http.server 8080

# 部署（修改后提交推送即可）
cd C:/Users/Administrator/Desktop/3dxml-viewer-deploy
git add -A && git commit -m "描述" && git push

# 桌面同步副本
cp index.html C:/Users/Administrator/Desktop/3dxml-viewer.html
```

## 架构

### 3DXML 格式支持

**ZIP 格式**（标准 CATIA 导出）和**纯 XML 格式**均支持。文件头 `PK` (0x50 0x4B) 检测决定路径。

**两种几何格式**：

| 格式 | 标志 | 3DRep 内容 | 渲染方式 |
|------|------|-----------|---------|
| TESSELLATED | `format="TESSELLATED"` | XML 文本 | 解析为 THREE.BufferGeometry |
| UVR | `format="UVR"` | CATIA V5 CFV3 二进制 | 渲染红色球体占位 |

UVR 检测在 `parseXML()` 解析 `ReferenceRep` 时设 `this.isUVR = true`。`buildModelFromTree()` 据此分支。

### 3DXML 数据结构

```
Reference3D (id, name)     → 装配节点
ReferenceRep (id, associatedFile, format) → 几何体引用
InstanceRep (IsAggregatedBy → IsInstanceOf) → 叶子绑定
Instance3D (IsAggregatedBy → IsInstanceOf, RelativeMatrix) → 父子+变换
```

`RelativeMatrix` 为 12 个空格分隔的浮点数，行主序 4x3 矩阵。

### 矩阵解析（关键逻辑，易出错）

参照 Unity `ModelTree.cs` 的 `LookRotation`，Three.js 右手坐标系不做 X 取反：

```javascript
// 从矩阵行提取轴，施密特正交化，用 makeBasis 构建
right   = ( m0,  m1,  m2).normalize()
up      = ( m3,  m4,  m5).normalize()
forward = ( m6,  m7,  m8).normalize()
position = (m9, m10, m11)
```

Z-up→Y-up 仅根节点做一次：`rootGroup.quaternion = R_x(-90deg)`。

每层用 `matrix.copy()` + `matrixAutoUpdate = false` 避免 Three.js 内部 decompose/recompose 引入浮点误差。

### 核心类

- **ThreeDXMLParser**: ZIP解压、XML解析、3DRep解析、矩阵解析。持有 treeDict/repDict/repFileMap
- **buildModelFromTree()**: 递归创建 THREE.Object3D 层级。叶子节点→mesh，非叶子→递归子节点
- **sharedGeomCache**: 全局 Map，缓存已解析的 BufferGeometry+Material。TESSELLATED 模式按文件名缓存，UVR 模式用 `__uvr_sphere__` 键

### CDN 依赖

使用 jsdelivr（国内可访问）:
- `cdn.jsdelivr.net/npm/jszip@3.10.1/dist/jszip.min.js`
- `cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js`
- `cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/controls/OrbitControls.js`

### 3DRep XML 格式 (TESSELLATED)

```
XMLRepresentation → Root → Rep(s)
  Rep → Faces → Face (triangles="0 1 2...")
          → SurfaceAttributes → Color (red/green/blue/alpha)
  Rep → VertexBuffer → Positions (逗号分隔浮点)
                      → Normals
```

3DRep 文件可能包含多个 `<Rep>` 元素，每个为一个独立零件实例。同一颜色的连续 Face 合并为同一 submesh。

### 已知限制

- UVR 二进制格式无法解析几何数据，仅显示球体占位
- 不支持纹理贴图
- 不支持 `format="EXACT"` 格式
