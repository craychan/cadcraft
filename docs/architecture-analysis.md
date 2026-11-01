# CADCraft 架构与实现逻辑分析（金字塔汇报）

> **依据**：`README.md`、`CLAUDE.md`、`Cargo.toml`、`crates/engine/src/{lib.rs,prompt.rs,cmd/mod.rs}`、`crates/doc/src/lib.rs`、`crates/render/src/lib.rs`、`crates/io/src/lib.rs`、`crates/mcp/src/lib.rs`、`crates/ui-egui/src/lib.rs`、`apps/cadcraft/src/main.rs`、`apps/cadcraft-web/src/main.rs`。
> 标 `〔推断〕` 者由注释/用法推得，未直读该模块源码。
> 配套脑图：`docs/architecture-mindmap.md`。

---

## 一、结论先行（塔尖）

**CADCraft 是一条「命令总线」串起的六层纯 Rust CAD 应用：下层只提供能力，上层 UI 只是「读状态 + 发命令」的薄壳；桌面、Web、CLI、MCP 四个宿主共享同一套行为。**

推论：**读懂命令系统 = 读懂 80% 的 CADCraft。** 渲染、IO、UI 都在给同一套命令做输入与输出。

---

## 二、三个设计支点（第二层）

### 支点 A｜文档即快照

| 落点 | 机制 |
|---|---|
| `DocState.doc: Arc<Drawing>` | 图纸是不可变快照 |
| `EntityStore`（`crates/doc`） | 实体集合写时复制 |
| `Snapshot { label, doc, selection }` | 撤销栈只存 `Arc` 指针 |
| `Session::doc_mut()` | `Arc::make_mut` 取写权 |
| `Arc::ptr_eq(before, now)` | 判断「这步到底改没改图纸」 |

**收益**：撤销 = 弹一个 `Arc`；命令失败 = 换回旧 `Arc` 即完全回滚，无需逆操作日志。代价：每次真实修改克隆一次 `Drawing` 外壳（实体数组仍共享）。

这也是 `guard.rs` 的底座：`catch_unwind` 捕获 panic 后直接换回 `run.before`。

### 支点 B｜命令注册表

```rust
pub struct CommandSpec {
    pub id: &'static str,                 // AutoCAD 命令名小写：line / circle / zoom
    pub label: &'static str,
    pub menu: &'static [&'static str],    // ["Draw","Circle",…]，同时是 parity 指标
    pub shortcut: Option<&'static str>,
    pub aliases: &'static [&'static str], // L / C / PL …
    pub params: &'static str,             // JSON 参数文档
    pub enabled: Enabled,                 // 前置条件
    pub run: Run,                         // 程序化：fn(&mut Session,&Value)->Result<Value>
    pub undoable: bool,
    pub interactive: Option<Factory>,     // 交互：造一台 Interactive 状态机
    pub transparent: bool,                // 可否 'ZOOM 嵌套
}
```

**同一个 `id` 两个入口，从不两套实现：**

- `run` —— JSON 参数、**永不弹对话框**，供控制通道 / MCP / CLI / 脚本 / `cmd {json}`。
- `interactive` —— `Prompt` 机器、逐点提示，供菜单 / 工具栏 / 键盘。

`command_specs()` 由 30+ 个 `mod` 的 `specs()` 拼成（file→edit→qselect→view→draw→annotate→hatch→blocks→modify→array→layer→layout→props→inquiry→settings→table→constraints→lisp…），**这份清单本身就是 AutoCAD 兼容度量尺**。

`resolve_alias()` 优先级：**精确命令名 > 用户 ALIASEDIT 别名 > 内置别名**。

### 支点 C｜交互即状态机

```rust
pub trait Interactive: Send {
    fn prompt(&self, s: &Session) -> Prompt;
    fn input(&mut self, s: &mut Session, i: Input) -> Result<Step>;
    fn begin(&mut self, _s: &mut Session) -> Result<Step> { Ok(Step::Continue) }
    fn preview(&self, _s: &Session, cursor: Vec2) -> Vec<EntityKind>;
    fn drag(&self, _s: &Session, cursor: Vec2) -> Option<Drag>; // 整对象变换高效预览
    fn editing_mtext(&self) -> Option<MText>;                  // 原地多行文本编辑
}
```

- `Prompt`：`message / keywords / default / accept / base / deferred / corner / grip / picker / raw`。
- `Accept` 位掩码（`point/number/text/select/enter`）——**UI 完全据此决策**：是否显示拾取框、是否支持直接距离输入、关键字怎么渲染。
- `Input ∈ {Point, Deferred, Keyword, Text, Pick, Enter, Cancel}`；`Step ∈ {Continue, Done, Cancel}`。

