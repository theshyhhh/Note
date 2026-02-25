# 虚幻引擎 Gameplay Ability System (GAS) 高级架构深度研究报告：复制、实例化与网络策略全解析





## 摘要



虚幻引擎（Unreal Engine）的 Gameplay Ability System（GAS）代表了多人游戏开发中 gameplay 逻辑架构的一次范式转移。它将传统的单体角色逻辑解耦为离散的 `UGameplayAbility` 对象、`UAttributeSet` 属性容器以及 `UGameplayEffect` 修改器。这种模块化设计虽然提供了无与伦比的灵活性，但也引入了极高的复杂性，特别是在网络同步、内存管理和执行流控制方面。

本报告旨在对 GAS 架构中的核心高级配置变量——`ReplicationPolicy`（复制策略）、`InstancingPolicy`（实例化策略）、`NetExecutionPolicy`（网络执行策略）以及 `NetSecurityPolicy`（网络安全策略）——进行详尽的深度剖析。基于最新的引擎文档与社区研究，特别是针对虚幻引擎 5.5 版本中 `NonInstanced` 能力的废弃 1 这一重大变革，本报告将论证 GAS 的最佳实践已从对象级复制模型全面转向基于预测的事件驱动模型。

报告将揭示现代 GAS 架构的核心原则：利用本地预测（`LocalPredicted`）消除感官延迟，通过标签（Tags）与属性（Attributes）的同步替代繁重的能力对象复制，并采用严格的服务器端安全策略来防御潜在的作弊行为。此外，本报告还将深入探讨属性系统（Attribute System）中 `BaseValue` 与 `CurrentValue` 的数学关系、修改器通道（Modifier Channels）的聚合逻辑，以及数据钳制（Clamping）的正确实现路径。

------



## 第一章：GAS 核心架构演进与实例化策略 (Instancing Policy)



`InstancingPolicy`（实例化策略）是定义 `UGameplayAbility` 对象生命周期的基石。它决定了能力在内存中是以单例形式存在、按需生成的瞬态对象存在，还是仅仅作为类默认对象（CDO）运行。这一决策直接影响 CPU 性能、垃圾回收（GC）压力以及状态持久化的能力。



### 1.1 虚幻引擎 5.5 的重大变革：NonInstanced 的废弃



在 GAS 的发展史上，UE 5.5 版本引入了一个里程碑式的变更：`NonInstanced`（非实例化）策略被正式标记为废弃（Deprecated）1。



#### 1.1.1 历史背景与设计初衷



在此之前，`NonInstanced` 策略常被视为高性能的“圣杯”。在这种模式下，`GameplayAbility` 不会生成任何 `UObject` 实例，所有逻辑直接运行在类默认对象（Class Default Object, CDO）上。由于不产生对象分配（Allocation）和初始化开销，它被广泛推荐用于高频调用的简单技能，如 MOBA 游戏中的普通攻击或小兵的 AI 行为 3。



#### 1.1.2 废弃的深层原因



然而，Epic Games 的工程团队最终决定废弃这一模式，核心原因在于其“易错性”（Error-Proneness）和网络同步的不确定性 1。

- **状态管理的混乱**：由于 CDO 是全局共享的单例，开发者无法在 Blueprint 中安全地使用成员变量。任何对变量的写入都会污染全局状态，导致所有使用该技能的角色出现逻辑错误。虽然 GAS 提供了 `InstancedAbilities` 的替代方案，但许多开发者仍因误用成员变量而导致难以调试的 Bug。
- **复制时序的不可预测性**：在网络环境中，属性复制（Property Replication）的时序与能力激活 RPC（Remote Procedure Call）的时序往往难以严格对齐。当开发者试图在一个并不真正存在的“实例”上进行属性复制时，引擎的底层处理变得异常复杂且不可靠。



#### 1.1.3 迁移路径



随着 `NonInstanced` 的废弃，现代 GAS 架构的“默认”选择被迫发生了转移。开发者必须在 `InstancedPerActor` 和 `InstancedPerExecution` 之间做出选择。这一变更迫使开发者重新评估内存预算，因为原本“零内存成本”的技能现在必须分配为实际对象。



### 1.2 实例化策略深度对比



