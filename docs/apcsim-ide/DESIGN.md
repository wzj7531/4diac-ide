# APCSIM IDE 设计方案

> 基于 Eclipse Theia + Langium + Sprotty 重新构建 IEC 61499 工程环境
>
> 版本:v1.0 草案

---

## 目录

1. [项目概述](#1-项目概述)
2. [技术选型](#2-技术选型)
3. [总体架构](#3-总体架构)
4. [领域模型设计](#4-领域模型设计)
5. [APCSIM DSL 语法设计](#5-apcsim-dsl-语法设计)
6. [持久化与互操作](#6-持久化与互操作)
7. [语言服务设计](#7-语言服务设计)
8. [图编辑器设计](#8-图编辑器设计)
9. [运行时、部署与调试](#9-运行时部署与调试)
10. [扩展机制](#10-扩展机制)
11. [工程结构](#11-工程结构)
12. [技术栈清单](#12-技术栈清单)
13. [实施路线](#13-实施路线)
14. [风险与缓解](#14-风险与缓解)
15. [附录:4diac-ide 现状实测数据](#15-附录4diac-ide-现状实测数据)

---

## 1. 项目概述

### 1.1 背景

4diac-ide 构建于 Eclipse RCP 之上,深度依赖 EMF、Xtext、GEF、SWT/JFace 等 Eclipse 生态框架。这套技术栈在能力上依然强大,但已不是当今主流的桌面端技术,在人才获取、生态活跃度、Web/云端部署能力上存在结构性劣势。

APCSIM IDE 是一次**从头构建**,而非迁移。它不继承 4diac-ide 的任何 Java 代码资产,但完整继承其**领域知识**与**文件格式契约**。

### 1.2 目标

- 提供功能对等于 4diac-ide 的 IEC 61499 工程环境
- 与现有 IEC 61499 XML 文件格式**双向无损兼容**,保证工程资产可迁入迁出
- 同一套代码支持**桌面(Electron)与浏览器**两种部署形态
- 具备**开放的第三方扩展机制**,支持设备厂商贡献设备描述、下装协议与代码生成目标
- 工程文件支持有意义的 `git diff` / `git merge`,使 PLC 工程可纳入现代软件工程流程
- 核心逻辑可在**无图形界面**环境下运行,支持 CI 流水线中的校验、编译与打包

### 1.3 非目标

- 不追求与 4diac-ide 的 UI 布局或交互一一对应
- 不提供 Eclipse 插件兼容层
- 首个里程碑不覆盖嵌入式/HMI 面板部署场景(该场景需另行评估原生方案)

### 1.4 命名约定

| 术语 | 含义 |
| --- | --- |
| APCSIM IDE | 产品名 |
| APCSIM DSL | 本方案定义的 IEC 61499 文本语法 |
| `.apc*` | APCSIM DSL 文本文件扩展名 |
| `.fbt` / `.sys` / `.dtp` / `.adp` / `.sub` | IEC 61499 XML 格式(兼容格式) |

---

## 2. 技术选型

### 2.1 选型结论

| 层次 | 选型 | 替代方案及排除理由 |
| --- | --- | --- |
| 应用外壳 | **Eclipse Theia** | VS Code 扩展(定制能力不足);fork Code OSS(永久合并债务);裸 Electron(等于重造 Theia) |
| 桌面运行时 | **Electron** | Tauri(WebView 碎片化,放弃 VS Code 扩展生态);Qt/Avalonia/WPF(见 2.3) |
| 语言工程 | **Langium** | Xtext(Java,导致跨语言双栈);手写解析器(放弃 LSP 基建) |
| 图形渲染 | **Sprotty**(+ GLSP 按需) | GEF(桌面专有);React Flow(缺少 LSP 集成与服务端模型) |
| 自动布局 | **ELK (elkjs)** | — |
| 表单/属性页 | **JSON Forms** | — |
| 模型框架 | **无**(Langium AST) | EMF(Java);EMF.cloud(已停摆,见 2.4) |

### 2.2 关键决策:文本优先

**决策:APCSIM IDE 采用文本优先架构。文本是唯一真相源,AST 由解析器派生,图形编辑器是文本的视图,所有图上操作最终转换为文本编辑。**

依据:

1. **Langium 的架构前提。** 在 Langium 中 AST 由 `DocumentBuilder` 从文档内容增量构建,不是可独立持有的对象图。任何"直接修改 AST 并持久化"的做法都是逆架构的。

2. **消除双真相源。** 图形优先方案需要在图模型与文本模型之间做双向同步,需处理并发编辑冲突、注释与格式保留、语义等价判定等问题。TypeFox 在同类项目中的结论是:让图始终由文本生成、绝不反向,比任何双向同步都容易得多。

3. **文件规模适配。** 4diac 现有类型库文件行数中位数为 38 行,最大系统文件 2805 行。这是"多文件、小文档"的工况,正是 Langium 文档级增量解析的最优场景。

4. **消除跨模型作用域解析。** 现有实现中 `STCoreScopeProvider`(348 行)与 `STAlgorithmScopeProvider`(88 行)需要通过 EMF 反射伸入 `libraryElement` 模型解析 `VarDeclaration`、`Algorithm`、`Method`。文本优先下,FB 接口声明与其 ST 算法位于同一文档、同一 AST,退化为普通的词法作用域查找。

5. **版本控制能力。** IEC 61499 XML 无法有意义地 diff 与 merge。文本格式使 code review、分支协作、冲突合并成为可能。

6. **免费获得的能力。** 撤销/重做直接复用编辑器文本撤销栈;多编辑器一致性天然成立;协同编辑可通过文本 CRDT(Yjs)实现,无需模型级 OT。

### 2.3 被排除的 UI 技术

- **WPF**:仅支持 Windows。
- **Qt**:图形性能与嵌入式部署能力最强,但需自建插件体系与语言工具链两大基建;离开 JVM/TS 生态;LGPLv3/商业双授权对下游集成商构成治理风险。**若产品定位收窄为 HMI 面板侧组态器,应重新评估此项。**
- **Avalonia**:跨平台可行,但图编辑与语言工具生态薄弱,且同样需自建插件体系。仅在团队为纯 .NET 班底时具备合理性。

### 2.4 被排除的模型框架

**EMF.cloud 不进入技术栈。** 实测其维护状态:

| 仓库 | 近一年提交 | 最后推送 | npm 周下载 |
| --- | --- | --- | --- |
| `emfcloud-modelserver` | 2 | 2026-07 | — |
| `modelhub` | 0 | 2025-04 | 12 |
| `emfcloud-modelserver-theia` | — | 2023-10 | 1 |
| `coffee-editor`(旗舰示例) | — | 2024-02 | — |
| `ecore-glsp` | — | 2023-02 | — |

对照活跃组件:GLSP 近一年 62 次提交、客户端周下载 2613;Langium 近一年 99+ 次提交;JSON Forms 周下载 26 万。结论:EMF.cloud 已事实停摆,其原本覆盖的场景由 GLSP 与 Langium 分别承接,而这两者均不依赖 EMF。

JS 侧 Ecore 实现同样排除:`ecore.js` 已停止维护,社区 fork `ecore-ts` 仅 5 star 且 npm 标注维护不良,CrossEcore 属研究性项目。

---

## 3. 总体架构

### 3.1 分层

```
┌─────────────────────────────────────────────────────────────┐
│  表现层 (Frontend)                                           │
│  Theia Workbench │ Monaco 文本编辑器 │ Sprotty 图编辑器      │
│  JSON Forms 属性页 │ 类型库树 │ 设备/资源视图 │ 在线监视     │
└──────────────────────────┬──────────────────────────────────┘
                           │ JSON-RPC over WebSocket
                           │ (LSP + 图形扩展方法 + 自定义服务)
┌──────────────────────────┴──────────────────────────────────┐
│  服务层 (Theia Backend, Node.js)                             │
│  ┌────────────────────────────────────────────────────┐     │
│  │ APCSIM Language Server (Langium)                    │     │
│  │  文法/解析 │ 作用域 │ 索引 │ 校验 │ 补全 │ 重构      │     │
│  │  + LangiumDiagramGenerator (langium-sprotty)        │     │
│  └────────────────────────────────────────────────────┘     │
│  ┌──────────────┐ ┌──────────────┐ ┌─────────────────┐      │
│  │ Model API    │ │ 类型库服务    │ │ XML 互操作层     │      │
│  │ 命令→文本编辑 │ │ 索引/解析     │ │ IEC61499 ↔ DSL  │      │
│  └──────────────┘ └──────────────┘ └─────────────────┘      │
│  ┌──────────────┐ ┌──────────────┐ ┌─────────────────┐      │
│  │ 求值引擎      │ │ 部署服务      │ │ 导出/代码生成    │      │
│  │ ST Evaluator │ │ 61499/OPC UA │ │ FORTE / Lua/FMU │      │
│  └──────────────┘ └──────────────┘ └─────────────────┘      │
│  ┌────────────────────────────────────────────────────┐     │
│  │ Debug Adapter (DAP)                                 │     │
│  └────────────────────────────────────────────────────┘     │
└──────────────────────────┬──────────────────────────────────┘
                           │ TCP / OPC UA / 文件
┌──────────────────────────┴──────────────────────────────────┐
│  外部系统                                                     │
│  FORTE 运行时 │ 设备 │ 工程文件系统 │ Git 仓库                │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 进程模型

| 进程 | 内容 | 桌面形态 | 浏览器形态 |
| --- | --- | --- | --- |
| Frontend | Theia 前端、Monaco、Sprotty 视图 | Electron Renderer | 浏览器页面 |
| Backend | Theia 后端、语言服务、模型服务、部署服务 | Electron 内的 Node 进程 | 远端 Node 服务器 |
| Language Server | Langium 语言服务 | Backend 内进程或独立进程 | 同左 |
| Debug Adapter | DAP 实现 | 独立进程 | 独立进程 |

**设计约束:所有领域逻辑必须位于 Backend。** Frontend 只负责渲染与交互采集。此约束保证:

- 浏览器形态与桌面形态共享全部业务代码
- 领域逻辑可在无 GUI 的 CLI/CI 环境中直接调用
- 大型工程的解析与索引不阻塞 UI 线程

### 3.3 无头模式

Backend 的核心能力必须能脱离 Theia 独立运行,以 CLI 形式暴露:

```bash
apcsim validate ./project              # 校验工程
apcsim build ./project --target forte  # 生成 FORTE 代码
apcsim convert ./legacy --to dsl       # XML → DSL
apcsim convert ./project --to xml      # DSL → XML
apcsim deploy ./project --device 192.168.0.10
```

这既是产品能力(CI 集成),也是架构约束(强制业务逻辑与 UI 解耦)。

---

## 4. 领域模型设计

### 4.1 核心原则:文法即元模型

APCSIM IDE **不设独立的元模型层**。领域模型由 Langium 文法中的显式 AST 类型声明定义,`langium-cli` 从中生成 TypeScript 类型与反射信息。

对照 4diac 现状:

| 项目 | 4diac-ide (EMF) | APCSIM IDE (Langium) |
| --- | --- | --- |
| 模型定义 | 14 个 `.ecore`,5351 行 | Langium 文法中的 `interface`/`type` 声明 |
| 生成代码 | 694,000 行 Java(全仓) | `ast.ts`,数千行 |
| 放大系数 | 约 20× | 约 1× |
| 运行时依赖 | EMF Runtime | 无 |

`lib.ecore` 的实际结构规模:133 个 EClassifier(19 抽象、6 接口)、209 个结构特征、84 处 containment、42 处 eOpposite、259 个 EOperation、4 个派生属性。这些落成 Langium 显式 AST 类型约为两三千行声明。

### 4.2 为什么不需要 Impl 层

4diac 的 `org.eclipse.fordiac.ide.model/src-gen` 共 107,579 行,其中 `*Impl.java` 占 55,096 行。这些代码承载的是 EMF 运行时协议,在本架构下逐项失效:

| EMF Impl 承担的职责 | APCSIM 中的处理 |
| --- | --- |
| `eNotify` 变更通知 | 不需要。变更事件在文档层面,由 `DocumentBuilder` 触发 |
| `eGet`/`eSet`/`eIsSet` 反射 | 不需要。TS 结构化类型 + Langium 生成的 `AstReflection` |
| containment 父子指针维护 | 不需要。解析器自动设置 `$container` / `$containerProperty` |
| `eInverseAdd`/`eInverseRemove` 双向引用 | 不需要。交叉引用单向惰性解析,反向通过索引查询 |
| `LibraryElementFactory` 构造 | 不需要。文本优先下不编程构造 AST 节点 |
| `LibraryElementSwitch` 访问者 | 生成的 `$type` 判别联合 + 类型守卫 |

42 处 `eOpposite` 尤其值得注意:在 EMF 中这是需要在 Impl 里维护、且容易被破坏的不变量;在 Langium 中转化为按需的索引查询(`References` 服务),不存在一致性风险。

### 4.3 显式 AST 类型声明

**必须显式声明 AST 类型,禁止依赖文法推断。** 校验器、图生成器、代码生成器、Model API 全部依赖这些类型,推断会使文法的细微改动意外破坏下游。

```langium
interface LibraryElement {
    name: string
    comment?: string
    identification?: Identification
    versionInfo: VersionInfo[]
    attributes: Attribute[]
}

interface FunctionBlockType extends LibraryElement {
    eventInputs: EventDeclaration[]
    eventOutputs: EventDeclaration[]
    inputVars: VarDeclaration[]
    outputVars: VarDeclaration[]
    sockets: AdapterDeclaration[]
    plugs: AdapterDeclaration[]
    body: BasicFBBody | CompositeFBBody | ServiceFBBody | SimpleFBBody
}

interface EventDeclaration {
    name: string
    typeRef?: @DataType
    with: @VarDeclaration[]
    comment?: string
}

interface VarDeclaration {
    name: string
    type: TypeReference
    arraySize?: ArraySpec
    initialValue?: STExpression
    comment?: string
}

interface ECState {
    name: string
    actions: ECAction[]
    layout?: Position
    comment?: string
}

interface ECAction {
    algorithm?: @Algorithm
    output?: @EventDeclaration
}

interface ECTransition {
    source: @ECState
    destination: @ECState
    condition: ECTransitionCondition
    layout?: Position
}
```

`@Type` 表示交叉引用,由 Langium linker 惰性解析。

### 4.4 文法分层

复用 4diac 现有的模块划分,概念上一一对应:

```
st-core.langium            ← 对应 structuredtextcore
  ├── 表达式、语句、类型引用、字面量
  │
  ├── st-algorithm.langium     ← 对应 structuredtextalgorithm
  ├── st-function.langium      ← 对应 structuredtextfunctioneditor
  ├── global-constants.langium ← 对应 globalconstantseditor
  ├── data-type.langium        ← 结构体/枚举/派生类型 (.dtp)
  │
  ├── fb-type.langium          ← FB 类型:接口 + BasicFB/CompositeFB/ServiceFB
  ├── adapter-type.langium     ← 适配器类型 (.adp)
  ├── fb-network.langium       ← 应用/子应用网络
  └── system.langium           ← 系统配置:设备、资源、映射
```

Langium 支持文法 import,`st-core` 作为公共基础被各上层文法复用,避免 4diac 现在四套 Xtext 语言各自生成一份 `.ide` 代码(共约 26 万行)的膨胀。

---

## 5. APCSIM DSL 语法设计

### 5.1 设计原则

1. **贴近 IEC 61131-3 惯例。** 工程师熟悉 `VAR_INPUT ... END_VAR` 这类结构,降低学习成本。
2. **信息完备。** 必须能承载 IEC 61499 XML 的全部信息,包括布局坐标、注释、版本信息、自定义属性。
3. **可读可 diff。** 每条语义单元独占一行或一个块,便于行级 diff。
4. **布局信息内联但可分离。** 坐标以注解形式内联,语法上明确标记为非语义信息,便于工具剥离比较。

### 5.2 功能块类型

以现有 `FB_RANDOM.fbt` 为对照,DSL 形式:

```iecst
FUNCTION_BLOCK FB_RANDOM
  COMMENT 'Generate a REAL Randomly'

  IDENTIFICATION
    Standard       := '61499-1';
    Classification := 'Mathematic';
    Function       := 'RANDOM';
    Type           := 'Mathematical function';
    Description    := 'Copyright (c) 2012 Profactor GmbH ...';
  END_IDENTIFICATION

  VERSION_INFO
    Organization := 'Profactor GmbH';
    Version      := '1.0';
    Author       := 'Gerhard Ebenhofer';
    Date         := '2012-05-31';
  END_VERSION_INFO

  EVENT_INPUT
    INIT : EInit WITH SEED;  // Initializes the random with the specified seed
    REQ  : Event;            // Calculates a new random number between 0 and 1
  END_EVENT

  EVENT_OUTPUT
    INITO : EInit;
    CNF   : Event WITH VAL;  // Execution Confirmation
  END_EVENT

  VAR_INPUT
    SEED : UINT := 0;  // the seed to initialize the random
  END_VAR

  VAR_OUTPUT
    VAL : REAL;  // Function output
  END_VAR

  BASIC_FB
    ECC
      INITIAL_STATE START @(855, 285);           // Initial State

      STATE REQ @(215, 755)                      // Normal execution
        ACTION REQ -> CNF;
      END_STATE

      STATE State @(2015, 430)
        ACTION INIT -> INITO;
      END_STATE

      TRANSITION START -> REQ   ON REQ  @(555, 600);
      TRANSITION REQ   -> START ON TRUE @(215, 425);
      TRANSITION START -> State ON INIT @(1705, 320);
      TRANSITION State -> START ON TRUE @(1585, 680);
    END_ECC

    ALGORITHM INIT : OTHER 'AnyText'
    <<<
    if (SEED() == 0) {
      srand( (unsigned int) time(NULL) );
    } else {
      srand( SEED() );
    }
    >>>
    END_ALGORITHM

    ALGORITHM REQ : ST
      VAL := REAL#0.0;  // 示例:ST 算法直接内联,由 st-algorithm 文法解析
    END_ALGORITHM
  END_BASIC_FB

  ATTRIBUTE TypeHash := '';
END_FUNCTION_BLOCK
```

要点:

- `ACTION <algorithm> -> <output event>` 对应 `<ECAction Algorithm=".." Output=".."/>`,两侧均可省略(`ACTION -> CNF` 或 `ACTION REQ;`)
- `@(x, y)` 为布局注解,语法上归入 `LayoutAnnotation` 规则,可被工具统一忽略
- `ON TRUE` 对应 XML 中的 `Condition="1"`
- 非 ST 语言的算法用 `OTHER '<language>'` + `<<< ... >>>` 原文块承载,由 terminal 规则整体捕获,不参与解析
- ST 算法体由 `st-algorithm.langium` 直接解析,与接口变量在同一 AST 中,作用域解析天然可达

### 5.3 复合功能块与网络

```iecst
FUNCTION_BLOCK MyComposite
  EVENT_INPUT  REQ : Event WITH DI; END_EVENT
  EVENT_OUTPUT CNF : Event WITH DO; END_EVENT
  VAR_INPUT    DI : INT; END_VAR
  VAR_OUTPUT   DO : INT; END_VAR

  COMPOSITE_FB
    NETWORK
      FB ADD1 : E_CTUD @(600.0, 300.0);
      FB SW   : E_SWITCH @(1600.0, 306.7);

      EVENT
        REQ      -> ADD1.CU  @[dx1: 360.0];
        ADD1.CUO -> SW.EI    @[dx1: 300.0];
        SW.EO0   -> CNF;
      END_EVENT

      DATA
        DI       -> ADD1.PV  @[dx1: 313.3];
        ADD1.CV  -> DO;
      END_DATA
    END_NETWORK
  END_COMPOSITE_FB
END_FUNCTION_BLOCK
```

### 5.4 系统配置

```iecst
SYSTEM ExampleSystem

  APPLICATION ExampleFbNetworkApp
    NETWORK
      FB E_SR     : E_SR     @(606.7, 300.0);
      FB E_SWITCH : E_SWITCH @(1600.0, 306.7);
      FB E_CTUD   : E_CTUD   @(2853.3, 240.0);

      EVENT
        E_SR.EO      -> E_SWITCH.EI @[dx1: 360.0];
        E_SWITCH.EO0 -> E_CTUD.CU   @[dx1: 300.0];
        E_SWITCH.EO1 -> E_CTUD.CD   @[dx1: 300.0];
      END_EVENT

      DATA
        E_SR.Q -> E_SWITCH.G @[dx1: 313.3];
      END_DATA
    END_NETWORK
  END_APPLICATION

  DEVICE DEV1 : FORTE_PC @(200, 100)
    PARAMETER MGR_ID := 'localhost:61499';
    RESOURCE RES1 : EMB_RES
      NETWORK
        // 资源内 FB 网络
      END_NETWORK
    END_RESOURCE
  END_DEVICE

  MAPPING
    ExampleFbNetworkApp.E_SR   -> DEV1.RES1;
    ExampleFbNetworkApp.E_CTUD -> DEV1.RES1;
  END_MAPPING

END_SYSTEM
```

### 5.5 布局信息的处理

坐标是**用户有意表达的信息**(工程师通过排布传达结构),不能用自动布局替代。处理策略:

| 方案 | 采纳 | 说明 |
| --- | --- | --- |
| 内联注解 `@(x, y)` | ✅ | 与 XML 现状等价(XML 中 x/y 本就是属性),diff 时坐标变更可见但可通过工具过滤 |
| 独立 sidecar 文件 | ❌ | 造成文件对,增加同步成本与丢失风险 |
| 完全依赖 ELK 自动布局 | ❌ | 丢失用户意图 |

为缓解"移动一个节点污染 diff"的问题,提供:

- `apcsim fmt --strip-layout` 生成无布局的规范化输出,用于语义 diff
- Git `.gitattributes` 可配置的 diff driver,默认隐藏纯布局变更

---

## 6. 持久化与互操作

### 6.1 双格式策略

**首个里程碑:磁盘格式保持 IEC 61499 XML,DSL 文本作为内存表示与可选视图。**

理由:

- 文本语法一旦作为持久化格式发布,即成为对用户可见的接口契约,难以再改
- 保证与 4diac-ide、其他 61499 工具的即时互操作
- 架构收益(单一真相源、Langium 全套能力、图从文本生成)在内存层面即已完全获得

后续里程碑在语法稳定后,向愿意使用 Git 工作流的用户开放 DSL 作为一等持久化格式,两种格式长期共存。

### 6.2 转换层设计

```
┌──────────┐   parse    ┌──────────┐   serialize  ┌──────────┐
│ .fbt XML │ ─────────► │ DSL 文本 │ ───────────► │ .fbt XML │
└──────────┘            └────┬─────┘              └──────────┘
                             │ Langium
                             ▼
                        ┌──────────┐
                        │   AST    │
                        └──────────┘
```

转换层为**纯函数式、无状态**,不参与日常编辑:

- `xmlToDsl(xml: string): string` — 打开旧工程时调用
- `dslToXml(text: string): string` — 保存/导出时调用
- 两者均不依赖 Langium 运行时,可独立测试与在 CLI 中使用

对应 4diac 现有的 `dataimport`(5182 行,20+ 个 importer)与 `dataexport`(2609 行),这部分格式知识必须逐条翻译,是整个项目中**不可省略、不可自动化**的核心工作之一。

### 6.3 无损性验证

仓库自带 1321 个 `.fbt`、24 个 `.adp`、7 个 `.sys`、4 个 `.dtp`、1 个 `.sub`,合计 1357 个文件,构成现成的高覆盖回归语料。

**验证协议(必须在 CI 中强制执行):**

1. **往返一致性**:`XML → DSL → XML`,规范化后逐字节比对
2. **语义一致性**:`XML → AST_a`,`XML → DSL → AST_b`,深度比较 `AST_a` 与 `AST_b`
3. **属性化测试**:对 AST 做随机变异,验证 `serialize → parse` 恒等

任何一项失败即阻断合并。

---

## 7. 语言服务设计

### 7.1 作用域解析

| 场景 | 实现 |
| --- | --- |
| ST 算法引用本 FB 的输入/输出/内部变量 | Langium 默认词法作用域,沿 `$container` 向上查找,**几乎零代码** |
| ST 引用全局常量 | `IndexManager` 跨文档查询 |
| FB 实例引用类型库中的 FB 类型 | `IndexManager` + 类型库索引 |
| 网络连接的端口引用(`E_SR.EO`) | 自定义 `ScopeProvider`:解析实例 → 其类型 → 类型的接口成员 |
| 结构体成员访问(`var.field`) | 自定义 `ScopeProvider`,基于类型推导 |
| 适配器接口(socket/plug) | 自定义 `ScopeProvider`,解析适配器类型的对偶接口 |

**关键收益:** 4diac 现有的 `STCoreScopeProvider`(348 行)+ `STAlgorithmScopeProvider`(88 行)中,用于跨越 EMF 模型边界的反射逻辑(如 `LibraryElementPackage.eINSTANCE.getVarDeclaration().isSuperTypeOf(clazz)`)将完全消失,因为文本与图形模型合并为同一 AST。剩余的自定义作用域集中在端口引用与类型推导,是真正的领域逻辑。

### 7.2 索引与类型库

类型库是大规模只读资源,需要与用户工程区别对待:

```typescript
interface TypeLibraryService {
    // 类型库:预构建索引,懒加载 AST
    loadLibrary(path: string): Promise<LibraryIndex>
    resolveType(qualifiedName: string): Promise<FunctionBlockType | undefined>

    // 用户工程:全量索引,常驻
    indexWorkspace(root: string): Promise<void>
}
```

优化策略:

- 类型库索引**离线预构建并缓存**(仅提取符号表:类型名、接口签名),避免每次启动全量解析
- 类型库 AST **按需加载**,仅在用户打开或需要深度解析(如实例化校验)时构建
- 索引缓存以类型库版本号为键,版本未变则直接复用

### 7.3 校验

Langium `ValidationRegistry` 注册领域校验规则。需从 4diac 移植的规则来源:

| 来源 | 行数 | 内容 |
| --- | --- | --- |
| `model/src/validation` | 565 | 模型级校验 |
| `src-gen/.../LibraryElementValidator.java` | 3,656 | 生成骨架 + 少量人工规则(含 1 处 `@generated NOT`) |
| 各 Xtext 语言的 Validator | — | ST 类型检查、常量求值 |

校验分级:

- `error` — 阻断部署(类型不匹配、未解析引用、连接非法)
- `warning` — 提示(未使用变量、缺失注释)
- `info` — 建议

### 7.4 增量构建与性能

Langium 的性能边界是已知风险(TypeFox 已公布 Go 实现的 Fastbelt 以应对大规模场景,官方数据显示大工作区下可快约 49 倍,并指出 Langium 中 CST 约占框架内存 75%)。

对本项目的分场景评估:

| 维度 | 现状数据 | 风险 | 应对 |
| --- | --- | --- | --- |
| 单文档规模 | 中位 38 行,最大 2805 行 | 低 | 无需处理 |
| 工作区文件数 | 仓库自带 1357 个;真实项目类型库可达上万 | **中高** | 类型库索引缓存 + 按需加载 |
| 最大 `.sys` | 2805 行 | 中 | 必要时拆分设备配置与应用网络为独立文档 |

**强制要求:原型阶段即用最大规模的真实工程做压测,不得推迟到后期。** 若类型库规模导致索引不可接受,预留切换到 Fastbelt(Go)作为索引后端的接口。

---

## 8. 图编辑器设计

### 8.1 选型:Sprotty 为主,GLSP 按需

`langium-sprotty` 提供 `LangiumDiagramGenerator`,图服务端直接内嵌在语言服务中,通过 LSP 的 JSON-RPC 扩展方法与前端通信。这是官方支持的一等集成路径,与文本优先架构天然契合。

GLSP 提供更完整的图**编辑**交互框架(工具面板、直接操作、命令栈),但其标准形态是图形优先(GLSP 服务端持有源模型)。在文本优先架构下,GLSP 需要一个 Langium Model Server 中间层,而该组件目前仅有研究性实现,**不采用**。

**决策:基于 Sprotty 自建编辑交互层,编辑操作经由 Model API 转换为文本编辑。** 若后续 GLSP 官方推出成熟的 Langium 集成,可评估迁移。

### 8.2 数据流

```
用户在图上拖动 / 连线 / 新建
        │
        ▼
Sprotty Action (前端)
        │  JSON-RPC
        ▼
Model API (后端)
  ├─ 读取当前 AST 定位目标节点的 CST 区间
  ├─ 计算文本编辑(TextEdit[])
  └─ 返回 WorkspaceEdit
        │
        ▼
Theia 应用文本编辑到文档
        │
        ▼
Langium DocumentBuilder 增量重建 AST
        │
        ▼
LangiumDiagramGenerator 重新生成 SGraph
        │
        ▼
Sprotty 增量渲染(基于 element id 做 diff)
```

**图永远由文本生成,绝不反向。** 撤销/重做完全由文本编辑栈承担。

### 8.3 Model API 命令清单

```typescript
interface ModelApi {
    // FB 网络
    addFBInstance(doc: DocumentUri, network: NetworkPath,
                  typeRef: string, name: string, pos: Position): WorkspaceEdit
    deleteFBInstance(doc: DocumentUri, instance: InstancePath): WorkspaceEdit
    moveFBInstance(doc: DocumentUri, instance: InstancePath, pos: Position): WorkspaceEdit
    connect(doc: DocumentUri, source: PortRef, target: PortRef,
            kind: 'event' | 'data' | 'adapter'): WorkspaceEdit
    disconnect(doc: DocumentUri, connection: ConnectionRef): WorkspaceEdit

    // 接口
    addEvent(doc: DocumentUri, dir: 'input' | 'output', name: string): WorkspaceEdit
    addVar(doc: DocumentUri, dir: 'input' | 'output' | 'internal',
           name: string, type: string): WorkspaceEdit
    setWithAssociation(doc: DocumentUri, event: string, vars: string[]): WorkspaceEdit

    // ECC
    addState(doc: DocumentUri, name: string, pos: Position): WorkspaceEdit
    addAction(doc: DocumentUri, state: string,
              algorithm?: string, output?: string): WorkspaceEdit
    addTransition(doc: DocumentUri, from: string, to: string,
                  condition: string, pos: Position): WorkspaceEdit

    // 通用
    rename(doc: DocumentUri, element: ElementPath, newName: string): WorkspaceEdit
}
```

每个方法都是**纯函数**:输入当前文档状态与操作意图,输出文本编辑。不持有状态、不产生副作用,因而易于单元测试。

### 8.4 四类图编辑器

| 编辑器 | 对应 4diac 插件 | 视图内容 | 布局 |
| --- | --- | --- | --- |
| FB 网络编辑器 | `application`, `fbtypeeditor.network` | FB 实例、事件/数据/适配器连接、子应用 | 手工坐标 + ELK 辅助整理 |
| ECC 编辑器 | `fbtypeeditor.ecc` | 状态、动作、迁移 | 手工坐标 + ELK `layered` |
| 接口编辑器 | `fbtypeeditor` | 事件/变量/适配器列表与 WITH 关联 | 表格 + 关联连线 |
| 服务序列编辑器 | `fbtypeeditor.servicesequence` | 时序图 | 自动布局(时序图无自由坐标) |

前三者复用同一套 Sprotty 基础设施(节点、端口、边、选择、缩放、对齐吸附),差异在 `DiagramGenerator` 的映射逻辑与视图组件。

### 8.5 选中与导航同步

Langium 的每个 AST 节点携带 CST 区间(`$cstNode`),据此实现:

- 图上选中元素 → 文本编辑器高亮并滚动到对应区间
- 文本光标移动 → 图上对应元素高亮
- 校验诊断 → 同时标注在文本与图上

这一能力在现有 GEF 架构下需要专门维护映射关系,在本架构下是解析的副产品。

---

## 9. 运行时、部署与调试

### 9.1 对应关系

| 4diac 插件 | 行数 | APCSIM 对应包 |
| --- | --- | --- |
| `deployment` | 11,204 | `apcsim-deployment` |
| `deployment.iec61499` | — | `apcsim-deployment/iec61499` |
| `deployment.opcua`(+ Milo SDK) | — | `apcsim-deployment/opcua` |
| `deployment.bootfile` | — | `apcsim-deployment/bootfile` |
| `model.eval`, `model.eval.st` | 13,899 | `apcsim-eval` |
| `fb.interpreter` | 19,933 | `apcsim-eval/interpreter` |
| `export.forte_ng`, `export.forte_lua` | — | `apcsim-export` |
| `fmu` | — | `apcsim-export/fmu` |
| `debug`, `debug.st`, `debug.replaydebugging` | — | `apcsim-debug` (DAP) |
| `deployment.debug` | — | `apcsim-debug/deployment` |

### 9.2 下装

IEC 61499 下装协议是基于 TCP 的文本命令协议,与语言无关,重写为 TypeScript 无技术障碍。OPC UA 侧需替换 Eclipse Milo(Java)为 Node 生态的 OPC UA 客户端库。

```typescript
interface DeploymentExecutor {
    connect(device: DeviceDescriptor): Promise<Connection>
    createResource(res: ResourceSpec): Promise<void>
    createFBInstance(fb: FBSpec): Promise<void>
    createConnection(conn: ConnectionSpec): Promise<void>
    writeParameter(target: string, value: string): Promise<void>
    startResource(name: string): Promise<void>
}
```

设备厂商通过扩展点贡献自定义 `DeploymentExecutor` 实现(见 §10)。

### 9.3 ST 求值引擎

`model.eval` 与 `fb.interpreter` 共约 33,800 行,实现 ST 表达式求值与 FB 执行语义模拟,用于调试与测试。这是纯计算逻辑,重写为 TypeScript 后可同时服务于:

- 桌面/浏览器内的离线仿真
- CI 中的自动化测试
- 调试时的表达式求值(DAP `evaluate` 请求)

需注意 IEC 61131-3 的数值语义(定宽整数溢出、类型提升规则、`TIME` 类型运算)在 JavaScript 中需显式实现,`BigInt` 用于 64 位整型,定点/浮点行为需逐项对齐标准与 FORTE 实现。**此项需专门的一致性测试套件。**

### 9.4 调试

实现标准 DAP 适配器,前端直接复用 Theia 内置调试 UI(断点、变量视图、调用栈、监视表达式),无需自建。

支持两种调试目标:

- **本地仿真**:`apcsim-eval` 解释执行,断点打在 ST 语句与 ECC 状态迁移上
- **远程 FORTE**:通过部署调试协议连接运行中的设备,读写变量、单步事件

回放调试(`debug.replaydebugging`)作为独立能力,基于记录的事件序列驱动 `apcsim-eval`。

### 9.5 在线监视

在线监视需要高频更新(数十 Hz),不适合走 LSP 通道。设计独立的 WebSocket 通道:

```
Backend 监视服务 ──(独立 WS,二进制帧)──► Frontend 监视层
                                              │
                                              ▼
                                    直接叠加渲染在 Sprotty 图上
                                    (不触发文档变更、不触发重解析)
```

**关键约束:监视数据绝不进入文档模型。** 它是纯运行时叠加层,与 AST 完全隔离。

---

## 10. 扩展机制

### 10.1 两层扩展

| 层次 | 机制 | 面向 | 能力 |
| --- | --- | --- | --- |
| 平台层 | **Theia Extension**(编译期,DI 注入) | APCSIM 自身开发 | 无限制,可增删改任何内核行为 |
| 生态层 | **VS Code Extension API**(运行时安装) | 第三方厂商 | 受 API 边界约束,通过自定义贡献点扩展 |

第三方扩展从**自建的 Open VSX 实例**分发,支持离线安装包(`.vsix`)以适配封闭产线环境。

### 10.2 厂商扩展点

对照 4diac 现有的 82 个 `plugin.xml`、639 处扩展点贡献,规划以下贡献点:

```jsonc
// package.json (VS Code extension)
{
  "contributes": {
    "apcsim.deviceProfiles":   [ /* 设备类型描述 */ ],
    "apcsim.deploymentTargets":[ /* 下装协议实现 */ ],
    "apcsim.exporters":        [ /* 代码生成目标 */ ],
    "apcsim.typeLibraries":    [ /* 类型库来源 */ ],
    "apcsim.validators":       [ /* 附加校验规则 */ ],
    "apcsim.diagramDecorators":[ /* 图元装饰器 */ ]
  }
}
```

每个贡献点对应一个 TypeScript 接口与运行时注册表。扩展通过 Theia 提供的扩展宿主与 Backend 服务通信。

### 10.3 品牌与裁剪

Theia 允许完全控制品牌、启动画面、菜单结构与默认布局(1.74 引入的 perspective 机制支持声明式贡献视图布局)。APCSIM IDE 将:

- 移除通用 IDE 特性(终端、Git UI 可选保留、通用语言支持)
- 默认布局围绕类型库树 + 图编辑器 + 属性页 + 问题视图组织
- 提供"工程"、"调试"、"部署"三个 perspective

---

## 11. 工程结构

```
apcsim-ide/
├── packages/
│   ├── apcsim-core/                 # 共享类型、常量、工具函数
│   ├── apcsim-langium/              # 语言服务(核心)
│   │   ├── src/grammar/
│   │   │   ├── st-core.langium
│   │   │   ├── st-algorithm.langium
│   │   │   ├── st-function.langium
│   │   │   ├── global-constants.langium
│   │   │   ├── data-type.langium
│   │   │   ├── fb-type.langium
│   │   │   ├── adapter-type.langium
│   │   │   ├── fb-network.langium
│   │   │   └── system.langium
│   │   ├── src/scoping/             # 端口引用、类型推导作用域
│   │   ├── src/validation/          # 领域校验规则
│   │   ├── src/index/               # 类型库索引与缓存
│   │   ├── src/diagram/             # LangiumDiagramGenerator 实现
│   │   └── src/lsp/                 # 补全、重构、格式化
│   ├── apcsim-model-api/            # 领域命令 → WorkspaceEdit
│   ├── apcsim-iec61499-xml/         # XML ↔ DSL 双向转换
│   ├── apcsim-typelibrary/          # 类型库管理与解析
│   ├── apcsim-eval/                 # ST 求值与 FB 执行语义
│   ├── apcsim-deployment/           # 下装协议
│   ├── apcsim-export/               # 代码生成(FORTE C++/Lua、FMU)
│   ├── apcsim-debug/                # DAP 适配器
│   ├── apcsim-diagram-client/       # Sprotty 前端视图与交互
│   ├── apcsim-theia-frontend/       # Theia 前端扩展:视图、命令、菜单
│   ├── apcsim-theia-backend/        # Theia 后端扩展:服务注册
│   └── apcsim-cli/                  # 无头 CLI
├── applications/
│   ├── electron/                    # 桌面产品
│   └── browser/                     # 浏览器产品
├── extensions/                      # 官方提供的 VS Code 扩展形态功能
├── examples/                        # 示例工程
└── testdata/
    └── legacy-corpus/               # 从 4diac 导入的 1357 个回归语料文件
```

**依赖方向约束(CI 强制检查):**

```
apcsim-theia-* ──► apcsim-diagram-client ──┐
                                            ├──► apcsim-langium ──► apcsim-core
apcsim-cli ──► apcsim-model-api ────────────┘
           ├──► apcsim-iec61499-xml ──► apcsim-core
           ├──► apcsim-eval ──► apcsim-core
           ├──► apcsim-deployment ──► apcsim-core
           └──► apcsim-export ──► apcsim-core
```

`apcsim-core` 及其下游的所有领域包**不得依赖任何 Theia 包**,以保证无头可用。

---

## 12. 技术栈清单

| 组件 | 包名 | 版本(2026-08) | 许可 | 健康度 |
| --- | --- | --- | --- | --- |
| 应用平台 | `@theia/core` | 1.74.1 | EPL-2.0 / GPL-2.0+CE | 月度发布,季度 community release;近一年 1207 提交、78 贡献者 |
| 语言工程 | `langium` | 4.3.1 | MIT | 近一年 99+ 提交 |
| 语言 CLI | `langium-cli` | 4.3.0 | MIT | 同上 |
| 图集成 | `langium-sprotty` | 4.3.0 | MIT | 同上 |
| 图形渲染 | `sprotty` | 1.4.0 | EPL-2.0 | 活跃(2026-08 更新) |
| 自动布局 | `elkjs` / `sprotty-elk` | 0.12.0 / 1.4.0 | EPL-2.0 | 活跃 |
| 表单 | `@jsonforms/core` | 3.8.0 | MIT | 周下载 26 万 |
| LSP | `vscode-languageserver` | 10.1.0 | MIT | 活跃 |
| 语言 | TypeScript | 7.x | Apache-2.0 | — |
| 桌面运行时 | Electron | 随 Theia 1.74(42.x) | MIT | — |

**备选/观察项:**

- `@eclipse-glsp/client` 2.7.0 — 若官方推出成熟 Langium 集成则重新评估
- Fastbelt(Go)— 若 Langium 索引性能不达标,作为索引后端备选

**基线策略:** 跟随 Theia **季度 community release**,而非月度发布。Theia 会随每个 community release 公布兼容的 Sprotty / GLSP / DAP 版本号,以此作为整体升级锚点。

---

## 13. 实施路线

按技术里程碑组织,每个里程碑有明确的可验证出口条件。

### M1 — 语言地基

**范围:** `st-core` + `fb-type` 文法;显式 AST 类型声明;XML ↔ DSL 转换层;回归语料验证。

**出口条件:**
- 1357 个语料文件全部通过 `XML → DSL → XML` 往返逐字节一致
- 语义一致性测试(`AST_a ≡ AST_b`)全部通过
- 用最大真实类型库完成索引性能压测,给出量化结论

**风险前置:** 本阶段即验证 Langium 的性能边界与语法的信息完备性,这是整个方案最大的两个不确定性。

### M2 — 文本编辑体验

**范围:** 语言服务完整化——作用域(端口引用、类型推导、结构体成员)、校验规则、补全、跳转、重命名重构、格式化;`st-algorithm` / `st-function` / `global-constants` / `data-type` 文法。

**出口条件:**
- ST 算法中引用 FB 接口变量的补全与跳转正确
- 跨文件类型引用解析正确
- 从 4diac 移植的校验规则集通过对照测试

### M3 — 图编辑器

**范围:** Sprotty 基础设施;FB 网络编辑器与 ECC 编辑器;Model API 命令集;图文选中同步。

**出口条件:**
- 图上的增删改连全部经由文本编辑落地
- 撤销/重做行为与文本编辑器一致
- 图文双向选中同步正确

### M4 — 工程与类型库

**范围:** 类型库服务;工程视图;接口编辑器;属性页(JSON Forms);搜索;批量编辑。

### M5 — 运行时

**范围:** `apcsim-eval` 求值引擎;FORTE 代码生成;下装(IEC 61499 + OPC UA);DAP 调试;在线监视。

**出口条件:**
- 求值引擎通过 IEC 61131-3 数值语义一致性测试套件
- 生成的 FORTE 代码与 4diac 输出功能等价

### M6 — 扩展生态与产品化

**范围:** 扩展贡献点;Open VSX 实例;品牌化;打包、签名、macOS 公证、自动更新;浏览器形态部署。

### 贯穿性工作

- **回归语料 CI**:从 M1 起持续运行,任何阶段不得破坏往返一致性
- **`@generated NOT` 与 genmodel body 清单核销**:见 §14.3
- **性能基线**:每个里程碑记录索引、解析、图渲染的基准数据

---

## 14. 风险与缓解

### 14.1 Langium 性能上限(中高)

TypeFox 已公开承认 Langium 在大工作区场景的瓶颈,并为此以 Go 实现了 Fastbelt。

- **暴露点:** 类型库规模(真实项目可达上万个 FB 类型)
- **缓解:** 类型库索引离线预构建 + 版本化缓存;AST 按需加载;`apcsim-langium/src/index` 设计为可替换后端
- **验证时机:** M1,不得推迟

### 14.2 文本语法的信息完备性(中)

DSL 必须承载 XML 的全部信息,包括冷僻属性、自定义 `Attribute`、非 ST 语言的算法原文、注释位置。

- **缓解:** M1 的往返逐字节验证是硬性门禁;语料覆盖仓库全部 1357 个文件
- **兜底:** 为无法用语法表达的信息保留 `ATTRIBUTE key := value` 通用承载

### 14.3 隐性逻辑丢失(高,易被忽视)

这是本项目**最容易造成静默功能缺失**的风险点。4diac 有三处领域逻辑不在常规源码目录中:

| 位置 | 数量 | 风险 |
| --- | --- | --- |
| `src-gen` 中的 `@generated NOT` | 全仓 65 处 / 37 文件(model 插件 15 处 / 7 文件) | 混在生成代码中,易随 `src-gen` 一起丢弃 |
| `.genmodel` 的 `body=` 内嵌实现 | 全仓 401 段(model 插件 308 段 / 389 逻辑行) | **不在任何 `.java` 文件中**,扫描源码目录完全看不见 |
| `.ecore` 的 `derived`/`volatile` 特征 | 4 处 derived / 2 处 volatile | 语义隐含在 EMF 生成逻辑中 |

`genmodel` 内嵌实现示例(`VarDeclaration` 的数组尺寸设置):

```java
if (arraySizeString != null && !arraySizeString.isBlank()) {
    if (arraySize == null) { setArraySize(LibraryElementFactory.eINSTANCE.createArraySize()); }
    arraySize.setValue(arraySizeString);
} else if (arraySize != null) {
    arraySize.setValue("");
}
```

**缓解措施(强制):** 项目启动第一件事,编写脚本导出全部三类隐性逻辑,连同所属 EClass / EOperation 名称生成核销清单,逐条标记"需保留 / 已被新设计覆盖 / 可丢弃",纳入版本控制并在每个里程碑复核。

### 14.4 IEC 61131-3 数值语义一致性(中高)

JavaScript 的 `number` 无法直接表达定宽整数溢出、无符号运算、`TIME`/`LTIME` 语义。

- **缓解:** 整型统一用 `BigInt` + 显式位宽掩码;建立与 FORTE 逐项对照的一致性测试套件;此套件在 M1 即开始建设,不等到 M5

### 14.5 Electron 资源占用(低-中)

安装包数百 MB,常驻内存高于原生应用,冷启动为秒级。

- **判定:** 对工程师工作站可接受
- **边界:** 若产品需进入 HMI 面板/嵌入式场景,该场景应另做原生方案,不在本方案范围内

### 14.6 主线追赶(低)

4diac-ide 近两年 2968 次提交,持续演进。APCSIM IDE 作为独立产品不存在合并压力,但需持续跟踪 IEC 61499 标准演进与 4diac 的格式变更。

- **缓解:** 回归语料定期从 4diac 上游同步更新

---

## 15. 附录:4diac-ide 现状实测数据

以下数据由本方案编写时对 4diac-ide 仓库实测得出,作为规模估算与决策依据。

### 15.1 整体规模

| 指标 | 数值 |
| --- | --- |
| 插件数 | 108 |
| `plugin.xml` 数 / 扩展点贡献数 | 82 / 639 |
| Java 文件数 | 3,888 |
| Java 总行数 | 993,911 |
| 生成代码(`src-gen`/`emf-gen`/`xtend-gen`) | 693,560 |
| 手写代码(`src`) | 298,068 |
| `.ecore` / `.genmodel` 文件 | 26 |
| `.xtext` / `.xtend` / `.mwe2` 文件 | 90 |
| 近两年提交数 | 2,968 |

### 15.2 框架耦合(引用该框架的文件数)

| 框架 | 文件数 |
| --- | --- |
| `org.eclipse.emf` | 1,624 |
| `org.eclipse.gef` | 731 |
| `org.eclipse.jface` | 649 |
| `org.eclipse.ui` | 539 |
| `org.eclipse.swt` | 520 |
| `org.eclipse.core.runtime` | 505 |
| `org.eclipse.xtext` | 385 |
| `org.eclipse.core.resources` | 348 |
| `org.eclipse.draw2d` | 316 |
| `org.eclipse.debug` | 125 |
| `org.eclipse.ltk` | 78 |
| `IFile`/`IProject`/`IWorkspaceRoot` | 276 |

### 15.3 代码量前 20 的插件

| 插件 | 行数 |
| --- | --- |
| `model` | 136,179 |
| `structuredtextfunctioneditor.ide` | 68,670 |
| `structuredtextalgorithm.ide` | 68,623 |
| `globalconstantseditor.ide` | 65,904 |
| `structuredtextcore.ide` | 63,270 |
| `structuredtextcore` | 48,381 |
| `structuredtextalgorithm` | 40,001 |
| `structuredtextfunctioneditor` | 38,794 |
| `globalconstantseditor` | 37,888 |
| `application` | 33,853 |
| `structuredtextcore.model` | 33,605 |
| `model.edit` | 32,587 |
| `gef` | 25,476 |
| `contractspec` | 25,249 |
| `fb.interpreter` | 19,933 |
| `contractspec.ide` | 19,412 |
| `model.eval` | 13,899 |
| `model.commands` | 11,956 |
| `deployment` | 11,204 |
| `typemanagement` | 10,761 |

> 四个 `*.ide` 插件合计约 26.6 万行,均为 Xtext 生成的编辑器无关层。Langium 架构下不产生对应物。

### 15.4 `org.eclipse.fordiac.ide.model` 插件构成

| 部分 | 文件数 | 行数 |
| --- | --- | --- |
| `src`(手写) | 212 | 28,600 |
| `src-gen`(生成) | 385 | 107,579 |

`src-gen` 细分:

| 类别 | 文件数 | 行数 | 占比 |
| --- | --- | --- | --- |
| `*Impl.java` | 185 | 55,096 | 51.2% |
| `*Package.java` | 3 | 23,212 | 21.6% |
| 纯接口 | — | 15,260 | 14.2% |
| `*Factory.java` | 6 | 5,319 | 4.9% |
| `*Switch.java` | 3 | 5,036 | 4.7% |
| `*Validator.java` | 1 | 3,656 | 3.4% |

`src` 细分(真实领域逻辑,需逐块重写):

| 子目录 | 行数 | 说明 |
| --- | --- | --- |
| `dataimport` | 5,182 | IEC 61499 XML 解析(20+ importer) |
| `typelibrary` | 4,659 | 类型库管理 |
| `libraryElement` | 3,938 | 模型辅助逻辑 |
| `dataexport` | 2,609 | IEC 61499 XML 序列化 |
| `helpers` | 2,428 | 通用辅助 |
| `value` | 1,511 | 值语义 |
| `errormarker` | 907 | 错误标记 |
| `resource` | 721 | 资源处理 |
| `datatype` | 641 | 数据类型 |
| `validation` | 565 | 校验 |
| 其余 | 1,439 | util / buildpath / annotations / data / preferences / emf / systemconfiguration |

### 15.5 元模型结构

`lib.ecore`(2,925 行):

| 指标 | 数值 |
| --- | --- |
| EClassifier | 133(含 19 抽象、6 接口) |
| eStructuralFeature | 209 |
| eOperation | 259 |
| `containment="true"` | 84 |
| `eOpposite` | 42 |
| `derived="true"` | 4 |
| `volatile="true"` | 2 |
| `transient="true"` | 6 |

全部 14 个 `.ecore` 合计 5,351 行。

### 15.6 隐性逻辑分布

| 位置 | model 插件 | 全仓库 |
| --- | --- | --- |
| `@generated NOT` | 15 处 / 7 文件 | 65 处 / 37 文件 |
| `.genmodel` 内嵌 `body=` | 308 段 / 389 逻辑行 | 401 段 |

`.genmodel` 内嵌实现体长度分布:289 段为单行,最长 9 行。

### 15.7 工程文件规模

| 类型 | 数量 |
| --- | --- |
| `.fbt` | 1,321 |
| `.adp` | 24 |
| `.sys` | 7 |
| `.dtp` | 4 |
| `.sub` | 1 |
| **合计** | **1,357** |

行数分布:中位数 38 行,最大 2,805 行(`CrossingSubappConnectionCommandTest.sys`),全部文件合计约 59,249 行。

### 15.8 作用域解析实现

| 文件 | 行数 |
| --- | --- |
| `STCoreScopeProvider.java` | 348 |
| `STAlgorithmScopeProvider.java` | 88 |
| 其余 7 个 ScopeProvider | — |

`STAlgorithmScopeProvider` 通过 `LibraryElementPackage.eINSTANCE.getVarDeclaration().isSuperTypeOf(clazz)` 等反射调用跨越 Xtext 与 EMF 模型边界——文本优先架构下此类逻辑消失。

---

## 变更记录

| 版本 | 说明 |
| --- | --- |
| v1.0 | 初始草案 |