**收益**：交互命令可在无 UI 环境下「喂输入序列」测试（`cmd/*_tests.rs` 普遍如此）。`Deferred`（延迟切点/垂足）也被抽象成输入的一种。透明命令靠 `Session.suspended: Option<Running>`，**只有一层**，与 AutoCAD 一致。

---

## 三、分层与依赖方向（第三层）

```
L0  geom, dxf                     纯数学、标签格式
L1  color, doc                    颜色、图纸数据库
L2  fonts, render, constraints    字体排版 / 显示列表 / 参数化求解
L3  io                            DXF·DWG·SVG 读写，PNG·SVG·PDF 导出
L4  engine                        会话、命令、交互机、捕捉、选择、撤销、LISP
L5  ui-egui, mcp                  可替换前端 · MCP 服务端
    apps                           cadcraft（桌面）· cadcraft-cli · cadcraft-web
```

**铁律**：`ui-egui` 以下不得依赖 egui / eframe / winit / wgpu / rfd（`cargo xtask layers` 强制）。→ 前端真能换：Web 版就是同一套 `CadApp` 跑在 eframe web runner 上。

---

## 四、逐层职责（第四层）

| 层 | Crate | 核心类型 / 职责 |
|---|---|---|
| L0 | `cadcraft-geom` | `Vec2`、`Mat3`、`Bounds2`、`Polyline`、`Prim`；求交、偏移、bulge、样条 〔推断〕 |
| L0 | `cadcraft-dxf` | DXF 标签流读写（ASCII/二进制），`Version` R12…2018 |
| L1 | `cadcraft-color` | ACI 索引色、真彩色、`Color::resolve(layer,block)`、`display_rgb`（颜色 7 随背景） |
| L1 | `cadcraft-doc` | `Drawing`（header/layers/linetypes/text_styles/dim_styles/blocks/model/layouts/vports/ucss/groups/plot_style_tables/constraints/parametric/scales）、`EntityStore`(CoW)、`Space::{Model,Paper}`、`EntityKind`、`Common`、`foreign`（未知数据原样回写） |
| L2 | `cadcraft-fonts` | 自有单笔画字体 + TTF/OTF（`skrifa`）、HarfBuzz 级整形（`harfrust`）、Unicode BiDi、MTEXT 排版与分栏 |
| L2 | `cadcraft-render` | `build()` → `DisplayList{prims,verts,tris,bounds,texts,pens}`；`Kind::{Polyline,Tris,Mask,Point,Infinite}`；`raster`（CPU 出图）、`paper`、`clip`、`pen` |
| L2 | `cadcraft-constraints` | 几何/尺寸约束、DOF 分析、LM 求解（`lm.rs`）、表达式（`expr.rs`） |
| L3 | `cadcraft-io` | `read()`（内容识别 DWG→DXF 桥；扩展名识别 SVG/SVGZ）、`write_with()`、`png_framed`/`svg`/`pdf`、`check_window` 敌意校验 |
| L4 | `cadcraft-engine` | `Session`、`DocState`、`Settings`(≈60 系统变量)、`Running`、`LastUsed`、`snap`、`select`、`grips`、`assoc`、`undoctl`、`sysvars`、`lisp`、`guard` |
| L5 | `cadcraft-ui-egui` | `CadApp`（薄壳）、`UiState`（可序列化，自动化可读写）、`docking`、`canvas`、`cmdline`、`control`（JSON-lines）、`menus::UI_COMMANDS`、`theme::Tokens`、`i18n`（13 语言） |
| L5 | `cadcraft-mcp` | JSON-RPC 2.0 over stdio；`Backend::{Headless, Remote}` —— 同套工具可进程内无头跑，也可连运行中的 App |
| app | `cadcraft` | eframe 宿主、日志、工作区持久化、macOS 原生菜单、SpaceMouse、系统主题/动画、控制服务器 |
| app | `cadcraft-web` | eframe web runner、WebGPU/WebGL2、`?host=parent` iframe 桥、浏览器下载交付 |
| app | `cadcraft-cli` | `info / convert / run / commands / mcp`，全走 `engine` |

---

## 五、关键数据流（第五层）

### 5.1 键盘输入 → 图元落盘

