# 第 21 章：后台 Agent 舰队——脱终端常驻与 daemon 监管

> 前面几章给了你两根轴上的自治。[第 17 章](17-autonomy-goal-loop.md)的 `/loop` 是时间轴：同一个会话隔一阵回来再干一轮。[第 19 章](19-dynamic-workflows.md)的 Dynamic Workflows 是空间轴：一段脚本此刻就把几十上百个 subagent 铺出去。但这两样都还活在一个绑在终端上的会话里——你把终端关了，它就没了。这一章讲 Claude Code 怎么把这条最后的绳子也剪断：一个会话怎么从终端上脱开、关掉终端也照跑，以及当你手里同时有好几个这样的后台会话时，它们是怎么被当成一支舰队来照看的。触发它的是一条命令 `/bg`，管它的是一张表 `claude agents`。
>
> 这一章的证据分两头，而且分得很不对称。脱终端会话在会话内的那套底座——background-task UI、`AgentTool` 里的 `run_in_background`——在泄露快照（标称 v2.1.88）的源码里能逐行读到。但真正把多个后台进程当舰队来监管的那台 daemon supervisor，整个没随快照泄露：在快照里 `grep -rlF 'tengu_bg_'` 命中零个文件，它只在 2.1.202 二进制的字符串里留了痕迹。所以本章会反复说清楚一件事：哪些是源码能读到的机制，哪些是二进制字符串里近八十个事件名拼出来的功能地图，哪些是从事件名往里推的逻辑，哪些是本地进程模型、根本照不进来的盲区。

## 21.1 两条轴之外的第三件事：脱终端

先接上刚才那两根轴。`/loop` 让会话在时间上续命，Workflow 让会话在空间上分身，但它们有个共同前提没被打破：会话跟终端是一根绳上的。你的 shell 在，会话就在；shell 一退，进程组收到信号，会话跟着走。这种长任务往往要跑几十分钟到几小时，启动它的人早就该转身去忙别的，这根绳子于是很碍事——你不能为了让它接着跑，就一直守着一个终端不关。

后台舰队要解决的正是这件事，它引入的第一个动作是"脱离"。`/bg` 把当前会话从终端上摘下来、丢到后台常驻，你的终端立刻腾出来干别的，而那个会话在别处接着跑。字符串里把这层意思写得最反直觉的一句是 "Sessions keep running if you close the terminal."——会话不再是终端的附属物，你关掉终端，它照活。这跟前两根轴是正交的：`/loop` 和 Workflow 说的是"一个会话内部怎么排布工作"，脱终端说的是"会话本身怎么脱离你的在场而存活"。三样叠起来，才凑齐"无人值守"这个词的全部含义——既不用你守着看，也不用你守着连。

换个角度看，这一章讲的东西其实是前几章的底座。`/loop` 的那一轮轮续跑要有个东西替它守着别掉，Workflow fan-out 出去的一批 subagent、多 agent team 里那些并行成员，也都默认脚下有一层"进程不会平白蒸发"的地基。前面几章各讲各的编排——什么时候派、派几个、怎么收——却都没把这层地基翻开细看。这一章讲的正是它：一个脱了终端的会话，怎么在你不在场、终端也关掉的情况下，仍有一台东西盯着它的死活。子 agent 跟脱终端会话同属这片后台能力版图，只是它们到底是不是由这台 `/bg` daemon 监管，现有证据打不通这道边界（详见 21.6）——所以这一章读起来不像一个新功能，更像是把前面几章一直踩着、却没细看的地面翻开给你看。

## 21.2 `/bg`：把会话从终端上摘下来

`/bg` 这条命令的自我介绍就一句话，把该说的都说了。字符串里逐字是这样：`/bg` detaches this session to run in the background，然后 `claude agents` 会把每一个被丢到后台的会话汇进一张表，每行带一个状态色——扫一眼就知道哪个在等你，空格回一句、回车进去接管。配套的 onboarding 提示更直白：`/bg` this session, then run `claude agents` in a new terminal。一个动作把会话推到后台，另一个动作在别的终端把它捞回来看。