| **实例化策略 (Instancing Policy)** | **对象生命周期 (Object Lifecycle)** | **状态持久性 (State Persistence)** | **性能特征 (Performance Profile)**          | **典型应用场景**                          |
| ---------------------------------- | ----------------------------------- | ---------------------------------- | ------------------------------------------- | ----------------------------------------- |
| **InstancedPerActor**              | 授予时创建，移除时销毁              | **高**：成员变量在激活间持久保留   | **内存占用高**，CPU 开销低（无频繁 GC）     | 绝大多数主动技能、Lyra 示例项目标准配置 3 |
| **InstancedPerExecution**          | 激活时创建，结束时销毁              | **低**：每次激活重置状态           | **CPU 开销高**（频繁分配/回收），内存占用低 | 极低频、需要绝对纯净状态的特殊技能        |
| **NonInstanced**                   | 无实例（运行于 CDO）                | **无**：完全依赖上下文传递         | **最高**（无对象开销）                      | **已废弃 (UE 5.5)**，不推荐新项目使用 2   |





#### 1.2.1 Instanced Per Actor：现代标准



在 UE 5.5 之后，`InstancedPerActor` 成为了事实上的工业标准。

- **机制**：当 Ability System Component (ASC) 调用 `GiveAbility` 时，引擎会实例化该 Ability 类的一个对象，并将其指针存储在 ASC 的 `ActivatableAbilities` 列表中。这个对象会一直驻留在内存中，直到能力被移除。
- **优势**：
  - **状态保存**：由于对象常驻内存，开发者可以使用成员变量来存储跨激活周期的状态。例如，一个“三连击”技能可以使用一个 `ComboCount` 整数变量来记录当前连击段数，而无需依赖复杂的 Gameplay Tags 或外部存储。
  - **RPC 稳定性**：网络 RPC 需要一个稳定的目标对象。在 `InstancedPerActor` 模式下，客户端和服务器上的 Ability 对象实例是稳定存在的，这极大降低了 RPC 因“目标对象未找到”而丢失的风险。
  - **任务绑定**：对于需要长时间运行的 `AbilityTask`（如 `WaitTargetData` 或 `WaitInputPress`），持久化的实例提供了安全的绑定环境，防止委托（Delegate）在任务完成前失效 3。
- **代价**：每个角色身上的每个技能都会占用几 KB 到几百 KB 的内存。在大型 MMO 或大逃杀游戏中，如果 100 个角色每人拥有 20 个技能，这意味着数千个 UObject 的内存开销。因此，在使用此策略时，保持 Ability 类的“轻量化”至关重要，避免在 Ability 中声明大量庞大的数据结构。



#### 1.2.2 Instanced Per Execution：特殊用途



- **机制**：每次调用 `TryActivateAbility`，系统都会从类模板生成一个新的对象。一旦 `EndAbility` 被调用，该对象即被标记为垃圾回收（Pending Kill）。
- **风险**：这种策略会产生巨大的 GC 压力。如果用于高频技能（如每秒射击 10 次的机枪），每秒将产生 10 个待回收的 UObject，这将导致严重的 CPU 峰值和帧率抖动 3。
- **适用性**：仅适用于那些极其罕见、且必须保证“全新状态”的技能，例如“复活”或“终极技能”，但这通常也可以通过在 `InstancedPerActor` 的 `OnAvatarSet` 或 `ActivateAbility` 中手动重置变量来实现。

------



## 第二章：网络复制策略与带宽优化 (Replication Policy)



`ReplicationPolicy` 决定了 `UGameplayAbility` 对象本身是否通过网络进行序列化同步。这是 GAS 开发中最容易产生误解的配置项之一。



### 2.1 “Replicate” (复制) vs “Do Not Replicate” (不复制)





#### 2.1.1 误区：必须开启复制才能联网



许多初学者错误地认为，为了让技能在多人游戏中工作，必须将 `ReplicationPolicy` 设置为 `Replicate`（是）。这是一个严重的架构错误 4。