```
"line 0,0 @10,0"
  CadApp::cmdline(text)
    ├─ menus::run_ui_command? 否
    └─ Session::cmdline
        ├─ current_prompt 是自由文本 → 整行直送
        └─ input_line：按空格切 token，空 token = Enter
            └─ start("line")
                ├─ resolve_alias → "line"
                ├─ find_command → &CommandSpec
                ├─ (spec.enabled)(session)          // 无图纸则拒绝
                ├─ guard::Guarded::create(factory)  // 包 catch_unwind
                ├─ running = Running{ id, machine, before: Arc<Drawing>, selection_before }
                └─ feed(None) → machine.begin()
            └─ 后续 token → typed(tok) → input(Input::…)
                ├─ point_mods（FROM / M2P / .x）优先
                ├─ parse_point_with（x,y / @dx,dy / @d<a / #x,y）
                ├─ 直接距离输入（沿橡皮筋或追踪路径）
                └─ feed(Some(i)) → machine.input() → Step
                     Continue → 保留机器，revision += 1（增量重绘）
                     Done     → finish(run, false)
                     Cancel   → "*Cancel*" + finish(run, true)
```

`finish()` 收尾顺序（**顺序本身是设计的一部分**）：

1. `refuse_non_finite` 校验非有限坐标
2. `after_command` 后处理管道：**`groups::after_command` → `constraints::after_command` → `assoc::after_command`**（编组剔除 → 约束推断/清理/重解 → 关联尺寸与填充跟随源几何）
3. 若透明命令或 LISP 分组则并入外层；否则 `push_undo(Snapshot{ label, doc: run.before, selection })`

> 后处理落在命令自己的撤销步内，所以「删掉被标注的线」时尺寸自动跟随，且一次 Ctrl+Z 全回退。

### 5.2 每帧渲染

```
App::logic(ctx)
  ├─ cad.logic(ctx)
  │    ├─ LISP 时间片（长脚本每帧一片，Esc 可停）
  │    ├─ 关闭确认 / 字体安装 / 主题解析 / 动画解析
  │    ├─ drain_control（控制通道请求，含截图排队）
  │    ├─ finish_opening（后台线程读完的文件）
  │    ├─ autosave_tick / gpu_fault_tick
  │    └─ menus::shortcuts
  └─ spacemouse.poll / workspace.save_if_changed
App::ui(ui)
  └─ CadApp::draw
       ├─ chrome::title_and_toolbar / menu_bar / status_bar / file_tabs
       ├─ CentralPanel → docking::show（Tool Sets / Layers / Properties）
       │    └─ canvas：读 Session 视图 + render::build(d, space, Options)
       │         → DisplayList → 上传 GPU（?cpu 时用 egui shape）
       └─ dialogs / prompt_picker / closing / autosave 提示
```

渲染的工程细节：

- **并行构建**：模型空间按 256 实体一块 `rayon` 并行，再按绘制顺序拼接，结果与单线程一致（`build_parallel`/`append`）。
- **活动视口**：画布用 `viewport_content()` 单独构建活动视口再裁剪，**平移缩放视口内不动整张布局**（`Options::skip_viewport`）。
- **块展开预算**：`MAX_BLOCK_EXPANSION = 2_000_000`，防敌意文件指数爆炸（深度另有 `MAX_BLOCK_DEPTH`）。
- **可取消**：`Options::cancel: AtomicBool`，缩放继续时丢弃过时重建。
- **`PlotText`**：绘图时记下哪些填充三角形其实是字形，PDF 可输出真实文本。

### 5.3 文件进出

```
读: io::read(bytes, name)
      ├─ cadcraft_dwg::is_dwg → dwg_to_dxf_noted → dxf_read（保留 read_notes）
      ├─ ".svg"/".svgz" → svg_read
      └─ ".dxf"/"" → dxf_read
写: io::write_with(d, name, SaveOptions{version, binary})
      ├─ dxf → dxf_write::tags → write_ascii_as / write_binary
      ├─ dwg → 先出 R2000 DXF，再 dxf_to_dwg_as（低于 2004 自动抬到 2004）
      └─ svg/png/pdf → write_framed(window: Option<Bounds2>)
```

**`IoHooks` 依赖倒置**：engine 的 `file` 命令通过 `set_io / set_write_as / set_entity_codec / set_deliver` 注入实现，**engine 不认识 `cadcraft-io`**。桌面与 Web 各自 `install_io()`。

### 5.4 四宿主一套内核

| 宿主 | 入口 | 连接方式 |
|---|---|---|
| 桌面 `cadcraft` | `eframe::run_native` | `install_io()` + `--control PORT` 本机 JSON-lines |
| Web `cadcraft-web` | `eframe::WebRunner` | 同 `install_io()` + `set_deliver(download)` 或 `host::deliver` |
| CLI `cadcraft-cli` | `run/convert/info/commands/mcp` | 直接构造 `Session`，无 UI |
| MCP `cadcraft-mcp` | stdio JSON-RPC | `Backend::Headless` 或 `Backend::Remote`（转控制通道） |

