# 发型与自定义道具搜索：可行性、证据与 Claude 开发计划

日期：2026-11-02。代码基线：`05c99d244a6f655017f0b710b240b0d6d48103b4`（1.0.11）。本报告不改变版本号。

## 1. 结论与交付边界

**发型搜索可行，实体数据来源已经确认；发型与道具页面导航已有实机证据。指定发型／道具的原生列表定位、试穿预览和更换尚未认证，不能宣称现有服装定位器已经支持它们。**

**交付分工已由用户明确：Claude Agent 在朋友的非 Windows、无游戏环境完成代码、自审、可运行的测试并提交 PR；本机接收 PR 后负责 Windows 构建、游戏验证和最终验收。** 不要求 Claude 安装游戏、搭建 Windows、运行 ImGui 窗口或完成实机认证后才能交付。

Claude 的目标是把发型／自定义道具功能代码尽量完整实现，包括目录、搜索、UI、请求与原生路由／执行接线，不把本任务缩成一份计划或只有接口的占位 PR。现有原生证据与缺口在下文明确列出，供其判断实现；不能验证的假设在 PR 中说明，由本机接续核查，不伪称已实机通过。缺少某项原生协议不是要求 Claude 在非 Windows 机器上完成逆向的硬条件。

本轮核对了生产代码、已提交的研究文档、现有原始静态日志，并定向读取既有 PC 地区定制表。没有游戏进程可供本轮验证，没有启动游戏、注入或新增实机动作。本轮的新增事实是 PC 表数据；导航实机结论来自此前记录。

## 2. UI 与用户需求（以本节为准）

用户已明确纠正：**模型、发型、自定义道具是三个独立类别，不能把饰品塞进发型页。**

顶部领域 Tab：**歌曲｜模型｜发型｜自定义道具**。现有服装领域内部仍可沿用 Module 命名；面向玩家的类别采用“模型”。恢复目前隐藏的 Accessories 入口为独立“自定义道具”，另增“发型”。不是给旧饰品 Tab 改一个标签就算完成。

用户提供的三张原生截图及逐图说明见 [原生自定义页面与 UI 层级](../design/NATIVE_CUSTOMIZATION_UI_REFERENCE.md)。图一为自定义父页，图二为发型子页，图三为自定义道具子页。父页三个相关入口同级；四个饰品部位只属于自定义道具子页。原生父页另外还有 T 恤编辑、按键音设定和游戏与操作设定，本项目无需为它们新增搜索 Tab。

| 页面 | 内容与筛选 | 不应混入 |
| --- | --- | --- |
| 模型 | 现有服装搜索及歌曲推荐模型功能 | 不改坏现有推荐、预览、更换 |
| 发型 | 仅 `parts=KAMI`；人物、来源、名称／别名／ID、排序 | 四部位饰品 |
| 自定义道具 | `ZUJO/FACE/NECK/BACK`；头／脸／胸口／背部、人物、来源、名称／别名／ID、排序 | 发型 |

三个定制领域分别保存查询、筛选、选择、滚动和查询修订。初次人物为全部，重新打开保留玩家筛选。未知部位保留诊断，不自动当成发型或头部。

沿用当前自适应卡片、名称与 ID 信息和键盘导航。发型卡片显示名称、人物、稳定 ID、来源；道具额外显示部位。缺图片时显示明确占位，不能显示错种类图片。无须重做界面框架。

交互要求：

- 同目标原生页内：主体单击／Enter 可以请求原位定位；尚无已认证定位能力时，只改变搜索选择并提示原因。
- **只有鼠标左键双击同一搜索结果主体，才允许跨原生页面。**包括模型→发型、发型→道具、主页／选曲→定制子页；切搜索 Tab 本身不操作游戏。
- 双击锁定 `{实体种类,id,领域,查询修订,目录代数}`；滚动、重排、改查询、失焦时清空首击。按钮区域不能同时命中主体双击。
- 悬浮或键盘聚焦卡片时可以显示“更换”“预览”操作条。悬浮本身不执行动作；按钮只操作所在卡片，不依赖原生高亮或另一搜索选中项。
- 预览／更换按钮不能获得跨页权限。未到正确原生页、人物／槽位／范围／拥有状态未知、原生动作未认证时禁用并说明原因。不能照搬模型命令。
- 搜索选中、原生高亮、临时预览、实际装备分别表示；页面到达不等于定位成功，更不等于已装备。
- 保留现有歌曲难度按钮单击跨页这一独立规则，不把新增定制领域的规则倒灌修改歌曲行为。