摘下来之后会话并不会变哑。它还在跑、还可能需要你——比如问你一个只有你能拍板的问题——只是你此刻不在它面前。所以脱终端不是"发射后不管"，而是"发射后改成异步照看"：会话继续推进，需要你的时候在那张表里亮个状态，你有空了再回去。这也解释了为什么这套东西会跟本机 CLAUDE.md 里那条手机远控串到一起——本地 `/bg` 脱离、手机上 attach 回来、进程始终由后台守着，同一个会话在本地、手机、常驻三处之间来回倒手，靠的就是"会话不绑终端"这一条。

## 21.3 `claude agents`：一张表管所有后台会话

脱终端解决了"一个会话怎么活下去"，`claude agents` 解决的是"好几个会话怎么一起看"。它在 CLI 里是一个正经的 Commander 子命令，描述逐字写着 Manage background agents。它带着一小组动作：`claude agents` 列出所有后台会话，`claude attach {id}` 把某一个在当前终端里打开接管，`claude logs {id}` 翻它最近的输出。这三条合起来就是一套朴素的舰队面板——列表看全局、attach 进单个、logs 事后翻账。

那张表的关键是那个状态色。它把"哪个会话需要你介入"压缩成一列能一眼扫过的信号——你不必挨个 attach 进去看，扫一眼就知道该先管谁。至于具体哪个颜色对应哪种状态，字符串里只写了"用状态色扫出哪些需要你"，没把颜色到状态的映射写死，本文也不替它下结论。这里还藏着一条会咬人的规则：一个会话如果已经作为后台 agent 在跑，你就没法在别处直接 `--resume` 它。字符串把话说得很清楚——它现在是个后台 agent，去 `claude agents` 里 attach 上它，或者先在那儿把它停掉，才能在这边 resume。同一个会话不允许被两处同时接管，attach 与 resume 是互斥的入口。这条约束本身透出一点：后台会话不是可随便复制的一份状态，它有唯一归属，必须被独占接管，是个活进程。

## 21.4 表面之下：一套跑在本地的进程舰队监管器

到这里为止都还是用户看得见的一层。真正撑起"关了终端也不掉、崩了还能回来"的，是二进制字符串里那近八十个以 `tengu_bg_` 打头的事件名拼出来的一台监管器。要先把话说在前头：这一节是全章证据最不对称的地方。事件名本身是 2.1.202 字符串里逐字读到的，实打实；但每个事件在什么条件下发、彼此怎么串成状态机，快照里没有一行对应源码可查（`grep -rlF 'tengu_bg_'` 命中零个文件），只能从事件名加周边压缩代码往里推。所以下面对"它在干什么"的描述，名字是硬的，逻辑是推的，我按这个分寸写。

把这些事件按名字聚一下，浮出来的是五六个子系统，各管一摊，凑起来活像一台小号的进程主管。最上面是总入口 `bg_agent_action` 和一族 `bg_dispatch`：把任务派下去，派之前先让 `bg_classify` 归个类。派的路上有一串兜底动作——`_dispatch_rescued` 把它救回来、`_stale_drop` 嫌它太旧丢掉、`_sigkill_escalate` 一路升级到 SIGKILL、`_low_mem` 和 `_fallback` 在内存紧张时走降级路。再往下是 daemon 的生死：起不来是 `_daemon_spawn_failed`，装服务是 `_daemon_install`，把僵死的重启是 `_daemon_zombie_restart`，又靠 `_zombie_false_positive` 不误杀假僵死，冷启动先问你一句是 `_daemon_cold_start_ask`；它还给 Windows 和 macOS 各铺一条路，分别叫 `_daemon_wmi_fallback` 和 `_daemon_macos_aqua_wrap`。