- **真相**：即使设置为 `Do Not Replicate`，GAS 依然拥有完整的网络同步能力。能力的激活是通过 ASC 的内部 RPC（如 `ServerTryActivateAbility`）进行的，而不是通过 Ability 对象的复制。
- **Lyra 的选择**：在 Epic 官方的 Lyra 示例项目中，绝大多数核心技能（包括射击、跳跃、冲刺）的 `ReplicationPolicy` 都被设置为 `Do Not Replicate` 4。



#### 2.1.2 策略详解：Do Not Replicate



这是现代 GAS 项目的推荐默认设置。

- **机制**：服务器和客户端各自拥有该 Ability 的实例（如果是 `InstancedPerActor`）。当技能激活时，客户端通过 Prediction Key 机制通知服务器执行。服务器执行后，通过复制 **Gameplay Tags**（游戏标签）、**Gameplay Cues**（游戏提示）和 **Attributes**（属性）的变化来通知其他客户端（Simulated Proxies），而不是同步 Ability 对象本身的内部变量。
- **优势**：
  - **带宽节省**：避免了对 Ability 对象进行持续的网络属性扫描和序列化。在拥有大量技能的场景下，这对带宽的节省是巨大的。
  - **解耦表现**：其他客户端不需要知道“你正在运行 Fireball Ability”，它们只需要知道“你身上有了 `State.Casting` 标签”以及“播放了 `GameplayCue.Fireball.Launch` 粒子效果”。



#### 2.1.3 策略详解：Replicate



- **机制**：将 Ability 对象视为一个标准的 Replicated Subobject。任何在 Blueprint 中标记为 Replicated 的变量都会从服务器同步到拥有该技能的客户端。
- **废弃警告**：Epic Games 明确指出，在 UE 5.5 中，**强烈不建议**在 Ability 上使用复制属性，并且这一功能已被视为废弃 1。这是因为 Ability 的激活本身是基于 RPC 的，而属性复制是基于快照的，两者之间的时序极难对齐，容易导致客户端在 Ability 激活时尚未收到关键变量的更新。
- **残留用例**：极少数情况下，如果技能有极其复杂的内部状态需要精确同步给客户端 UI（例如一个蓄力条的精确浮点数值），且无法通过 AttributeSet 实现时，可能会考虑此选项，但这通常被视为“代码坏味道”（Code Smell）。



### 2.2 ASC 的复制模式 (Replication Mode)



`ReplicationPolicy` 仅控制 Ability 对象本身，而 Ability System Component (ASC) 的 `ReplicationMode` 则决定了 Gameplay Effects 和 Tags 如何在网络中传播。为了配合 `Do Not Replicate` 的 Ability 策略，ASC 的配置至关重要 3。

| **复制模式 (Replication Mode)** | **行为描述**                                           | **适用对象**   | **推荐场景**         |
| ------------------------------- | ------------------------------------------------------ | -------------- | -------------------- |
| **Full**                        | 复制所有 Gameplay Effects (GE) 到所有客户端。          | 单人游戏角色   | 单机游戏             |
| **Mixed**                       | GE 只复制给 Owner；**Tags** 和 **Cues** 复制给所有人。 | 玩家控制的角色 | 多人联机 (MOBA/FPS)  |
| **Minimal**                     | 不复制 GE；**Tags** 和 **Cues** 复制给所有人。         | AI 控制的角色  | 多人联机中的 AI/小兵 |

深度分析：

在多人游戏中，玩家控制的角色应设置为 Mixed 模式。

- **对 Owner (自己)**：我需要知道我身上的每一个 Buff/Debuff 的详细信息（持续时间、层数），以便 UI 显示。

- 对 Proxy (别人)：我不需要知道敌人身上具体的 GE 数据结构，我只需要知道他是否有 State.Stunned 标签（以便播放晕眩动画）或 GameplayCue.Burn（以便播放燃烧特效）。

  这种区分极大地优化了网络流量，使得成百上千个 GE 的同步变得轻量化。

------



## 第三章：网络执行策略与预测机制 (Net Execution Policy)



`NetExecutionPolicy` 定义了技能在网络环境中的执行权责：谁来发起？谁来执行？客户端是否可以“先斩后奏”？这是决定游戏手感（Game Feel）的关键变量。



### 3.1 策略枚举值解析





#### 3.1.1 Local Predicted (本地预测)



这是 FPS 和 ACT 游戏中最关键的策略 3。