既有需求参考：[卡片及双击规范](../design/CUSTOMIZATION_ACTIONS_SPEC.md)、[动作实现计划](CUSTOMIZATION_ACTION_IMPLEMENTATION_PLAN.md)。其中“模型页”专属协议不能直接用于发型／道具；新领域使用本报告的能力边界。

## 3. 数据证据：PC 本体确实含发型

本轮只展开既有定向提取的一份 `rom_steam_region/rom/gm_customize_item_tbl.farc`，读取其中 UTF-8 文本 `gm_customize_item_id.bin`，没有全量导出。

| PC 基础地区表事实 | 结果 |
| --- | --- |
| `cstm_item.data_list.length` | 588 |
| `KAMI` 发型 | 384 |
| `ZUJO` 头部 | 95 |
| `FACE` 脸部 | 52 |
| `NECK` 胸颈 | 25 |
| `BACK` 背部 | 32 |
| 原 FARC SHA-256 | `0F9C6EC359021BF430E71E68DE6F79834F42E69B900A584F07660AC4A69106D0` |
| 内部文本 SHA-256 | `47BCC62CBBD76600D7F526AE7ACD0AFB59BC61E318CA1F2061C855A9D5D95F9A` |

发型人物分布：MIKU 170、RIN 45、LEN 37、LUKA 42、NERU 4、HAKU 4、KAITO 40、MEIKO 34、SAKINE 5、TETO 3，合计384。此为一份 PC 表的记录数，**不是安装 DLC／Mod 后最终有效目录，也不是可装备数量或已拥有数量**。

最小事实样本（仅摘录必要字段）：

```ini
cstm_item.204.bind_module=0
cstm_item.204.chara=MIKU
cstm_item.204.id=500
cstm_item.204.name=初音ミクヘア
cstm_item.204.obj_id=500
cstm_item.204.parts=KAMI
cstm_item.204.sort_index=100
```

**表内序号204、稳定 ID 500、对象 ID、模型绑定值是不同字段。** `bind_module=0`应保留原值，不解释为“未绑定”，也不能仅凭此推断游戏中的发型兼容性、推荐关系或装备协议。

旧 PS4 表只有496条（KAMI335、头77、脸47、胸颈12、背25），不可拿它替代 PC 有效目录。本轮 PC 头95／脸52与此前原生饰品页观察数量相符，但数量相等仍不证明 ID→列表记录映射。

生产读取应复用 `ReadOfficialCpkEntries` 和 `ReadFarcEntryMemory`，按后缀读取官方 `gm_customize_item_tbl.farc` 及内部 `gm_customize_item_id.bin`；保留 CPK、内部路径、逻辑路径和指纹。遍历当前基础／DLC来源，不硬编码上述数量，不要求玩家先导出文件。

Mod 现有扫描已支持 `gm_customize_item_tbl.farc`、`mod_gm_customize_item_tbl.farc`。继续遵守 DML enabled／priority／include 及逻辑路径覆盖规则：普通表整体替换，追加表按稳定 ID 覆盖；名称表不能凭空创建实体。名称沿用 `customize.<id>`，兼容 `cstm_item.<id>`，并保留多语言、别名和来源贡献。

**重要缺口：当前 `ReadOfficialModuleTables` 只读官方模型表；Mod 道具表可读，不等于官方发型／道具目录已经自动加载。** 必须补官方定制表入口。

## 4. 数据领域与检索设计

建议保留原始实体命名空间 `CustomizationKind::Module / Accessory`：发型和四部位道具均来自 `cstm_item`，共用 CustomizeItem ID 空间。另增 `CustomizationPart::Hair`（映射 KAMI）及独立搜索领域 `Models / Hair / Accessories`。发型领域条件为 `kind=Accessory && part=Hair`，道具领域条件为 `kind=Accessory && part∈{Head,Face,Neck,Back}`。