再往下有意思了。有一族 `spare` 事件——`_spare_spawn`、`_spare_claim`、`_spare_enable`、`_prewarm_per_sweep`——按名字推测，像是在维护一个预热好的备用 worker 池：任务还没来，就先把几个进程热在那儿，任务一到直接认领，省掉现起进程那几秒冷启动。真正 worker 的起落是另一族：`_worker_spawn`/`_exit`/`_vanished`/`_stalled`——生、正常退、莫名消失、卡住不动，各有各的事件。而最像 supervisor 的是 respawn / adopt / orphan 这一组。worker 崩了自动重生记 `_respawn`，重生前先由 `_respawn_unconfirmed_bail` 防重复；没主的进程被 `_adopt` 收养，收养时 token 丢了就走 `_adopt_token_lost_respawn` 重生；没人认领的孤儿由 `_orphan_reap` 回收。整支舰队的花名册叫 roster：孤儿被登记回来是 `_roster_orphan_adopted`，登记解析失败是 `_roster_parse_failed`。attach/detach 那一族管你接管的瞬间——首帧记 `_first_frame`、卡住重生记 `_stall_respawn`、踢一脚记 `_attach_kick`；最底下还有一层 transport，管 PTY 拿不到、认证对不上、协议不匹配、桥接被截断这些管道层的意外。

拿一个 worker 走一遍，就能感觉到这套东西的密度。它由 `_worker_spawn` 诞生，被登进那本叫 roster 的花名册；跑着跑着要是莫名没了记 `_vanished`、卡住不动记 `_stalled`；从 `_respawn` 和 `_respawn_unconfirmed_bail` 这两个名字推，监管器不会当没看见，而是判要不要重生，且重生前先确认它是真死、而非网络抖了一下，免得同一个活儿起两份。机器内存吃紧时又是另一套动作：它把占着内存又不那么要紧的 worker 退休掉，给要紧的腾地方，对应 `_retire_pinned_low_mem` 和 `_low_mem_mb`。要是监管器刚接手，发现一堆没主的进程还在跑（比如上一个 daemon 没干净地退），它会用 `_adopt`、`_orphan_reap`、`_roster_orphan_adopted` 把这些孤儿认领回来，而不是放任它们变成谁都不管的僵尸。这几条连起来，就是一台监管器该有的样子：记账、查活、崩了补、挤了退、没主的收编。

把这几摊合起来看，结论就很清楚了：`/bg` 背后不是"起个后台进程"这么轻。它是一整套带健康检查、崩溃重生、孤儿收养、内存压力退休、预热备用池的进程舰队监管，量级接近 systemd 或 pm2 的一个子集——只不过整台机器是在你本地客户端里跑的，为的是让那些脱了终端的会话，在你不看时也有台东西替你盯着死活。这台监管器是本章最硬的骨头，也是最只能靠字符串说话的部分：名字是从二进制里逐字读到的，可它们连成的这台状态机长什么样，是我照着名字和周边逻辑推的，不是从某份源码里读出来的——这个分寸得替你记着。

## 21.5 按需起、闲了退、可以一键关

你可能以为这么一台监管器是个常驻守护进程，开机就在那儿吃内存。不是。字符串里它的定位是 on-demand——用到才起。没有活儿吊着它的时候，它会说自己 nothing holding this daemon open，随即 idle-exit 退掉；下一次你敲 `claude agents` 或 `claude --bg`，再拉起一个新的。这跟第 17 章那种"常驻盯着目标"的自治是两种脾气：监管器只在真有后台会话要看的时候存在，没活就自己走，不白占资源。

按需起也带来一个要防的坑：同一时刻起了两个 daemon 怎么办。字符串里有一条很干净的让位规则——an on-demand daemon never displaces a running one，只有 transient（临时的那种）才允许被顶掉。翻过来就是：已经在正经看着一堆会话的 daemon 有优先权，一个刚按需起来的不许把它挤掉，避免两台监管器打架、把舰队的账记乱。

整套东西也留了总闸。设置项里有一条逐字写着：Disable agent view（`claude agents`、`--bg`、`/background`、还有那个 on-demand daemon 一起关），通常在托管设置里下发，等价于环境变量 `CLAUDE_CODE_DISABLE_AGENT_VIEW=1`。也就是说，脱终端加舰队监管这一整块能被一个开关整体摁掉——对不希望员工机器上冒出后台常驻进程的组织，这是一条能从上面统一关死的路。

