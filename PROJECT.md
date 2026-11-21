# DsFish「摸鱼 Chat」项目速览

> 给 AI 助手压缩记忆用的项目文档。人也能看，但按"快速恢复上下文"的密度写的。
> 更新：2026-11-15

## 一句话

uni-app x (uvue/UTS) 写的纯本地 AI 聊天 App，Android 优先（web 预览 http://localhost:5173），无后端无账号，唯一联网出口是用户配置的 LLM 接口（DeepSeek/OpenAI/Ollama），数据全在本地 storage。视觉是「编辑部」纸质风：纸底 #F7F2E8 / 墨 #1C1A16 / 砖红 #B4553C / 发丝线 #DCD4C4，衬线标题（Songti SC）。用户要求**始终中文回复**。

## 文件地图

| 文件 | 职责 |
|---|---|
| `pages/chat/chat.uvue` (~2500行) | 主聊天页：流式、断句、工具轮、主动消息、图片、好感度、骰子 |
| `utils/store.uts` (~1700行) | 数据层：类型/存储/种子/内置卡/备份导入导出/提示词拼装 |
| `utils/api.uts` (~950行) | 网络层：SSE 流式、abort、DSML 兜底、buildRequest/<系统消息>块 |
| `utils/memory.uts` (~850行) | 长期记忆：存储/打分查询/memoryTools schema/memoryRule/runMemoryTool/强度衰减+自动遗忘 |
| `utils/cardgen.uts` | 角色卡生成器：模板一(生成提示词)+模板二占位(合并模板)+parseCardJson 导入解析+附加规则段 |
| `utils/context.uts` | token 粗估 + 超预算压缩切分 planSplit |
| `utils/reply.uts` | 流式断句 splitReply/lastSplitIndex/squeezeSpace |
| `utils/affinity.uts` | 好感度 7 档/衰减/parseAffinity/affinityRule |
| `utils/image.uts` | 选图→永久目录、缩略图落盘/读取（全 `#ifdef APP`，web 回空） |
| `utils/backup-file.uts` | 备份文件读写（App 文件 API） |
| `utils/opt.uts` | optStr()：跨端读 onLoad 参数（App getString / web 索引） |
| `utils/mood.uts` | 情绪系统，**MOOD_ENABLED=false 已禁用** |
| `pages/index/index.uvue` | 首页三 tab：会话列表/角色卡/我的（备份入口） |
| `pages/char/char.uvue` | 角色卡编辑（顶部大字=列表名，与角色昵称跟随/定制联动） |
| `pages/chatset/chatset.uvue` | 会话设定（标题/补充提示词/好感度开关/主动消息/pass 开关） |
| `pages/config/config.uvue` | 模型服务商配置（默认温度 1.0 / 最大输出 2048 / ctx 按服务商：DS 200000 其他 128000）；底部一行入口跳语音页 |
| `pages/ttsconfig/ttsconfig.uvue` | 语音配置独立页（开关/key/三音色预设 chip/语速音调模型/试听；预设整套参数联动） |
| `pages/memory/memory.uvue` `pages/restore/restore.uvue` | 记忆管理 / 备份恢复 |
| `pages/cardgen/cardgen.uvue` | 角色卡生成器：填需求→生成提示词→复制去外部模型 |
| `pages/cardimport/cardimport.uvue` | 导入 AI 生成的角色：粘 JSON→解析→摘要→存卡跳 char 页；与 cardgen 互跳 |
| `static/card/` | 内置卡头像（deepseek.png=小鲸鱼 deepseek2.png=大肥鱼 maoniang.png=铃喵） |

## 存储键（uni storage）

`dsfish_chats`、`dsfish_chars_v2`（注意带 v2！）、`dsfish_msgs_<chatId>`、`dsfish_profiles`、`dsfish_active`、`dsfish_me`、`dsfish_memory_v1`。

## 核心机制（改动前必读）

### 线程铁律（Android 崩溃教训）
canvas/img.onload/setTimeout 回调可能在子线程，碰 reactive data（this.xxx）必须包模块级 `runOnMain(fn)`（UTSAndroid.runOnUiThread，`#ifdef APP-ANDROID`）。全项目已按此清理，新增异步回调照此办理。

