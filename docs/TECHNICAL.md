# 技术总览

[← 文档首页](README.md)

## 1. 项目概况

| 项 | 内容 |
|---|---|
| 类型 | VR 材质解谜游戏 |
| 团队 | 3 人小组；本人为核心开发 |
| 周期 | 2026.04 — 2026.06 |
| 引擎 | Unity 6000.0.59f2 · URP 17.0.4 |
| XR | XR Interaction Toolkit 3.3.0 · OpenXR 1.15.1 |
| 着色器 | 手写 HLSL（主揭示 Shader 与环境 Shader）；第一版揭示原型用 Shader Graph |
| 内容生产 | Blender 建模；ComfyUI 本地生成贴图 |
| AI 辅助 | Codex 辅助编码；需求拆解、代码审查与测试验证由本人完成 |
| 验证环境 | Unity 编辑器 + XR Device Simulator（没有真机数据，见 [验证与性能](VALIDATION.md)） |

**本人负责**：全部 Unity 开发（程序、着色器、交互）、渲染与动画，以及约 1/3 的 Blender 建模。另外两名组员各完成约 1/3 的建模。详见 [分工与内容归属](CREDITS.md)。

## 2. 游戏流程

```text
开始场景  ──注视停留──▶  塔外场景  ──走入入口──▶  塔内
                                                │
          ┌─────────────────────────────────────┘
          ▼
   一号房（8 件） ──传送门──▶ 二号房（4 件） ──传送门──▶ 三号房（6 件）
          │
          每个房间内：
          射线照射器物 → 看到真实质地、听到敲击声
            → 抓起器物 → 放到对应质地的槽位
               ├─ 放错：槽位闪红 + 错误音 + 器物被弹开
               └─ 放对：器物收起，槽位上的缩小模型恢复真实质地
            → 全部放对：灯光压暗 → 光柱升起 → 整房质地恢复 → 传送门出现
            → 走进传送门：黑场 + 音乐交叉淡化 → 进入下一个房间
```

## 3. 模块划分

| 层级 | 模块 | 职责 | 详见 |
|---|---|---|---|
| 渲染 | `DUIZHAO/Stylized Reveal` Shader（5 个 pass） | 工笔风格光照、PBR、世界空间揭示遮罩、描边、阴影与深度 | [SHADER](SHADER.md) |
| 渲染 | `DUIZHAO/Gongbi Environment` Shader | 墙面、地面等环境物体的工笔矿物色表面，与器物共用同一套色彩语言 | [SHADER](SHADER.md) |
| 交互 | 射线揭示与表面声音 | 射线检测、写入揭示参数、按材质播放敲击声 | [INTERACTION](INTERACTION.md) |
| 交互 | 抓取修饰 | 抓起时统一尺寸、吸附到手前方、松手后受重力 | [INTERACTION](INTERACTION.md) |
| 谜题 | ID 槽位、错误反馈、缩小模型、缩略提示 | 判定、反馈、提示 | [INTERACTION](INTERACTION.md) |
| 进度 | 房间谜题管理、完成演出、出口传送 | 完成检测 → 演出 → 传送门 → 换房间 | [INTERACTION](INTERACTION.md) |
| 反馈 | `RevealPatchDriver` / `RevealSlotSwatch` / `RevealRoomFinale` | 局部、单件、整房三个尺度的揭示和控制权 | [FEEDBACK](FEEDBACK.md) |
| 内容配置 | `RevealMaterialLibrary` / `RevealPuzzleTable` | 共享材质族；每个房间的器物 → 质地映射 | [MATERIAL_LIBRARY](MATERIAL_LIBRARY.md) |
| 内容生产 | ComfyUI 客户端与生成窗口 | 贴图生成、平铺、回填配置、来源记录 | [AIGC](AIGC.md) |
| 工具 | 编辑器菜单与批处理入口 | 场景构建、碰撞体、石台、审计、截图 | [TOOLS](TOOLS.md) |
| 验证 | PlayMode 测试 | 揭示、放置、整房恢复、三个房间的完成流程 | [VALIDATION](VALIDATION.md) |

划分原则：**材质决定表面怎么呈现，配置表决定器物属于哪种质地，运行时组件决定什么时候变化。** 调整视觉风格不需要改谜题判定，增减谜题也不需要改 Shader。

## 4. 数据流

```text
            RevealMaterialLibrary（材质族：贴图 / 模板 / 敲击声 / 来源）
                       │
                       ▼
            RevealPuzzleTable（每房间：itemId → familyId，房间参数）
                       │  编辑器同步
        ┌──────────────┼──────────────┬────────────────┐
        ▼              ▼              ▼                ▼
   器物揭示材质     桌面样片材质     缩小模型材质       敲击声
        │              │              │                │
        └──── 运行时：MaterialPropertyBlock 写入 _HitPosWS / _Radius / _Softness ────┘
                       ▲
     射线（局部） / 槽位（单件） / 房间完成（整房）  —— 三者按控制权分配
```

## 5. 代码规模

只统计本人编写的代码，不含第三方资源包与 Unity 示例。

| 部分 | 文件 | 行数 |
|---|---:|---:|
| 揭示 Shader 与环境 Shader（`.shader` + `.hlsl`） | 7 | 约 740 |
| 交互、谜题与进度框架（C#） | 42 | 约 3,900 |
| 揭示运行时组件（C#） | 8 | 约 650 |
| 编辑器工具与 AIGC 管线（C#） | 20 | 约 4,400 |
| PlayMode 测试（C#） | 4 | 约 560 |

## 6. 关键技术决定

| 决定 | 原因 | 代价 |
|---|---|---|
| 揭示 Shader 从 Shader Graph 改成手写 HLSL | 需要自定义的阶梯光照、带揭示裁切的描边 pass，以及多个 pass 共用同一份揭示函数 | 需要自己维护阴影、深度、立体渲染等 pass |
| 揭示范围用世界空间，纹样用物体空间 | 多零件器物共用一个揭示球；抓取移动时纹样不滑动 | 需要做缩放补偿 |
| 运行时参数用 `MaterialPropertyBlock` | 不为每个命中状态复制材质，保持 SRP Batcher 兼容 | 多个组件写同一组属性时，需要明确控制权 |
| 新组件在 `LateUpdate` 里接管属性，不改原有脚本 | 原有的判定和声音逻辑可以继续工作，迭代风险低 | 每帧多一次射线查询 |
| 一个材质族 ID 派生四种输出 | 避免视觉改了、声音或缩小模型还是旧的 | 新增质地要走同步命令 |
| 场景改造全部由编辑器工具完成 | 可重复执行，原场景不受影响 | 工具本身需要维护 |