## 21.6 同一片版图的另一层：子 agent 默认后台

到这儿要补一层跟脱终端会话并列、但证据来源完全不同的东西：子 agent 的后台化。你 `/bg` 出去的是整会话；而在一个会话内部，主会话派出去的子 agent 默认也跑在后台。这正好接回[第 8 章](07-multi-agent.md)和[第 19 章](19-dynamic-workflows.md)那套 `run_in_background`：在当前版本里，子 agent 默认就是后台跑的。`AgentTool` 的描述在快照源码里能读到，字符串里也留了几个措辞略有出入的变体，意思一致：Subagents run in the background by default，跑完会通知你；只有当你在往下走之前非拿到它的结果不可，才传 `run_in_background: false` 让它同步跑。schema 里 `run_in_background` 的默认值就是 true，还带一个 `isolation: 'worktree'` 字段管文件系统隔离——这份 `run_in_background` 在快照里出现在十个文件（`AgentTool`、`BashTool`、`PowerShellTool`、bundled 的 batch 等），这一层有源码可逐行读，不是从字符串往里推的。

后台跑的子 agent 干完活怎么回来？靠 `<task-notification>`。它完工时不是把结果塞回你当前那句话，而是作为一条 task-notification 重新进入主循环——压缩代码里它带着 `promptOrigin:{kind:"task-notification"}` 的标记，还被 `promptIsMeta` 标成一条元消息、进到一个专门的命令队列模式（`new Set(["task-notification"])`）里。换句话说，"某个后台 agent 完工了"这件事在主循环里有它自己的一类入口，跟你手打的那句话是两回事——主循环因此把"人的输入"和"后台回报"分成两种来源各自处理，而不是把回报硬塞成一句伪装的用户消息。这跟那个 `isolation: 'worktree'` 字段是一套配合：schema 里暴露了 worktree 这一隔离选项，后台并行的子 agent 可以走独立的 git worktree、互不覆盖对方的改动，正是[第 8 章](07-multi-agent.md)讲过的隔离底座。至于是不是每个子 agent 都各占一个 worktree、默认策略怎么定，schema 本身没坐实，本章也不替它下这个结论。

这条通知机制还配着一句很重的行为约束，三个变体里最狠的那版逐字写着：当一个 agent 在后台跑，它完工时你会被自动通知——do NOT sleep, poll, or proactively check on its progress。别睡、别轮询、别主动去戳它。这句话把"异步"这件事从建议钉成了纪律：你派出去就该松手，等通知，而不是守在那儿反复查——恰好也是本机运行时纪律里"禁 sleep-poll"那条的同源出处。所以"后台"在 Claude Code 里其实是两层叠着：一层是你手动 `/bg` 出去、由 21.4 那套 daemon 照看的脱终端会话；另一层是主会话在会话内派出、默认落到后台、完工用 task-notification 叫你回来的子 agent。后一层由快照源码坐实，前一层只有 strings；它们同属后台能力版图，但子 agent 到底是不是由那台 daemon 监管、算不算舰队的"兵"，现有证据打不通这道边界，本章不把它当实锤写。

## 21.7 干完活自己收尾：commit、push、开 draft PR

一个能脱终端、能几小时无人值守的后台任务，还差最后一环才算闭合：它把活干完了，得替你把结果落下来，而不是把一堆改动晾在那儿等你回来手动收。字符串里有一段专讲这件事的收尾指令，行为要点是这样的（原文的关键短语引在附录里，正文是转述）：改完代码就 commit，push 那个分支，然后开一个 draft PR（`gh pr create --draft`），别停下来问；不许用 "say the word and I'll open the PR" 这种话把活撂在没提交的状态里结束。

