源码组成大纲（架构）
1.1 单文件结构
index.html（单文件 SPA，零依赖）
├── 1–1011 行   HTML 结构 + 内联 CSS（青玉配色 v2.1）
│               ├── 面板卡片（属性/功法/行囊/家室等，#pLooks 相貌、#fLove 婚配）
│               ├── 设置弹窗（密钥/模型/文风/自由度）
│               └── 故事区 #story / 选项区 / 自由输入框
├── 1012–6610 行  <script> ① 核心引擎
├── 6613–8121 行  <script> ② 斗法引擎 + 对话
└── 8124–10143 行 <script> ③ 渲染 / 回合循环 / 存档 / 动作入口

1.2 三个 script 块分工
块
行号范围
职责
代表函数/常量
① 核心引擎
1012–6610
配置键、工具函数、状态壳与存档迁移、头像/画像系统、纪年历法、寿命成长、NPC 年轮、修士榜、开局底子、提示词体系、模型调用 callLLM、关系/家庭系统、功法/物品/法器系统、品阶上限、回合结算 applyTurn
migrate / rollXianBase / EGGS / relSet / spouseOf / normArt / capGrade / applyTurn / styleSystem / turnPrompt / callLLM
② 斗法+对话
6613–8121
斗法全流程（组队→回合→结算）、战斗数值、对话快照
startDuel / duelRound / duelNext / duelFinish / renderDuel / adDmg / snapshotConvo / restoreConvo
③ 渲染/回合/存档/入口
8124–10143
面板与 NPC 详情渲染、章节流、骰子条、动作入口（自由输入分类→判定）、回合主循环、存档读档、IndexedDB、输入解析器（新增）
renderPanel / npcDetail / act / classifyAction / judgeFromClass / makeJudge / runTurn / saveGame / loadGame / parseDirectArts / parseDirectItems

1.3 核心模块细分（小项目级）
① 基础与状态
工具函数：plain（清洗）/ num / clamp / asArr / cnToNum（中文数字）
状态壳：newStateShell 建全量状态；migrate 存档迁移（版本 v2→v12，逐版补字段）
纪年：干支年 ganzhiYear、历名 calName、dateOf/dateStr（架空历法，开局随机）
② 人物体系
属性五维：悟性/体魄/神识/心性/谈吐（开局引擎掷定 rollXianBase）
灵根：rollLinggen / lgMake（天/地/单/双/三/四/五灵根、变异、残缺）
境界：parseRealm / newRealm（凡人→炼气→筑基→金丹→元婴→化神→合体→渡劫→大乘）
标签：rollTags + 标签库 XTAGS（悟性通玄/根骨奇佳/剑心通明/心魔深重…）
同名彩蛋 EGGS：玩家填 20 个特定名字（叶凡/韩立/萧炎…）触发专属底子与命格
③ 关系与家庭
关系表 rels：relSet / relOf / relOther；类型 REL_TYPES（相识/好友/仇人/恋人/夫妻/同门/亲属/师徒）
婚恋：spouseOf / loveSheet / isMarried；正室一位、后进为妾；亲属辈分禁婚恋
家庭：父母/子女/继亲/姻亲判定、家室汇总、道侣同修灵力加成
④ 功法 / 物品 / 法器
功法：ART_STYLES 五路数（攻伐/护体/诡术/神通/遁术）、normArt 规范化、artKnown 查重（同名/包含/相似≥0.7）、熟练度
品阶：GRADES 12 档（黄阶下品→天阶上品）、gradeIndex/gradeText、gradeCapFor 按境界压档（炼气≤黄阶上品…渡劫≥天阶）
物品：STACK_CATS（丹药/符箓/毒药/灵材可堆叠）、splitItemCount 数量拆分（×3/三颗/三颗回春丹）、addItems / takeItems / normBag / stackItems
法器：normWeapon（bonus 0–20 按当世水位放大）、guessWeaponBonus 兜底
⑤ 提示词体系（两层）
固定系统提示词 styleSystem()：文风说明、禁用词、范例、温度、母题权重（按文风切换）
每回合动态拼装 turnPrompt()：worldRules（世界规则）→ stateBlocks（主角/在场 NPC/行囊/关系快照）→ 玩家行动 → judgeBlock（引擎判定结果，模型不得篡改）→ 写作要求 → OPTIONS_RULE（选项规则）→ WORLD_STATE_RULES → JSON_RULE + 输出模板
其他提示词：开局 initSchemaPrompt、铸造世界 worldPrompt、斗法续写 duelAftermathPrompt、对话 convoPrompt、历史压缩 volumePrompt、立传 bioPrompt、行动分类 classifyPrompt
⑥ 模型调用
callLLM：fetch POST + SSE 流式读取；response_format:{type:'json_object'}；默认 DeepSeek（base=https://api.deepseek.com，model 自动映射）；超时 90s/240s，失败自动重试；think 模式加 token
⑦ 回合与判定
动作入口 act：选项点击 / 自由输入两条路
自由输入：classifyAction（LLM 低温分类 + 关键词兜底 heuristicClassify，15 类）→ judgeFromClass → makeJudge → rollCheck
判定公式：有效值 = 属性 + 自由度加成(随心+25/传奇+8/写实0) + 标签修正 − 饥寒；骰面修正 = round((骰-10.5)×4)；成功 = 骰 20 或（非 1 且 有效值+修正 ≥ max(20, 难度档+难度修正)）；难度四档 40/55/70/85，修正 −8/0/+8
回合结算 applyTurn：把模型写的 playerChanges（五维/功法/物品/灵石/贡献/灵力/状态/关系）全部入账并套上限
⑧ 存档
saveGame（localStorage + IndexedDB 双写）、loadGame → migrate → showGame；多槽位 xian_saves；章回全本 xian_book（断线可恢复）
1.4 数值上限速查（原版，均被修改版部分放开）
项目
原版上限
修改版（自由输入）
功法熟练度/回合
min(写入, 4×月数)，大吉×2
放开（总上限仍 100）
新学功法熟练度
≤30
放开
五维属性/回合
±3
放开
宗门贡献/回合
≤20
放开
灵石横财
按境界/月数封顶
放开
奇遇灵力
500/1500 分档
放开（封顶本级所需）
功法/物品品阶
按境界压档（炼气≤黄阶上品…）
放开（写什么入什么）
熟练度/等级总上限
100
保留（世界观"登峰造极"）