### 流式链路
api.uts `streamChat`：enableChunked + onChunkReceived，sseBuffer 按 \n 切行（data: 前缀和裸 NDJSON 都认），[DONE] 结束，45s 空闲超时兜底。DSML 兜底：deepseek-flash/v4 有时把工具调用以 DSML 标记文本吐进 content（官方已知问题），api.uts 本地识别、扣住不显示、收尾解析成本地 tool_calls。**变体必须放宽**：真实输出是双竖线 `<｜｜DSML｜｜invoke>`（半角/全角、单/双根都可能）——normDsml 用正则 `[|｜]+DSML[|｜]+` 归一；dsmlHoldPoint 直接在原文上正则定位（返回原始下标，勿在归一化文本上找下标再去 substring 原文，长度不同会错位）。abort/已收内容优先的 fail 处理。finish 有 finished 幂等标志。

### 断句显示
rawText 累积 → emitChunk 攒到句末标点切一条气泡（模拟连发）；flushTimer 处理末尾半句；displayText 剥 [affinity]/[mood] 标记 + stripSysEcho 剥复读的 `<系统消息>` 块（含半截扣住）+ **stripDsml 剥漏网 DSML 标记**（invoke/parameter 整元素连内容删、孤立标签删、半截扣住；竖线任意根数；闭标签斜杠在竖线**前** `</｜｜DSML｜｜x>`）。显示层兜底的意义：以后模型再冒新变体，顶多工具没执行，绝不上屏。

### 记忆系统
三个工具 memory_add/query/remove（schema 在 memoryTools，规则文字在 memoryRule，两者口径必须一致，否则模型挑软的听）。软删 3 天回收站、上限 500。强度：新建 60 起，被查询命中 +5/+1 封顶 100；**每天 -1 惰性衰减**（strengthOf 现算：存储值 − 距 lastHitAt/createdAt 天数），有效强度 ≤3 自动进回收站（autoForget，约 57 天；restore 会重置衰减时钟）。`tool_choice` 强制：用户消息含「记一下/记住/帮我记/别忘」→ 强制 memory_add，「来着/还记得/我说过/上次」→ 强制 memory_query（chat.uvue detectForceTool，只看本轮最后一条，仅第一轮生效）。**已知误伤**：「我记住了」「上次」等普通叙述会误触发（审查 M3，未修）。

### 上下文注入结构
system = composePrompt(角色提示词, {{char}}/{{user}} 替换) + `<系统工具说明>…</系统工具说明>`（包好感度/图片/记忆规则，开头有"不要复读标签"禁令）。每条用户消息前挂 `<系统消息>\n[当前时间][当前好感度][用户昵称]\n</系统消息>`（metaBlock，api.uts）。好感度显示按**每条消息的 aff 快照**（Msg.aff，发出时定格；老消息无快照回退当前值）。设计目标：请求内容逐字节稳定，DeepSeek 前缀缓存友好。

### 主动消息
停留聊天页 N 秒（10~180 可配，随机）无回复 → 发隐藏指令消息（mine+hidden）让 AI 主动开口；连续 3 次未回停（**[pass] 跳过也算一次**，2026-11 改的）；pass 吞气泡 + toast「XX 这次没说话」。proStrikes/proPending/proGone 状态在 chat.uvue。

### 图片
选图→saveFile 拷 USER_DATA_PATH/dsfish/（CACHE_PATH 会被系统清，不能用）；480px JPEG 缩略图 base64 落盘 `dsfish/t<id>.jpg`；Msg 只存路径（image=原图，imageThumb=缩略图），imageData 纯内存字段 saveMsgs 剥除；载入 backfillThumbs() 回填 → 重启后请求内容一致。图片功能仅 App 有效（web 无 FS）。