- **工作流**：
  1. **客户端**：玩家按下按键，Ability **立即**在本地激活。客户端生成一个 **Prediction Key**（预测密钥）。
  2. **RPC**：客户端发送 `ServerTryActivateAbility` RPC 给服务器，附带该 Key。
  3. **服务器**：收到请求，验证条件（如冷却、消耗、Tag限制）。
     - **通过**：服务器执行 Ability，并将该 Prediction Key 标记为“确认”。服务器产生的属性变化（如扣除法力）会带有该 Key。
     - **拒绝**：服务器发送拒绝消息。客户端触发 **Rollback**（回滚），撤销本地的所有预测性修改（如恢复法力值、取消动画）。
- **意义**：它消除了往返延迟（RTT）。玩家感觉操作是“零延迟”的。如果没有预测，玩家按下“开火”，必须等待 100ms（Ping 值）后才能看到枪口火焰，这在竞技游戏中是不可接受的。



#### 3.1.2 Server Only (仅服务器)



- **工作流**：客户端按下按键 -> 发送 RPC -> 服务器执行 -> 结果复制回客户端。
- **特点**：客户端本地不运行任何逻辑，直到服务器数据传回。
- **适用场景**：对延迟不敏感、但对安全性要求极高的交互。例如：打开战利品箱、与商人交易、全图范围的“终极审判”技能。在这些场景下，预测可能导致严重的逻辑冲突（如物品被复制），且毫秒级的延迟是可以容忍的 7。



#### 3.1.3 Local Only (仅本地)



- **工作流**：仅在客户端执行，服务器完全不知情。
- **适用场景**：纯粹的 UI 交互（如打开背包界面）、本地教程提示、仅对本地玩家可见的装饰性特效。**严禁**用于涉及 Gameplay 状态（如扣血、加 Buff）的逻辑，否则会导致客户端与服务器状态永久脱节。



#### 3.1.4 Server Initiated (服务器发起)



- **工作流**：服务器主动通过 RPC 命令客户端运行某个 Ability。
- **适用场景**：通常用于受击反应（Hit Reaction）或被控制效果（CC）。例如，Boss 释放冲击波，服务器判定玩家被击中，强制玩家运行“击退”Ability。



### 3.2 预测与复制的辩证关系



一个常见的困惑是：“如果我用了 Local Predicted，是否需要把 Replication Policy 设为 Replicate？”

答案是否定的 5。

在 LocalPredicted 模式下，客户端跑的是自己的“本地实例”，服务器跑的是“服务器实例”。两者通过 Prediction Key 进行逻辑上的握手，而不是通过对象数据的硬性复制。开启对象复制反而会干扰预测系统，造成带宽浪费和潜在的逻辑冲突。

------



## 第四章：网络安全与反作弊体系 (Net Security Policy)



在 C/S 架构中，客户端永远是不可信的。GAS 提供了 `NetSecurityPolicy` 来防御非法调用。



### 4.1 安全策略枚举值 

3



1. **ClientOrServer**：客户端或服务器均可发起激活。这是默认设置，适用于大多数玩家主动释放的普通技能。
2. **ServerOnlyExecution**：**严禁客户端发起**。如果客户端试图调用 `TryActivateAbility`，服务器将直接忽略该请求。
   - **应用**：用于由系统逻辑触发的技能，如“被动回血”、“死亡逻辑”、“升级奖励”。防止外挂通过伪造包来强制触发这些不该由玩家控制的逻辑。
3. **ServerOnlyTermination**：客户端可以发起激活，但**不能主动取消**。
   - **应用**：用于“承诺式”技能（Committed Abilities）。例如一个需要持续施法的强力攻击，一旦开始就必须完成，防止玩家利用取消机制来卡动画（Animation Cancel）或规避硬直。
4. **ServerOnly**：完全由服务器控制，客户端既不能激活也不能取消。



### 4.2 远程取消漏洞 (Server Respects Remote Ability Cancellation)



这是一个布尔值配置，控制当客户端取消技能时，服务器是否应该跟随取消 3。