修改内容明细（共 6 个版本，27 处改动）
V1 · 自由输入必成 + 绕过单回合上限（12 处）
新增总开关（1021–1022 行）
const FREE_INPUT_OVERRIDE=true;                     // false 即还原原版
function isFreeInput(judge){ return !!(FREE_INPUT_OVERRIDE&&judge&&judge.freeInput); }

判定必成（4 处）
位置
改动
rollCheck
增加 free 参数：success = free || freeAct || 骰20 || (骰≠1 && 总值≥难度)；骰 1 仅在非自由输入时判大失败
makeJudge
自由输入（无选项）时给 judge 打 freeInput:true 标记
judgeBlock 提示词
追加"这是玩家自由输入的行动：必须办成、照玩家写的拿到结果，不许打折、不许找补"
判定骰子条
自由输入命中显示"必成"标识（c.free）

上限绕过（6 处，applyTurn 内）——全部加 !isFreeInput(judge) 条件
行号
上限
改动
6245
五维 ±3
自由输入不截断
6273
灵石横财上限
不截断
6283
宗门贡献 ≤20
不截断
6319
新学功法熟练度 ≤30
不截断
6346
功法熟练度 4×月数
不截断
6349 附近
奇遇灵力 500/1500
自由输入按 1e18 不截断（封顶本级所需）

V2 · 品阶上限放开（2 处）
行号
位置
改动
6318
功法 capGrade(p,na,'功法')
自由输入跳过压档，写什么品阶入什么品阶
6377
物品（法器/玉简）capGrade(p,item,cat)
同上

V3 · 输入功法解析器（引擎直接入册，不赌模型）
新增（9268–9292 行）：
DIRECT_ART_VERBS：40+ 触发动词（得到/学会/捡到/抢到/偷来/参悟/掉落/继承…）
DIRECT_ART_LEARN / DIRECT_ART_HINT / DIRECT_ART_NAME_END：功法信号
parseDirectArts(action)：从输入提取《功法名》（可多门）+ 路数（五选一）+ 品阶（黄/玄/地/天阶±下中上）+ 熟练度（0–100，缺省 50）
接入：
makeJudge：自由输入时解析结果挂到 judge.directArts
applyTurn（6300–6316 行）：directArts 先入册（走 normArt→artKnown 查重→品阶），模型本回合写的同名功法被 artKnown 拦下不重复
输入示例：
在古修洞府得到《紫霄剑诀》（攻伐，玄阶中品，熟练度70），当场参悟学会
学会《龟息功》（护体，黄阶上品，熟练度60）
参悟《五行遁术》（遁术，玄阶上品，熟练度120）