### 内置卡与种子
`SHOW_DEMO=false`（store.uts）→ seedDemo 跑 purgeDemo（删示例卡 ch1-5/会话 demo1-3）+ seedBuiltins。BUILTIN_CHARS 三张：cfish 大肥鱼(女儿版 label)、cmeow 铃喵(布偶猫娘)、cwhale 小鲸鱼——用户手改过，**提示词里雷点与名字有已知矛盾待用户定夺**。BUILTIN_OLD_NAMES 命中才做一次性迁移（改名/改提示词/补头像）。`seedDefaultChat()`：空列表时建默认会话，**只在首页 onLoad 调**（曾放 seedDemo 里导致"开聊建两条"bug，勿回退）。

### 角色卡生成器（app 内不调模型、不要 key）
闭环：描述需求 → cardgen 页拼模板一生成提示词（复制）→ 用户拿去外部模型 → 返回 JSON 粘进 cardimport 页 → parseCardJson 解析（剥围栏/截{} /修全角引号尾逗号/校验，失败报 stage+snippet）→ 摘要确认（可勾「去除 AI 味」「说话方式」附加段，默认双勾，追加到 prompt 尾部）→ 存卡跳 char 页微调。
- 首页「新建角色卡」弹底部菜单三项：自己写 / 让 AI 写 / 导入 AI 生成的角色。cardgen ↔ cardimport 页面互跳。
- 七键：name/identity/appearance/personality/speaking_style/init_dialog/initial_affection。前四字段按 CARD_MERGE_TEMPLATE 合并进 prompt（**不含「你是XX」头**——composePrompt 自动补，写了重复）；init_dialog 拆 user/assistant 进 dialogs；initial_affection 存 c.affinity。
- 模板一（CARD_GEN_TEMPLATE）和两个附加段（CARD_BLOCK_NOAI/CARD_BLOCK_SPEAK）是用户定稿文案**一个字不许改**；模板占位替换用 `{{用户输入}}`。

### 语音（BreezeBlue TTS，2026-11-21 从 MiniMax 整体迁移）
独立页 `pages/ttsconfig/ttsconfig.uvue` 配置（key 存本机 `dsfish_tts_v2`，MiniMax 时代的 `dsfish_tts_v1` 已弃用不再读）。接口 `POST {baseUrl}/v1/text-to-speech/{voice_id}?output_format=mp3`，头 `xi-api-key`，体 text/model_id/language_code=zh/instructions/voice_settings{speed 0.5-2, volume 0.01-2}；**响应是 mp3 二进制流**，Android 端 getFileSystemManager 落盘 `USER_DATA_PATH/voices/v<ts>.mp3` 永久有效（消息 voice 字段存本地路径，解决 MiniMax 24h 过期）。三预设（TTS_PRESETS，参数勿改）：迷糊版 voc_8yud7hz7c9q2（默认）、偶像版 voc_rn489h83rcvs、机灵版 voc_u3nxkd5578wc，各带 instruction 表演指令（自然语言，无 pitch 概念）。emotion 枚举→emotionInstruction() 翻成中文指令覆盖预设。send_voice(text,emotion) → 落盘 → 语音气泡（字幕=text，点击播放 playingId 单例）；结果回传 tool 消息继续下一轮。**平台无 CORS：web 预览端 synthesizeTts 直接短路报错（条件编译 #ifndef APP-ANDROID），只有 Android 能合成**——web 端 E2E 只能测配置层。试听文案「你怎么又在点试听呀…」。计费：45 字符≈4.5 分（balance 接口可查）。待升级：角色卡绑音色、mood 联动 emotion、历史语音文件清理。

主动消息连发上限 proCapOf(chat)（Chat.proCap，1~10 缺省 2）：chat.uvue 两处 `proStrikes >= proCap` 判断，chatset「连发上限」输入框，setProactive 第六参。

### 缓存清理（utils/clean.uts）
「我的」页「清除缓存」行：scanCache() 显示孤儿文件数/体积（进 tab2 时刷新），确认后 cleanCache() 删除。孤儿判定 = USER_DATA_PATH 的 dsfish/ 和 voices/ 目录里的文件不在引用集（会话 charAvatar/bgImage + 消息 image/imageThumb/voice + 角色卡 avatar/bgImage + Me.avatar）中。web 端无文件系统安全空转。语音同步链路（onChunkReceived 攒块 → arrayBufferToBase64 → writeFileSync base64），存盘失败自动退 history-item-id 异步链路。