难得的是它同时把红线也钉死在同一段里：Never push to main/master, force-push, or merge——绝不直推主干、绝不 force-push、绝不 merge。这跟[第 18 章](18-auto-mode.md)里 Auto Mode 那组 Git 规则同向：推到你自己的功能分支放行，直推默认分支和改写历史拦下。也跟本机那条 auto_commit_deploy 约定同向——commit 和开 PR 属于"替你把活收好"，push 到主干、merge 属于"替你拍板发布"，前者可以自动、后者必须留给人。字符串里还接着交代了一种退路：万一你没能隔离出 worktree（EnterWorktree 失败，或者当前目录本来就是用户自己的 checkout），收尾逻辑要换一种更保守的走法。这一整段都是行为描述、没有对应源码，属于二进制字符串里能逐字读到、但只能看到"它被要求这么干"、看不到"它在代码里怎么实现"的那一档。

## 21.8 它在遥测里长什么样

这套东西对接的是[第 16 章](16-observability.md)那套离散事件遥测。近八十个 `tengu_bg_*` 事件本身就是一张功能地图——把它们按前缀在字符串里数一数词项密度，舰队各部件的分量分布就出来了。先说清楚：这是二进制里静态字符串的词项计数，不是运行时的事件触发频率。静态拓扑大致是：`background` 一千多处压着场子，`attach` 五百多、`daemon` 三百多，说明"脱终端 + 接管 + daemon 生命周期"是主干；再往下 `detach`、`spare`、`task-notification` 一两百量级，`supervisor`、`roster` 几十处，对应前面那几个更专门的子系统；最稀的是 `/bg`、`draft PR` 这种个位到十几处的，它们是入口和收尾这类只在少数几处被引用的动作，字符串里出现得稀也正常。

这张词项密度表的用处不只是好看。它是一条侧证：`background`/`attach`/`daemon` 在字符串里的密度远大于 `spare`/`roster`，跟 21.4 里推的子系统分工能对上——总入口和 daemon 生死是被反复引用的主干，预热池和花名册是更专门、引用更少的维护性子系统。但别把它读成运行时频率：这是静态拓扑，不是 telemetry 跑出来的调用次数。第 16 章讲的"用离散事件复盘一次运行"落到这里，就是你能顺着一个后台会话真正发出的 `tengu_bg_*` 序列，把它从 dispatch、spawn、attach、到最后 detach 或 reap 的一生串出来——前提是你能拿到那一次运行完整的 `tengu_bg_*` 事件序列。

## 21.9 一条可对账的演进线

这套能力也能两端对账，而且对得比前几章更干脆——因为它的证据本来就分在两头。

快照那头（泄露快照，标称 v2.1.88）：会话内的后台底座已经齐全。`components/tasks/` 下那几个大文件（`BackgroundTasksDialog`、`BackgroundTaskStatus`、`TaskListV2`）、`tasks/LocalAgentTask/`、还有二百多 KB 的 `AgentTool`——带着 `run_in_background` 和 worktree 隔离——都在里面，能逐行读。也就是说，"子 agent 默认后台、完工用 task-notification 回来"这一层，快照时就成型了。

当前这头（2.1.202）：把单会话的后台任务撑成"脱终端 + 多会话舰队监管"的那台 daemon supervisor，整个是新长出来的。`/bg`、`claude agents`/`attach`/`logs`、on-demand daemon、那近八十个 `tengu_bg_*` 事件，在快照里一个都搜不到（`grep -rlF 'tengu_bg_'` 命中零文件），只在 2.1.202 二进制的字符串里留了痕。对一下就看出这半年发生了什么：会话内怎么派后台子 agent，快照时就有；而把这些后台会话当成一支需要健康检查、崩溃重生、孤儿收养的舰队来监管，是快照之后才铺出来的一层。又一条"没随快照泄露，不等于分析不了"的线——只不过这一次，泄露的和没泄露的，正好沿着"会话内"跟"跨会话"这道缝分开。

## 21.10 附录：我们怎么知道的（逆向方法，可复现）

这一章的证据来自四条能自己复现的路，只是这次四条路的分量特别不均。