- **默认行为**：通常建议设为 **False**。
- **风险分析**：如果设为 True，拥有高延迟或使用“Lag Switch”（断网开关）的作弊者可以利用这一点。
  - *场景*：玩家激活一个强力技能，本地预测播放了攻击动画。在服务器即将扣除法力值或计算冷却的前一瞬间，作弊者发送“取消”包。
  - *后果*：如果服务器“尊重”这个取消，它可能会终止技能的执行（不扣蓝、不进冷却），但客户端可能已经利用预测期造成了某些视觉或逻辑上的混乱。
- **最佳实践**：服务器应当是技能结束的唯一仲裁者。技能应该因为自然播放完毕（EndAbility）或服务器端的逻辑判断（如被打断）而结束，而不应盲目听从客户端的取消指令。



### 4.3 输入复制的陷阱 (Replicate Input Directly)



该选项会强制将按键的按下/松开事件直接复制给服务器 3。

- **分析**：Epic 建议**不要使用**此选项。
- **原因**：它绕过了 ASC 的标准输入绑定流程，容易导致“输入垃圾邮件”（Input Spam）。网络抖动可能导致服务器在短时间内收到多次“按下”事件，如果逻辑未加保护，可能导致技能瞬间连发多次。现代 GAS 推荐使用 `AbilityTask_WaitInputPress` 等任务配合 ASC 的原生输入绑定来处理输入，这样可以利用 Prediction Key 机制过滤掉无效的输入请求。

------



## 第五章：能力生命周期与重触发机制



`RetriggerInstancedAbility` 控制了当一个 Ability 正在运行时，再次尝试激活它会发生什么 1。



### 5.1 重触发逻辑解析



- **True**：系统会先调用当前运行实例的 `EndAbility`，将其终止，然后立即再次调用 `ActivateAbility`。
  - *适用*：某些需要打断自身重新计时的技能。
  - *风险*：如果配合 `InstancedPerExecution` 使用，这会导致旧对象销毁、新对象生成的循环，性能极差。即使是 `InstancedPerActor`，频繁的 End/Activate 循环也可能导致 GameplayEffect 的反复应用与移除，增加计算开销。
- **False**：系统会直接拒绝新的激活请求，直到当前技能执行完毕（调用 `EndAbility`）。
  - *适用*：绝大多数动作技能。例如“挥剑”，在挥剑动作结束前，再次按键不应重置动作，而应被忽略或进入输入缓冲（Input Buffer）。



### 5.2 Lyra 的连发处理方案



在 Lyra 的全自动武器逻辑中，`RetriggerInstancedAbility` 被设为 **False** 11。

- **实现方式**：武器开火技能不是通过“按一次键激活一次技能”来实现的。相反，技能在按下扳机时激活一次，然后进入一个 `While Input Active` 的循环任务。
- **流程**：激活 -> 开火 -> 等待射速延迟 -> 检查输入是否仍按下 -> (是)再次开火 -> (否)调用 EndAbility。
- **优势**：这种模式避免了反复的 Ability 激活开销，将连发逻辑封装在一次 Ability 生命周期内，极其高效且稳定。

------



## 第六章：属性系统数据流与完整性 (Attribute Data Flow)



虽然 `UGameplayAbility` 是逻辑的载体，但数值的核心在于 `FGameplayAttributeData`。



### 6.1 BaseValue vs CurrentValue 的本质区别



`FGameplayAttributeData` 结构体内部存储了两个浮点数：`BaseValue` 和 `CurrentValue` 12。

- **BaseValue (基值)**：属性的“永久”数值。
  - *修改源*：**Instant** (即时) Gameplay Effects。例如：升级增加力量、永久属性药水。
  - *特性*：它是聚合计算的基础。
- **CurrentValue (当前值)**：属性的“临时”数值，是基值经过所有活跃修改器计算后的结果。
  - *修改源*：**Duration** (持续) 和 **Infinite** (无限) Gameplay Effects。例如：+50 移动速度的 Buff。
  - *公式*：$CurrentValue = (BaseValue + \sum Add) \times \prod Multiply / \prod Divide$ 3。



### 6.2 聚合器与修改器通道 (Modifier Evaluation Channels)



为了解决复杂的数值计算顺序问题（例如：装备加成是加在基础值上，还是加在被 Buff 强化后的值上？），UE5 引入了 **Modifier Channels** 13。

