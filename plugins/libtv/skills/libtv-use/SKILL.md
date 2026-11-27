---
name: libtv-use
description: "Use whenever the user wants to generate images, videos, or audio, test or compare models (\"用这几个模型跑一下\", \"跑个demo\"), read or edit canvas nodes, manage scripts, storyboards, timelines, characters, voices, or assets, or export media in LibTV — even when no specialized workflow skill is named. Routes general creation and model evaluation directly to LibTV MCP tools and delegates specialized productions (director treatment, Blender white-model previs to live action, music-driven product ad) to dedicated LibTV skills."
---

# LibTV 通用创作与模型路由入口

当用户希望生成图片、视频或音频、对比测试多款模型、读写画布节点、管理剧本/分镜/时间线/角色/声音/资产或导出交付成片时，优先使用本技能路由到当前已连接的 LibTV MCP 工具。

## 1. 运行时建立与工作流真源

- 具体的工具路由、模型契约与证据边界，以当前 LibTV MCP 连接 `initialize` 返回的 `instructions`（工作流摘要）及各工具实时 schema 为唯一权威依据。
- 每次会话操作画布前，先调用 `doctor` 确认身份与连接就绪，再用 `target_project` 绑定唯一、精确的画布项目（复用用户指定项目，无明确项目时可先通过 `list_projects` / `project_list` 查询，或按用户要求调用 `create_project` 新建）。
- 打开浏览器画布是按需动作（仅在用户主动要求查看画布，或工具返回 `makeVisible=true` 时打开），不作为普通媒体生成或模型评测的强制前置步骤。

## 2. 通用生图、生视频、音频与模型评测

当用户提出“用这几个模型跑一下”“跑个视频/图片 demo”“根据这张参考图生成视频”等通用创作或评测需求时，直接通过 LibTV MCP 工具执行，不进入复杂专项工作流：

1. **模型发现与实时契约**：
   - 调用 `search_models` 查找目标模型（如 Seedance、Kling、Hailuo、Veo、Sora、Wan、Seedream、Flux、Midjourney、Minimax 等），需要推荐视频模型时可调用 `recommend_video_models`。
   - 调用 `get_model` 读取选中模型的实时 `inputContract`。将 `inputContract.model` 传入顶层 `model` 字段，`params` 只填写 `inputContract.parameterSchemas` 中声明的创作参数。
   - 用户明确指定的参数优先，其次是宿主基于任务作出的必要选择；其余参数省略以沿用 Web 默认值，不擅自拉满最高分辨率、最长时长或额外张数（图片新生成默认 `count=1`）。
2. **多模型对比评测（跑 Demo）**：
   - 当用户要求“用 A 和 B 两个模型分别跑一下同一提示词/参考图”时，先对各模型分别执行 `search_models` 与 `get_model` 核对各自支持的模式与参数，再在同一目标画布下为每个模型创建或提交独立的生成任务（各自使用独立的稳定 `idempotency_key`），保留清晰的节点命名与引用连线以便在画布上直观对比。
3. **选择正确的执行入口**：
   - **纯文本或无画布节点引用的持久任务派发**：使用 `generation_submit`（传入 `request={model, prompt, params}`）。
   - **视频生成与带画布参考素材（图片/视频/音频节点、首尾帧、续写、整段编辑）**：按当前环境的运行能力（见 `doctor` 与 `instructions`），使用 `generate_video`（配合 `intent`、`startFrameNode`、`endFrameNode`、`referenceImageNodes` / `referenceVideoNodes` / `referenceAudioNodes`）或 `create_node` / `update_node`（配合 `left` / `leftEnsure` 等连线参数与提示词中的 `{{Node <nodeId>}}` 引用）。
   - 若当前环境关闭画布内联运行（`run` 只接受 `false`），带参考素材的节点使用 `run=false` 准备并写入画布后，提示用户在浏览器画布上点击运行，不要删掉参考图悄悄退回纯文本生成。
4. **本地参考素材上传**：
   - 需要将用户提供的文件挂入画布作为参考时，按 `upload_ticket` → 直传真实字节 → `upload_complete` → `upload_media` 完整链路执行，不把本地路径或对话临时附件直接当作画布资源。
5. **任务等待、恢复与产物核验**：
   - `generate_video` 实际运行视频生成时默认在同一次调用内等待结果；其他异步任务或需恢复的任务通过 `task_status` 回读。
   - 调用未知或中断时，先用原 `task_id` 或原 `idempotency_key` 回读恢复，绝不换键重复发起付费生成。
   - 生成完成后使用 `read_node` / `inspect_media` 核验实际产物。

## 3. 画布编排、资产、剧本与导出交付

- **画布节点与分组**：使用 `read_project`、`get_project_snapshot`、`read_node`、`create_node`、`update_node`、`delete_node`、`manage_group` 读写节点、连线与分组。
- **角色、声音与资产库**：按需使用 `list_characters` / `get_character` / `save_character` / `generate_character_style`、`list_voices` / `get_voice` / `design_voice` / `synthesize_speech`、`search_assets` / `get_asset` / `save_asset_from_node` / `apply_asset_to_canvas` 管理跨项目复用的角色、音色与资产。
- **剧本、分镜与时间线剪辑**：新版剧本使用 `create_screenplay` / `list_screenplays` / `read_screenplay` / `write_screenplay`；分镜表与时间线使用 `create_script_v2` / `read_script_v2` / `edit_script_v2` / `read_timeline` / `apply_timeline_edit`；独立音频剪辑使用 `edit_audio`。
- **成片导出与本地交付**：使用 `export_readiness` / `export_video` / `export_status` 合成导出视频（支持已保存时间线变速、转场及按需中文字幕烧录），并通过 `prepare_project_delivery` 获取交付清单后由宿主实际下载保存并校验文件。

## 4. 专项制作流程分流

当用户的需求明确属于以下专项制作流程时，切换并遵循对应的专项 Skill，不在通用入口里简化其制作标准：

- **画布成果转 16:9 导演提案（Treatment / MV 阐述幻灯片）**：转交 `libtv-to-treatment`。
- **剧本 + 场景参考 → LibTV 双视角场景与角色三视图 → Blender 3D 白模运镜 → Seedance 2.5 实拍化**：转交 `libtv-blender-live-action`。
- **Zen Piano 真实配乐演奏录音 → 20 秒横向剖面一镜到底音乐驱动产品广告**：转交 `music-driven-product-ad`。