V4 · 输入物品解析器（引擎直接入册）
新增（9295–9330 行）：
DIRECT_ITEM_UNIT：24 个量词（颗/枚/瓶/卷/柄…）
DIRECT_ITEM_CATS：六类（丹药/符箓/毒药/灵材/法器/玉简）
guessItemCat：品类推断——显式品类词 > 行囊已有同名 > 名字关键词 > 灵材兜底
parseDirectItems(action, skipNames)：三种识别写法（《名》/数量+量词+名/名×N），跳过已判定为功法的名字，品阶词自动剥离，玉简上下文强制归玉简类
接入：applyTurn（6335–6350 行）——directItems 并入 itemsAdd 走同一条入册/查重/升阶链路；灵石特判直接并入家财（横财上限已放开）。
输入示例：
捡到三颗回春丹
抢来疗伤丹×5
买下一柄天阶上品飞剑
得到一卷玉简《太上感应篇》
从废墟里翻出两瓶筑基丹

功法 / 物品自动区分：学习动词（学会/习得/参悟…）或路数/品阶/熟练度，或《名》以"诀/经/谱/典/咒/掌/拳/功"结尾 → 功法；其余 → 物品。一条输入可同时获得功法和物品。
V5 · NPC 相貌显示修复（1 处）
行号
原代码
新代码
作用
8396
['相貌',[npcPortrait(n),n.appearance].filter(Boolean).join('；')...]
['相貌',n.appearance||'—']
NPC 详情"相貌"只显示人物描述，去掉画像（雪碧图）描述

V6 · 父母婚配关联修复（3 处）
问题：开局生成时父亲与母亲之间只挂了"亲属"关系、没有"夫妻"关系，导致 NPC 详情婚配显示"单身"。
修复（2590–2608 行新增 backfillSpouses）：
规则：恰好"一父一母"、无继亲、性别相异 → 补建夫妻关系；复杂家庭不自动配（防错配）；幂等（已有夫妻/妾室不动）
挂点 1（6020 行）：新开局生成后立即执行
挂点 2（1183 行）：migrate 读档迁移末尾执行——旧存档读档即修复，无需重开
修改总表（27 处）
版本
改动数
核心内容
生效范围
V1
12
自由输入必成 + 6 项上限绕过
自由输入动作
V2
2
功法/物品品阶不压档
自由输入
V3
4
功法解析器（常量+函数+2 接入点）
自由输入含《功法》
V4
5
物品解析器（常量+2 函数+2 接入点+灵石特判）
自由输入含物品
V5
1
相貌只显人物描述
NPC 详情弹窗
V6
3
父母自动结为夫妻
开局 + 读档迁移


使用方法
打开：浏览器直接打开 HTML 文件（本地双击即可，无需服务器）
填密钥：首次打开进"设置"填 OpenAI 兼容 API 密钥（默认 DeepSeek），存于浏览器 localStorage
总开关：FREE_INPUT_OVERRIDE 置 false 可还原原版判定逻辑
自由输入：在底部输入框直接打字（行动 / 功法 / 物品皆可，见上文示例），必成且绕上限
注意：密钥与存档都在浏览器里（换浏览器/清站点数据会丢，建议用游戏内"存档"导出备份）；文件本身零依赖、永久有效

验证记录
验证项
方法
结果
JS 语法
提取 3 个 <script> 块 node --check
3/3 通过（每轮改动后均重验）
判定公式
node 复算 sim(attr,roll,need,free,freeAct) 样张
原版样张失败/自由输入必成/骰20必成/骰1选项必败，行为符合预期
功法解析器
从交付版提取真实函数跑 16 用例
全过（字段提取/多门/默认值/压顶/误触发防护）
物品解析器
同上 16 用例
全过（功法物品分离/×N/量词/品阶剥离/玉简/灵石/混合输入）
婚配修复
backfillSpouses 8 用例
全过（标准双亲/幂等/覆盖相识/保持夫妻/复杂家庭不误配/继父不配/无 rels 兜底）


与原站的差异说明（作者后续更新，本版未同步）
对比时发现原站作者后续更新了以下内容，本修改版未同步（不影响本版功能）：
新增 resolveBagName（更聪明的物品名匹配：原名→拆数量→近似名）
takeItems 改用 resolveBagName
applyTurn 新增 itemsRemove 硬校验：正文没写到的东西不许扣（防"正文写交出筑基丹、itemsRemove 却扣了结婴丹"）
如需同步上述作者更新，或希望把本版改动回馈给作者，可另行处理。

已知边界（未改动部分）
斗法：自由输入"杀了他/切磋"仍走斗法引擎推演，未做必赢（改需动整套斗法结算）
对话中的自由发言未做必成处理（与行动回合是两条线）
熟练度/等级总上限 100 保留（"登峰造极"世界观），绕开的只是单回合增量上限
复杂家庭（多父多母、继父继母）不自动配婚，避免错配
已会功法不重复入册（artKnown 同名/包含/相似≥0.7 拦截）