第一条，读快照源码。会话内的后台底座全在 v2.1.88 快照里：`AgentTool` 的 `run_in_background`、`components/tasks/` 那套 background-task UI、`LocalAgentTask`——两百多 KB 的组件加一族 hook，能定位到文件。它给的是"会话内怎么派后台子 agent、怎么用 task-notification 收"这一层的机制。

第二条，看 CLI 自己的命令面。`claude agents` 是个正经 Commander 子命令，`claude --help` / `claude agents` 能看到 agents / attach / logs 这组动作和它们的描述，`CLAUDE_CODE_DISABLE_AGENT_VIEW` 这个关停开关也能在设置里查到。这条路比翻二进制干净，但它只能确认命令面和开关的存在，够不到底下那台监管器。

第三条，从二进制抽字符串。这一章的大头在这里：那近八十个 `tengu_bg_*` 事件名、`/bg` 的脱终端文案、"Sessions keep running if you close the terminal"、on-demand daemon 的 idle-exit 和让位规则、收尾 commit/push/draft-PR 那段——都是 `strings` 加 `grep -aF` 从 2.1.202 二进制里逐字读到的。快照里 `grep -rlF 'tengu_bg_'` 命中零文件，所以 daemon 舰队监管这一整块，字符串是唯一能拿到的证据。

第四条，抓明文流量。架一个明文反向代理（配方见[第 17 章](17-autonomy-goal-loop.md) 17.5），你能看到一个后台子 agent 被派出去的请求、以及它完工后那条 `<task-notification>` 重新进主循环。但这条路对本章有个天然的天花板得讲明：daemon 怎么监管它底下的 worker 和 session、PTY 怎么转、roster 怎么记，走的是本地进程间通信和 PTY，不是 HTTPS——它们根本不上网，反代抓不到。这恰恰是为什么第三条（字符串）在本章挑大梁：舰队监管的行为真相，除了字符串没有别的旁证。

诚实边界一句话摆清楚。会话内后台子 agent 那一层由源码坐实（源码、字符串、加本会话里注入的工具描述，三处印证）；`/bg`/`claude agents`/on-demand daemon/关停开关/收尾指令/近八十个事件名，由 2.1.202 字符串坐实；而每个事件在什么条件下触发、dispatch 与 respawn 与 adopt 怎么连成状态机，是从事件名往里推的推理级，不是源码；至于 daemon supervisor 的进程模型细节、PTY-RV transport 的协议内部、roster 文件的具体格式，是拿不到的盲区——它们既没进快照，也不走网络。这条接缝，本章从头到尾没试图抹平。

---

## 附录：后台舰队的关键字符串与工具描述（含省略）

下面把本章倚重的原文摘在这里。UX 文案、on-demand daemon 与关停开关、子 agent 后台默认的三个变体、收尾指令，都是从 2.1.202 二进制里逐字读到的；`tengu_bg_*` 事件族给的是分组后的事件名地图（名字逐字、分组是归纳）；较长处用 `…` 省略。daemon 监管器的内部状态机逻辑没有对应源码可贴，这里只列它留在字符串里的事件名。

### `/bg` 与 `claude agents`（逐字，2.1.202 字符串）

```text
/bg detaches this session to run in the background, and `claude agents` shows
every backgrounded session in one table with a status color — glance to see which
ones need you, space to reply, enter to attach.

（onboarding 提示）/bg this session, then run `claude agents` in a new terminal

（脱终端存活）Sessions keep running if you close the terminal.

（resume 冲突）… as a background agent. Open `claude agents` to attach to it, or
stop it there first to resume here.

（Commander 子命令，逐字）.command("agents").description("Manage background agents")
```

作者归纳的命令面（非逐字；原始串里子命令实参是 `${e}` 模板，这里换成 `<id>` 便于阅读）：

```text
claude agents        列出所有后台会话
claude attach <id>   在当前终端接管其中一个
claude logs <id>     翻它最近的输出
```

### on-demand daemon 与关停开关（逐字，2.1.202 字符串）