- **机制**：开发者可以在 `DefaultGame.ini` 中定义多达 10 个通道（Channel 0-9）。
- **流程**：聚合器会按照通道顺序（0 -> 1 ->... -> 9）依次计算。
  - Channel 0 的计算结果（Base + Mods）会成为 Channel 1 的输入 BaseValue。
  - Channel 1 的计算结果成为 Channel 2 的输入。
- **应用**：可以设定 Channel 0 为“基础属性”，Channel 1 为“装备加成”，Channel 2 为“百分比 Buff”。这样可以确保“+10% 攻击力”的 Buff 是基于“裸体攻击力+装备攻击力”的总和来计算的，而不是仅仅基于裸体攻击力。

------



## 第七章：数据钳制与反应机制 (Clamping & Reaction)



在 GAS 中，保证属性值不越界（如生命值不小于 0，不超过最大值）是 AttributeSet 的责任，而非 Ability 的责任。



### 7.1 PreAttributeChange：显示值的钳制



`PreAttributeChange` 函数在属性值发生变化**之前**被调用 3。

- **目的**：主要用于钳制 **CurrentValue**，以确保 UI 显示正确。

- **局限**：此处的修改**不会永久改变** ASC 中的修改器（Modifier）。它只是改变了本次查询的返回值。这意味着如果重新计算属性（例如添加了一个新 Buff），钳制逻辑必须再次运行。

- **代码示例**：

  C++

  ```
  void UMyAttributeSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue) {
      if (Attribute == GetHealthAttribute()) {
          NewValue = FMath::Clamp(NewValue, 0.0f, GetMaxHealth());
      }
  }
  ```



### 7.2 PostGameplayEffectExecute：逻辑反应与基值钳制



`PostGameplayEffectExecute` 仅在 **Instant** Gameplay Effect 应用后调用 3。

- **目的**：这是处理游戏逻辑（如死亡判断、受击反应）和钳制 **BaseValue** 的最佳位置。
- **应用**：当受到伤害（Instant GE）时，可以在这里将伤害值从 Health 中扣除，并立即将 Health 的 BaseValue 钳制在 [0, Max] 范围内。如果 Health 降为 0，则触发“死亡”Gameplay Event。



### 7.3 Meta Attributes (元属性) 的高级应用



为了处理复杂的伤害逻辑（如护盾抵扣），GAS 引入了 **Meta Attributes** 3。

- **概念**：Meta Attribute 是一个临时的属性（如 `IncomingDamage`），它不代表角色的持久状态。
- **流程**：
  1. Gameplay Effect 不直接修改 `Health`，而是修改 `IncomingDamage`。
  2. 在 `PostGameplayEffectExecute` 中，读取 `IncomingDamage` 的值。
  3. 逻辑判断：如果有护盾（Shield），先从 Shield 中扣除。
  4. 剩余的数值再从 `Health` 中扣除。
  5. 最后将 `IncomingDamage` 清零，等待下一次计算。
- **优势**：这种模式允许开发者在伤害生效前进行复杂的拦截和修改（如护甲减免、元素抗性计算），而不污染实际的 Health 属性。

------



## 结论





虚幻引擎的 Gameplay Ability System 通过高度抽象的策略变量，为多人游戏提供了一套工业级的解决方案。深入分析表明，**InstancedPerActor** 结合 **LocalPredicted** 和 **Do Not Replicate** 已成为构建高性能、低延迟且安全的网络技能系统的黄金标准。

随着 UE 5.5 废弃 `NonInstanced` 策略，GAS 的设计哲学更加明确地指向了**安全性与稳定性的优先**。开发者应摒弃旧有的“对象复制”思维，转而拥抱基于 Prediction Key 的预测模型和基于 Tag/Attribute 的状态同步模型。同时，充分利用 AttributeSet 的 `PreAttributeChange` 和 `PostGameplayEffectExecute` 分离显示层与逻辑层的钳制，配合 Modifier Channels 精细控制数值计算，是构建健壮数值系统的关键。

对于正在开发或维护 GAS 项目的团队，建议立即全面审计现有的策略配置，移除对 `Replicate` 策略的依赖，并重构所有 `NonInstanced` 技能，以确保项目符合未来的引擎演进方向。