这样可保留现有 `customize.<id>` 名称合并和实体去重；不要单纯增加 Hair kind 后让同一 ID 在合并／名称／纹理缓存中产生两份身份。若选择重构 Kind，必须同步转换所有合并键、名称绑定、导入、序列化及测试，不能只改枚举。

搜索结果主键仍为实体键；UI 状态另外带领域。请求中补足部位／目标原生页，由执行器根据最新目录重新核对，不能相信 UI 任意传入的目标页。原始人物与显示分类分开，不能不加判断地对发型使用模型专用 `ModuleSearchCharacter` 分类。

复用 `PrepareCustomizationItem`、`SearchCustomization`、`Normalize` 的英文大小写／全半角／平片假名、多词检索、来源过滤和排序。当前定制检索键没有歌曲那样完整的读音罗马音链路；需要给可靠假名／显式 reading 或 alias 增加 `Romanize` 检索键，汉字读音和中文译名由真实多语言名称／别名提供，不自动猜。人物、领域、部位、来源与文字条件应同时生效。

模型的“搜歌曲→推荐模型”功能继续保留。本轮没有证明“歌曲→推荐发型”关系；不要把模型推荐 ID 当成发型 ID。

## 5. 已确认导航：子页与外层分别处理

以下 RVA 只对应已验证的 EXE SHA-256：
`6F4DF714C28515C93B3023E7D6DAFA9095DD8835F762D54DB1726FB8BC04504D`。
使用动态主模块基址加 RVA，禁止抄历史堆地址；构建不匹配即关闭原生能力。

### 5.1 当前页面与同层跳转

权威生产入口：`src/native_page_router.cpp::ReadNativePage`。自定义任务为 `base+0xCC6F100` 本体，控制器为其 `+0x78` 指针，管理器为 `*(controller+0x998)`。管理器虚表 `base+0xCC4E38`；验证场景树成员后读当前节点 `manager+0x18`，节点 `+0x20` 是页键。

| 页 | 键 | 场景虚表 RVA | 内部状态机指针偏移 | 面板指针偏移 | 完成字节 |
| --- | --- | --- | --- | --- | --- |
| 自定义父页 | 1 | `0xCC5170` | `+0x178` | 不适用 | `+0x19` |
| 模型 | 2 | `0xCC4BB0` | `+0x330` | `+0x310` | `+0x25` |
| 发型 | 3 | `0xCC4678` | `+0x3A0` | `+0x380` | `+0x25` |
| 自定义道具 | 4 | `0xCC48C8` | `+0x300` | `+0x2E0` | `+0x29` |

场景对象同时存在，不能靠扫描到对象来判断当前页。空闲要求目标节点 `manager+0x20` 等于树头、过渡 `manager+0x28==1`、该页内部状态为0，并结合弹窗、配置和槽位等守卫。

已验证原语：在匹配真实内部状态帧 `0x66B2D0` 的帧末，一次性调用 `0x6968D0(manager,targetKey)`；受理后用 `0x682530(panel)` 隐藏**来源页**面板，设置**来源页**完成字节为1，等待原生切换及独立读回。不是直接写当前场景节点。复用 `scripts/research/route_actions.js::routeArmNavigation`；已实机验证模型→发型→道具→模型及子页返回父页。

当前生产 `InvokeRoute` 只把发型／道具来源导向模型，`ModelArrival` 和提交网关也锁定 Module；**必须泛化目标页和到达后分发，不能仅添加 Tab**。

### 5.2 主页／选曲进入与退出

| 起点 | 已验证协议 | 完成判据 |
| --- | --- | --- |
| 主页 | 匹配外层控制帧 `0x2C4A40` 帧末调用 `0x2C4BB0(7)` | 真实自定义父页1、来源为主页模式7 |
| 选曲 | 匹配选曲帧 `0x6EDC40`，先保存有效 PVID，再 `0x1DE750(pvid,null)`，随后 `0x6EECD0(song,1)` | 真实父页1、同一来源歌曲 |
| 父页→模型 | 父页光标0、phase1且空闲；`0x69B040(scene)`，父页 `+0x1C=2` | 原生页键2稳定 |
| 子页→父页 | 同层协议 target1 | 页键1、父页稳定 |
| 父页→原来源 | 请求管理器键8 Finish；受理后 `controller+0x28=1`、父页 `+0x19=1` | 来源为5回选曲，为36回主页，等待原生保存／收尾 |
| 主页→选曲 | 外层帧末 `0x2C4BB0(2)` | 真实选曲页可用 |