```text
（idle-exit）nothing holding this daemon open — will idle-exit shortly …
the next `claude agents` or `claude --bg` will start a new one

（让位规则）an on-demand daemon never displaces a running one …
only a transient daemon can be displaced

（settings schema 关停开关）Disable agent view (claude agents, --bg,
/background, the on-demand daemon). Typically set in managed settings.
Equivalent to CLAUDE_CODE_DISABLE_AGENT_VIEW=1.
```

### 子 agent 默认后台（逐字，三个变体，源码 + 字符串）

```text
Subagents run in the background by default; you'll be notified when one completes.
Pass `run_in_background: false` for a synchronous run when you need the result
before continuing.

Agents run in the background by default; you will be notified when one completes.
Set to false to run this agent synchronously when you need its result before
continuing.

Agents run in the background by default. When an agent runs in the background, you
will be automatically notified when it completes — do NOT sleep, poll, or
proactively check on its progress.
```

### 完工收尾：commit / push / draft PR（行为描述摘录，含首尾省略、非严格逐字；2.1.202 字符串）

```text
… the task: when you've made code changes, commit them, push the branch, and open
a draft PR (`gh pr create --draft`) without stopping to ask — don't end the job
with uncommitted work or 'say the word and I'll open the PR'. Never push to
main/master, force-push, or merge. If you're working in the user's own checkout
instead — you never isolated, EnterWorktree failed, or your cwd was already a
worktree …
```

### daemon 舰队监管器的事件族（事件名逐字，分组为归纳；2.1.202 字符串，快照命中 0 文件）

```text
[总入口 / dispatch]  bg_agent_action  bg_dispatch  _dispatch_rescued  _stale_drop
                     _sigkill_escalate  _low_mem  _fallback  _rejected
                     _watcher_failed  bg_classify  _classifier_config

[daemon 生命周期]    _daemon_spawn_failed  _daemon_install  _daemon_zombie_restart
                     _zombie_false_positive  _daemon_cold_start_ask  _ask_answer
                     _daemon_service_stale_exec  _service_poll_fallthrough
                     _daemon_wmi_fallback（Windows）  _daemon_macos_aqua_wrap（macOS）
                     _daemon_binary_takeover  _daemon_bg_disabled_skip

[spare / worker 池]  _spare_claim_fail  _spare_spawn  _spare_claim  _spare_enable
                     _worker_spawn  _exit  _vanished  _stalled
                     _prewarm_per_sweep  _low_mem_mb  _retire_pinned_low_mem

[respawn/adopt/孤儿] _retired  _respawn  _respawn_unconfirmed_bail  _stale
                     _exhausted  _no_transcript  _adopt  _adopt_token_lost_respawn
                     _upgrade_respawn  _unverified  _sock_unlinked  _orphan_reap
                     _roster_orphan_adopted  _roster_parse_failed

[attach / detach]    _attach  _attach_outcome  _first_frame  _legacy_autorespawn
                     _upgrade  _stall_respawn  _stall_ms  _stall_gave_up  _attach_kick

[transport / PTY]    _pty_unavailable  _pty_auth_mismatch  _ptyhost_crash
                     _proto_mismatch  _rv_reply_rejected  _rv_connect_exhausted
                     _rv_auth_mismatch  _bridge_flush_truncated  _skew_nudge
                     _binary_takeover
```

> 来源：`/bg`、`claude agents`/`attach`/`logs`、on-demand daemon、关停开关、收尾指令，以及全部 `tengu_bg_*` 事件名，来自对 2.1.202 客户端二进制的明文字符串抽取；子 agent 后台默认的工具描述另有 v2.1.88 快照源码与本会话注入的工具集两处印证。事件名逐字、分组为归纳，`…` 处为省略。daemon 监管器的内部状态机逻辑（各事件触发条件、dispatch/respawn/adopt 的连接）无对应源码，属推理级；PTY-RV transport 协议内部与 roster 文件格式在快照里 `grep -rlF 'tengu_bg_'` 命中 0 文件，是拿不到的盲区。