> MCP 工具：`command_line`、`execute`、`inspect_drawing`、`query_entities`、`render`(PNG)、`screenshot`、`ui_click`。「程序化调用永不弹对话框」是这条链能通的前提。

---

## 六、工程纪律（第六层）

| 规则 | 落地 |
|---|---|
| **永不崩溃**（优先级最高） | `unsafe_code="forbid"`；各 crate 根部 `#![deny(clippy::unwrap_used, expect_used, panic, unimplemented, todo, unreachable)]`；`guard::Guarded` 外层 `catch_unwind`；敌意输入 `get()` 不用 `[]`、检查算术、限循环与分配 |
| **净室实现** | 只黑盒观察 AutoCAD；绝不复用 SHX 字体 / `.pat` / `.lin` / 图标 / 帮助文本；格式来自公开规范 |
| **资产必须自有** | 图标代码绘制（`icons.rs`）、填充与线型自定义（`library.rs`）、制图字体自有（`stroke.rs`）；任何文件资产须在 `ATTRIBUTION.md` 登记（`cargo xtask assets`） |
| **一切皆命令** | 用户可见行为必须落成 `CommandSpec` + 测试；UI-only 命令集中 `menus::UI_COMMANDS` |
| **UI 必须薄** | 面板只读 `Session`，动作走 `app.run(id,params)` / `app.start(id)` / `app.cmdline(text)`；颜色只来自 `theme::Tokens` |
| **质量门禁** | `cargo xtask ci`：fmt · clippy -D warnings · tests · assets · layers · wasm |

---

## 七、风险与改进机会

**结构性风险**

1. **`Session` 接近上帝对象**：约 50 个字段，混合「图纸会话 / 交互状态 / 输入缓冲 / LISP / 剪贴板 / 绘图窗口 / 上次选项」。`execute()` 单函数横切 bedit 拒绝、refedit 校验、non-finite 检查、LISP 撤销分组四件事。→ 可拆 `Runtime` / `Input` / `Automation`。
2. **`after_command` 管道顺序是隐式契约**（groups → constraints → assoc），只靠注释守护。→ 建议加顺序断言测试（删除被约束且有关联尺寸的图元，验证一次撤销后三者一致）。
3. **`Arc::make_mut` 克隆成本**：每次真实修改克隆 `Drawing` 外壳（表、块 map、布局 vec）。拖动夹点走 `drag()` 矩阵不落盘，影响有限，但值得在超大图纸上 profiling。
4. **渲染无空间索引**：`Options::view` 只做粗略过滤，视口内容靠 `entity_bounds().intersects(win)` 逐实体判。engine 已有 `spatial` 模块，说明方向已定。
5. **`catch_unwind` 是安全网不是设计**：真防线是 clippy deny + `Result` 文化。出现 `EngineError::Internal` 在仓库语义里等价于「发现 bug」。

**上手成本风险**

- `cmd/` 下 80+ 文件，`draw.rs` / `modify.rs` / `annotate.rs` 仍是巨型文件。新贡献者需要一条「最小完整路径」入口（即下文第八节）。

---

## 八、快速掌握路径（约半天）

| 步 | 动作 | 目标 |
|---|---|---|
| 1 | `cargo run --release -p cadcraft -- --sample --control 7979` | 先看见它跑起来 |
| 2 | 读 `command_specs()` | 建立能力地图（即 AutoCAD 兼容度表） |
| 3 | 读 `Session::{execute,start,input,feed,finish}` | **引擎主干，这五处是全部** |
| 4 | 追踪 `cmd/draw.rs` 里的 LINE 机器 | spec→factory→prompt→input→Step 走通一遍 |
| 5 | 读 `render::{build_space, entity_parts}` | 理解「图纸 → 三角/折线」投影 |
| 6 | 读 `io::{read, write_with}` | 理解格式桥与版本策略 |
| 7 | 控制通道验证：`{"id":1,"method":"cmdline.input","params":{"text":"circle 0,0 5"}}` 再 `ui.screenshot` | 形成「假设→注入→观察」闭环 |

**调试三件套**：`RUST_LOG=cadcraft*=debug`（引擎会记录每条命令行消息）· 控制通道 `engine.execute` + `inspect_drawing` · `cadcraft-cli run --script` 无头回归。

---

## 九、一页总结

> CADCraft = **6 层代码 × 1 条命令总线 × 3 根柱子**。
> 六层保证「能力可复用、前端可替换」；一条总线（`CommandSpec` + 双入口）保证「人、脚本、Agent 见到同一套行为」；三根柱子（CoW 快照撤销 / 命令注册表 / 交互状态机）保证「便宜、一致、可测」。
> 工程纪律（永不崩溃、净室、资产自有）不是附加项，而是这个仓库敢对标 AutoCAD 的前提。