主页外层稳定为 mode9、sub42/43；主页来源自定义为 mode7、sub36/37；选曲来源自定义外层仍为 mode2、sub5/6。必须以内层页键区分，不能把 mode7当作所有自定义。

父页直接进入发型／道具的自然光标语义已知（0模型、1发型、2道具），但主动调用完整协议尚未独立验收。开发首版采用已验证的“父页光标0→模型→目标兄弟页”。**父页光标不是0时应明确拒绝；当前证据不足以承诺任意父页光标都自动进入。** 后续另证实父页光标准备／直接进入协议，不能后台硬写光标或发键绕过双击权限。

完整14阶段原生脚本曾通过：主页→父页→模型→发型→道具→模型→父页→主页→选曲→父页→模型→定位316→父页→选曲。它证明页面导航及模型定位，不证明发型／道具 ID 定位。

来源：[最新外层实机报告](research/OUTER_NATIVE_ROUTING_VERIFIED.md)、[当前原生路由研究](research/CURRENT_NATIVE_ROUTING_RESEARCH.md)、[场景映射](research/PAGE_STATE_MAPPING.md)。最后一份较早文档开头的“父页进入未解决”已被后续报告和生产代码部分取代，应按日期及实际协议阅读。

## 6. 原生目标定位与动作：还差什么

### 6.1 发型

已知 SceneHairstyle 虚表、构造入口 `0x688040`、状态机与导航完成字段；尚无可信的发型列表容器、ID→record 映射、高亮更新入口、角色分类更新协议、预览／更换回调及装备读回字段。**模型的 `scene+0x60/+0x68`、分类函数 `0x694D00`、动作回调 `0x695250` 和 command3／7不能复制过去。**

首个研究目标是自然左右移动和换人物时，取得发型真实记录的稳定 ID 与列表身份，而不是先调用猜测函数。导航代码现在可以写；指定 ID 的原生执行器必须通过证据 Profile 开启。

### 6.2 道具：已有记录可直接接续

来源：[饰品两轮只读观察](research/ITEM_SELECTION_OBSERVATION.md)。页面截图、自然移动及字段序列已对照，但没有主动调用下列道具候选。

| 道具 scene 字段 | 实测事实 | 尚不能声称 |
| --- | --- | --- |
| `+0x30` | 头0→脸1→头0 | 胸／背枚举已实机认证 |
| `+0x78` | 右移0→1；换部位归0；头部左环绕0→94→右回0 | 它就是稳定 ID，或直接写值即可定位 |
| `+0x84` | 头95、脸52 | 容器边界与全部记录布局已认证 |
| `+0x50` | 随部位改变的指针 | 已解出可用记录容器 |
| `+0x108` | 瞬态变化 | 稳定目标 ID |

有限静态候选：`0x68C700(scene,direction)`左右（-1/+1）、`0x68EF00(scene,direction)`部位切换（-1/+1）、`0x68C780(scene)`确认分支、`0x68C7D0(scene)`未知分支、`0x68DB10`／`0x68E7C0`辅助。初始化 `0x68C180` 绑定回调对象 `scene+0x2D8`，其中+8绑定scene、+0x10指向 `0x68F430`；输入分支 `0x68FF10`。**这些是研究入口，不是已认证可调用 ABI；回调命令未确认。**

### 6.3 后端契约与开放条件

建议新增 `customization_item_locator.*`，发型／道具共享事务框架，分别持有页 Profile；避免重写成熟模型定位器。接口名称可调整，但职责必须明确：