## 已确认未修的问题（2026-11 审查，按优先级）

1. **[高] 停止/离开的窗口期幽灵请求**：工具轮间歇 setTimeout(chat.uvue ~1280)无 proGone/sending 守卫；「停止」在工具轮间歇/压缩阶段拦不住后续轮（abortChat 只杀当前流，aborted 还会被下轮 streamChat 重置）→ 并发流共用模块级 sseBuffer 互踩、删除的会话消息可被复活。建议：请求代际 id + 页面存活标志统一守卫。
2. **[中] 主动轮 onError 不回收 hidden 指令**（chat.uvue ~1195）：连续网络失败会攒出多条空分隔行。
3. **[中] detectForceTool 关键词误伤**（见上记忆系统）。
4. **[中] onUnload 漏清 flushTimer/工具轮延时**。
5. **[中] toolDegraded 降级重跑重复气泡**；isToolRejected 子串匹配过宽。
6. **[低] Msg.id=Date.now() 同毫秒撞 id**；emoji 硬切劈半代理对；pass 守卫把 p/跳 开头文本整轮扣到流尾（观感）。
7. **[低] 删角色卡不清头像文件**（deleteChar 不调 removeAvatarFile）。
8. 大肥鱼卡：名字改「大肥鱼」后雷点段「被叫大肥鱼会炸毛」自相矛盾，等用户定夺。
9. 旧配置默认值不追溯（温度/maxTokens/ctx 改默认只影响新装），用户未要求迁移。

## 测试基建（%TEMP%，重启会丢）

- `dsfish-cdp.mjs <url> <out.png> [w] [h] [--eval|--evalfile f] [--wait ms] [--shotdelay ms]`：无头 Edge CDP 截图+执行 JS。**每次运行全新浏览器，localStorage 不跨运行**；plant+触发+断言必须写在一个 --evalfile 脚本里；脚本是纯 JS（无 TS `!`）。页面里 `performance.getEntriesByType('resource')` 找含 'store.uts'/'api.uts'/'memory.uts' 的 URL → `await import(url)` 拿同一模块实例；**未被页面引用的新 .uts 模块要带 `?import` 查询**（如 `import('/utils/cardgen.uts?import')`）否则动态 import 失败。Vue 页实例从 `document.querySelector('#app').__vue_app__` 树里按 $data 键找。
- `pro-mock-server.js`：SSE mock（REPLY 环境变量，`|` 分块，NL=换行，/last 看最后请求）。`pro-mock-tool.js`/`pro-mock-dsml.js`：工具轮/DSML 复读场景 mock。用完 kill，端口 8787。注意 mock 的 DSML 样例要用**双竖线**真实格式（2026-11 曾因单竖线 mock 自验自通过、漏掉真机双竖线 bug）。
- %TEMP% 会被系统清（2026-11-15 发生过一次全丢）；dsfish-cdp.mjs 已重建过，丢了就照上一条的用法说明重写。

## 协作约定

- 全程中文；改完尽量 web 端 CDP 实测再汇报；web 能验的：逻辑/存储/请求拼装，验不了的：Android 线程/文件系统（用户 HBuilderX 编译真机）。
- Vite 对 .uvue 的 HMR 缓存偶尔不失效（改 store.uts 引用、页面没变）→ touch 页面文件强制重编译。
- HBuilderX 编辑器缓冲可能盖回旧文件（memory.uts 发生过）：外部改动后提醒用户关标签重开。
- UTS/Kotlin 编译警告族：`x != null` 恒真（类型声明非空处别防御）、try/catch 前赋 null 冗余（提取小函数返回可空）、JSON.parse 返回可空要显式判、死代码会被摇树（未引用的 export 别留）。
- 发版/git 约定见 `~/.dsh/AGENTS.md`「DsFish 项目发版约定」。