1. 只读快照：页、实体领域／部位、人物／槽位／范围、列表身份、稳定高亮 ID、真实装备 ID、加载状态、contextGeneration。
2. 请求：沿用 `CustomizationLocateRequest` 的 key、目录指纹、UI代数、查询修订、serial、origin、首击身份、scope和slot，补领域／部位；网关独立重算目标页。
3. 执行器：`Submit / Tick或OnNativeFrame / Poll / Cancel / Capabilities`；未知Profile明确 `Unsupported`，不能只写缓存并返回成功。
4. 列表就绪后由稳定 ID 唯一查找 record；跨分类先走已认证分类协议，刷新后重取容器，禁止跨页持有旧 record。
5. Locate完成必须跨帧读回真实高亮ID；Preview／Equip分别读取临时显示、原装备与实际装备，不以请求ID回显冒充成功。
6. 单事务串行，页面、来源歌曲、槽位、scope、目录／查询变化或超时即取消未执行步骤；帧回调不 Sleep、不扫描全目录、不阻塞磁盘。

可以先实现 Profile 接口与模拟后端，但生产未认证字段留空，不能从模型填默认值。导航、列表就绪、素材就绪、目标高亮是独立阶段。若目标定位不受支持，应在开始导航前说明缺口，不展示“已定位”。

## 7. 明确开发入口与建议改动

以下入口均以本报告基线为准；[固定代码快照](https://github.com/HYDlink/DivaInGameSearch/tree/05c99d244a6f655017f0b710b240b0d6d48103b4)可供异机阅读。

| 文件／符号 | 当前情况 | 本次入口 |
| --- | --- | --- |
| `src/ui.h::SearchMode`、`src/ui.cpp::SetMode/Draw` | Songs/Modules/Accessories，实际只画歌曲／服装 | 增 Hair；顶部四个独立Tab，独立道具入口 |
| `src/customization.h`、`customization.cpp` | 仅四部位；KAMI保留为Unknown | 增Hair部位和独立查询领域，严格分流；补假名罗马音键 |
| `src/customization_catalog.cpp::ReadCustomizationStream/LoadCustomizationCatalog` | 保留rawFields、objectId、bindModule；官方只加载Module，Mod可读cstm | 补官方定制表；按同一cstm ID空间合并后再按领域投影 |
| `src/module_assets.h/.cpp::ReadOfficialCpkEntries`、`src/farc.*` | 已有CPK单条读取、FARC内存读取 | 复用容器读取，不要求外部服务或全量导出 |
| `src/customization_ui.h::CustomizationView` | `states[2]`、`sourceAssociationCounts[2]`；mode靠Module/Accessory二分 | 改三个定制领域状态；检查所有三元表达式、异步reply及缓存键 |
| `src/customization_ui.cpp::SubmitPendingLocate/RequestLocate/RequestAction/Draw` | 非Module明确拒绝定位；图片和动作也只开Module | 分领域目标页／能力分发；保留主体与按钮互斥命中 |
| `src/customization_action.h::CheckCustomizationNavigation` | 导航权限已有双击守卫，文案为模型 | 泛化目标页文案，权限不放宽 |
| `src/native_page_router.*::SubmitCustomizationRequest/InvokeRoute/OnBoundary/ReadNativePage` | 能读Hair/Accessory；目标及到达分发锁定Module | 目标页参数化、子页Profile、到达后分发；复用已安装Hook |
| `src/mod.cpp`、`src/host.cpp::SyncModuleAssets` | 注入现有定位／动作／图片回调 | 在统一后端接线，UI不直接调用游戏地址 |
| `src/module_locator.*`、`src/module_action.*` | 已认证模型协议 | 只参考事务、校验和取消，不照搬布局及命令 |
| `src/preview.cpp` | 有 `--customization-demo`、`--accessories` | 增三领域合成样本与能力模拟，无游戏可操作预览 |
| `tests/customization_check.cpp`、`tests/module_catalog_check.cpp`、`tests/native_page_router_check.cpp` | 已有解析及隔离页读取测试 | 扩展或新增同类测试，不把隔离测试叫实机通过 |

旧 `TryReadCustomizationContext/GetCustomizationCapabilities` 仍是stub，但生产模型走独立定位／动作能力，不要因此误判模型未实现，也别把旧stub作为新领域唯一能力入口。

缩略图是独立增量：当前 `ModuleAssets` 只读模型 `SPR_SEL_MD...`。道具公开参考有 `SPR_CMNITM_THMB%03d_ITM_IMG[_FT]`，应按实际生效 sprite db查集合和条目，再走现有后台解码／缓存／DX11上传。**本轮未认证发型的图片映射，不能用发型ID套模型图规则，也不能把3D预览与缩略图混为一谈。** 首版允许占位。公开参考：[FTC Mod.cpp](https://github.com/ActualMandM/DivaDllMods/blob/a3fb8ba380063966febfed942324e73c4e9439ab/Source/FutureToneCustomization/Mod.cpp)、[Armoire](https://github.com/jaykthx/Armoire/tree/8acdfada755323afdab03624d9dd3f35828ee222)。这些是已有研究引用，不是本轮对全部发型资源的验证。

数据格式参考：[ReDIVA customize_item_table.cpp](https://github.com/korenkonder/ReDIVA/blob/207c270472ba3f0edcda29bae777e5bd4a8952d9/src/CRE/customize_item_table.cpp)。ReDIVA该版本只列四饰品部位，不能据此否定PC表中的KAMI，也不能把开源重建工程结构大小当商业PC版内存布局。

## 8. Claude 实施顺序与双方交付分工

下面的测试条目是设计参考和可执行时的自测建议，**不是要求 Claude 在非 Windows 环境逐项跑通的完成门槛**。Claude 在现有环境能运行哪些就运行哪些；不能执行的 Windows／ImGui／游戏项在 PR 中说明即可。无需为满足本任务额外搭 Windows 虚拟机或 Wine。

### A. 数据与三个独立定制领域

- 新增Hair部位／查询领域，补官方定制表读取及现有名称来源合并。
- 构造可提交的**人工合成**UTF-8 fixture：不同人物的KAMI，ALL头部，脸／胸颈／背部，未知token、重复ID、名称孤立、普通表替换／追加覆盖、不同种类相同数字ID。放 `tests/fixtures/`，不要放被忽略的 `tests/runtime/` 或依赖F盘。
- 验证KAMI只进发型、四部位只进道具、模型ID与cstm ID不冲突；表内序号不能当ID，id/objectId/bindModule不同值也正确保留。
- 多语言、别名、假名、明确罗马音来源、ID、人物、部位、来源筛选联合检索；无汉字读音时不捏造。

### B. UI、请求和后端隔离

- 顶部歌曲／模型／发型／自定义道具四Tab；三个定制状态独立，输入法候选确认不触发定位，切Tab不切游戏页。
- 合成Preview覆盖长名、空结果、缺图、多来源、不同人物与部位；占位图片不冒充原生预览。
- 注入Fake后端记录调用：跨页普通单击、Enter、底栏、悬浮、更换／预览按钮均不得发导航；双击同主体一次；按钮双击不冒泡；排序／滚动后旧双击失效。
- 保留模型悬浮动作、预览后续接和歌曲推荐已有行为。

### C. 泛化导航与执行器骨架

- 用目标页取代ModelArrival硬编码；请求网关校验领域、部位与最新实体一致。
- 隔离模拟主页→父页→模型→发型、模型→道具、发型→模型、子页→父页，以及父页非0光标拒绝；来源配置、槽位、scope、线程、目录变化／超时取消。
- 扩展现有 `NativePageRouterCheck` 模拟内存夹具与原生调用替身，断言调用顺序和次数，避免测试实际跳入生产RVA。
- 新定位器未知Profile默认Unsupported；测试伪造成功结果、陈旧record、目标缺失、重复ID时不能误报成功或写入。

### D. 本机接收 PR 后验证

Claude 完成实现、自审和适用测试后即可提交 PR。由用户或本机 Agent 拉取 PR，完成 Windows 构建、共用预览和游戏内验证；遇到确实未认证的原生协议，再按下一节补证／调整 Profile。实机通过与最终合并验收由本机负责，不要求 Claude 提供游戏截图、Windows 构建日志或硬件运行结果。

以下是**本机验证方**的 Windows x64 依赖和命令，不是 Claude 的环境前置：PowerShell7、CMake3.24+、完整MSVC与Windows SDK、.NET SDK8及NativeAOT依赖；源码依赖在 `third_party/vendor/`，详见 [依赖锁定](DEPENDENCY_LOCK.md)。Claude 可以在非 Windows 环境做源码审查、差异检查、可移植测试或其现有工具支持的检查，无须保证原 Windows 工程在其系统完整构建。

```powershell
./scripts/build.ps1
ctest --test-dir build/release -C Release --output-on-failure
./build/release/SearchPreview.exe --customization-demo
```

以上为本机的现有命令；Preview目前还没有完整发型演示，须在B阶段补。`verify.ps1 -GameDirectory`、真实资源审计与游戏注入同样由本机负责。本轮仅改文档，不声称重新构建或跑过所有测试。

### Claude 的 PR 交付标准

1. 提交功能实现代码，按本报告明确的四 Tab 与独立页面需求完成接线；不要求先跑本机验收再开 PR。
2. 自行审查变更，运行其环境可执行且与变更相关的测试／检查。
3. 推送分支并创建 PR，简述改动、已做检查、因环境未运行的项目，以及需要本机确认的原生假设／缺口。

Windows 全量构建、实机跳转／定位／预览／更换、截图和最终功能验收属于接收方后续工作，不作为 Claude 完成 PR 的阻塞项。仍应如实区分“代码已实现”和“实机已验证”。

## 9. 剩余实机研究的最小交接格式

无需从零全局扫内存，复用 `scripts/research/run_home_native_route.py`、`route_actions.js`、`run_item_observation.py` 与已验证页面读取，先只读自然操作，再主动调用已证实协议。自然按键沿用AutoHotkey。

| 阶段 | 操作与必须交付的证据 |
| --- | --- |
| 发型列表 | 固定人物／槽位／范围，记录基线、左右各一、环绕、切人物、返回；同时给截图、当前高亮名称和稳定ID，认证容器begin/end/步长、记录ID字段及边界 |
| 道具列表 | 补胸／背；验证+0x50是否真容器、+0x84边界及ID记录；同ID数值与显示序号明确区分 |
| 自然调用 | 有界观察左右／分类及确认分支，记录调用签名、参数、对象绑定、线程、调用前后字段；发型从真实SceneHairstyle虚表／内部状态分支找起，道具从第6节候选找起 |
| 主动定位 | 先两个已知ID，再跨人物／跨部位；每步帧末执行并跨帧读回高亮ID；失败即停，不用连续盲发或只写索引 |
| 预览 | 记录进入／退出命令、是否依赖高亮、临时模型及原装备独立读回；退出应恢复显示且不意外装备 |
| 更换 | 明确拥有状态、可操作scope／slot及实际装备字段；不触发购买，选未拥有项必须拒绝；保存影响单独记录 |
| 导航验收 | 分开测子页互跳、主页来源、选曲来源；普通单击／按钮不能跨页，双击到达后还要核对指定ID |

给Claude的 Profile 包必须是仓库可读的精简资料：构建哈希、RVA与函数签名、结构偏移和字段类型、边界／不变量、自然调用样例、成功及拒绝时的字段序列表、对应截图说明、执行前置和失败行为。原始全量游戏文件与完整内存dump不入仓库；摘录必要语义，不只给一个本机artifacts路径。

本机最终验收分别记录：数据可搜、UI可用、页面到达、目标定位、临时预览、更换装备，不用一项成功替代另一项。这是本机验收分类，不是 Claude 提交 PR 前必须逐项完成的实验清单。

## 10. 朋友开工时的最短阅读路径

1. `AGENTS.md`、`README.md`、本报告及第2节最新UI需求。
2. `src/ui.*` → `customization_ui.*` → `customization_catalog.cpp`／`customization.cpp`。
3. `customization_action.h` → `native_page_router.cpp` → `module_locator.*`／`module_action.*`（后两者只参考通用框架）。
4. `tests/native_page_router_check.cpp` 和 `tests/customization_check.cpp`，先建立人工fixture再改领域。
5. 完成功能代码、自审和环境可运行的检查后推送 PR；实现可以按A、B、C分提交，列明已测和本机待验项，不增版本、不部署、不改其他Mod。

现有资料索引：[整体定制计划](CUSTOMIZATION_SEARCH_PLAN.md)、[当前状态](STATUS.md)、[数据导入约定](CUSTOMIZATION_IMPORT.md)、[依赖与来源](SOURCES.md)。旧计划里“不纳入KAMI”和“饰品隐藏”的阶段范围已由本报告扩展，不能继续用旧阶段范围阻止实现独立发型页。
