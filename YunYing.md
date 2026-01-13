项目名称叫 HeroGAS，设计文档初稿未写完


# 技能拆解
使用GAS框架完成技能
## 云缨技能
```
被动： 枪意·掠火冷却值：0消耗：0 云缨与神兵“掠火”心念相通，释放技能会领悟一层枪意，使用普攻会消耗所有枪意，释放对应的枪意技【坚意·止戈】、【锐意·摧城】和【真意·燎原】，并减少【断月】和【追云】1.5秒冷却时间。枪意技可暴击，受50%攻速影响。

一技能：断月冷却值：9/8.7/8.4/8.1/7.8/7.5消耗：40
云缨举枪向前方上挑后下劈，上挑可造成0.5秒击飞，每段造成180/216/252/288/324/360(+60%额外物理攻击)物理伤害。

二技能： 追云冷却值：10/9.6/9.2/8.8/8.4/8消耗：40
云缨提枪蓄力冲锋，获得最高增加25/30/35/40/45/50%移速，结束时向前突进横扫，对敌人造成180/216/252/288/324/360(+60%额外物理攻击)~360/432/504/576/648/720(+120%额外物理攻击)物理伤害和15/18/21/24/27/30%减速，持续1秒。
蓄力冲锋时间越长，加速越高，结束突进越远，伤害越高。

大招：逐星冷却值：40/36/32消耗：80
云缨引燃“掠火”枪尖，向前方区域来回突击，每段都对敌人造成180/270/360(+60%额外物理攻击)物理伤害和击飞。在突击至最远处时会连续刺击4次，每次造成90/135/180(+30%额外物理攻击)物理伤害。技能释放期间处于霸体状态并减少30/45/60%受到的伤害。
激活技能二段【啸日】，挥动“掠火”向前横扫并留下火径，持续3秒。横扫造成180/270/360(+60%额外物理攻击)物理伤害，火径每0.2秒造成50(+16%额外物理攻击)法术伤害。【逐星】释放期间，可以使用技能第二段横扫四周，打断后续突进造成120(+40%额外物理攻击)的物理伤害并留下圆形火径，持续3秒。

【坚意·止戈】：对周围近距离敌人造成4次物理伤害，释放期间可移动，并获得2秒护盾 。（攻击距离段：主要用来清兵线、野怪、推塔）
【锐意·摧城】：向前突刺3次，每次造成物理伤害并回复生命值（血量越低回复越高） 。（攻击距离长：打消耗，回血）
【真意·燎原】：无视地形跃至空中劈出斩击，对路径敌人造成物理伤害（目标血量低于50%时伤害提升），跳跃期间减少10%伤害 。（超远距离斩击，用来消耗和收割）
```
## 技能机制
抛开技能数值，这里简单拆分需要实现的技能机制，我拆分为：普通技能、蓄力技能、分段技能、需要前置的技能
这样拆分的原因主要技能的呈现效果是否和控制耦合。主要是技能在实现时和UE的 EnhancedInputAction_IA 是否解耦。
### 普通技能
玩家触发后立即执行完整动作序列的技能，其表现逻辑与输入控制解耦，通常表现为“按下即释放”。示例：断月——按下即释放，生成伤害判定区域，完成后进入CD。
- 玩家按下技能键后，角色立即播放技能动画；
- 技能过程中可选是否附带伤害判定体（Hitbox）或特效；
- 动画播放完毕后或按下技能键时，进入冷却（Cooldown）状态；
- 无额外输入要求，不依赖前置状态或连续输入。
实现：
character: Skill1 -> TryActivateAbilities(GA_DuanYue)
GA_DuanYue: activate -> commitabilityCD -> applygameeffect(add Qiangyi) -> spawnactor(BP_AbilityCollison) -> playmontage -> destoryactor -> endability


### 蓄力技能
技能效果根据按键持续时间动态变化，支持短按快速释放与长按蓄力释放两种模式，并可能在蓄力过程中响应其他输入（如移动）以触发附加行为。示例：追云——短按释放快速突进；长按进入蓄力，松开释放强化突进；蓄力中移动触发高速奔跑。
- 短按（Tap）：立即释放基础版本技能；
- 长按（Hold）：进入蓄力阶段，角色可能播放蓄力动画或进入特殊状态；
- 释放按键：根据蓄力时长释放对应强度/形态的技能；
- 蓄力期间输入移动指令：可中断蓄力状态，转为高速移动或其他机动行为（如冲刺）；
- 蓄力过程通常存在最大蓄力上限和最小释放阈值。
实现：
IMC_Skill或IA_E：设置E键触发器，长按，时间阈值为激活蓄力的阈值（这个和设置的动画有关）。
character: enhancedInputAction IA_E
Triggered(上述达到时间阈值触发): DoOnce -> GA_ZhuiYunPre GA_ZhuiYunPre 在播放动画时需要注意要截取时间长度为时间阈值的动画，然后再把后一帧的动画拉到非常长。 如果在播放动画前，角色有速度，则GE_加速。如果在播放动画前，角色没有速度，就正常播放动画。因为有加速且没有播放蓄力动画，自然执行的是行走和奔跑的动画。
Started(按下E就会触发): Reset DoOnce -> GA_ZhuiYunNormal
Complete(松开E键触发): GA_ZhuiYunRush GA_ZhuiYunRush执行冲刺模板
可以看到如果按下E没有到时间触发阈值，就会释放 GA_ZhuiYunNormal，如果达到了就是执行 GA_ZhuiYunPre 和 GA_ZhuiYunRush


### 分段技能
由多个连续子动作组成的技能序列，后续段落需在限定时间窗口内通过重复输入触发，各段可具有不同表现、效果或资源消耗。示例：逐星——首按触发一段冲刺；X秒内再按释放二段火焰爆炸。
- 首次输入触发第一段动作；
- 在预设的时间窗口（Input Buffer Window）内再次按下技能键，可触发第二段（及后续段）；
- 若超时未输入，则技能序列中断，无法继续；
- 各段可独立配置动画、判定、资源消耗及CD逻辑（通常整体共享CD）；
- 可扩展为三段及以上，但需明确段间衔接规则。

### 需要前置的技能
技能的实际表现取决于角色当前所处的状态（如资源层数、连段计数、Buff标记等），同一输入在不同状态下触发不同动作或效果。示例：
无枪意时：普攻为三段连击（1→2→3），受输入窗口限制；
有枪意时：按下普攻根据层数释放对应强化技（1层：止戈，2层：摧城，3层：燎原），并消耗相应层数。
- 基础输入（如普攻键）在不同上下文中映射到不同技能行为；
- 状态通常由先前动作积累（如连击计数、资源叠加）；
- 存在“输入窗口”机制：若超出时间或动作间隔，状态重置；
- 支持多级状态切换（如1层→A技能，2层→B技能，3层→C技能）；
- 状态消耗或保留策略需明确定义（如释放后清空层数）。
实现：
character: Attack -> GA_AttackBasicRouter 以GA_AttackBasicRouter为普通攻击的入口
GA_AttackBasicRouter： 
无枪意，通过 TAG Ability.Attack.Basic 来激活 GA_BasicAttack1/GA_BasicAttack2/GA_BasicAttack3， 这三个资产标签：除了Ability.Attack.Basic 还有对应的 Ability.Attack.BasicX。 在播放的攻击动画中，添加ANS_BasicAttackWindow， 给普攻加上允许攻击窗口。GA_BasicAttack2 GA_BasicAttack3 激活所需标签添加
State.Action.InAttackWindow。以上就可以实现连招攻击了
有枪意，根据枪意值，分别激活 GA_ZhiGe GA_CuiCheng GA_LiaoYuan

## A技能可以打断B技能
技能A里配，取消带标签的能力tag：技能B
## 某技能不能重复释放
GA激活以后，很有可能输入不断，但GA还没有运行到commit CD GE的时候，需要阻止不断释放该技能
技能A里配：技能已拥有标签：技能A，激活阻止标签：技能A

## 冲刺 模板

## GE 加速减速
## ANS_BasicAttackWindow
重写Received Notify Begin 给 owner 添加 State.Action.InAttackWindow 的tag
重写Received Notify End 给 owner 移除 State.Action.InAttackWindow 的tag

## 伤害判定 actor BP_AbilityCollison

## 释放技能不可动 State.Action.Rooted
我是通过添加一个 State.Action.Rooted 来表示这个技能不能移动的时候释放
在 character 的 doMove 添加一个需要不存在 State.Action.Rooted tag 的时候才能输入domove

在需要站立释放的技能中 加上激活已拥有标签：State.Action.Rooted 保证技能释放时没有移动输入

# character
在基础character基础上添加些功能来作为 Hero 基类

`核心 GAS 成员变量`
```
// GAS 的核心管理组件，负责能力、技能、属性修改、游戏玩法效果的统筹管理
TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent;
// 自定义属性集，存储角色的核心属性（如血量、蓝量、移速等，继承自 UHeroAttributeSet）
TObjectPtr<UHeroAttributeSet> AttributeSet;
```

`默认能力列表与能力授予方法`
```
// 可在编辑器中配置的默认能力数组，指定角色出生时自动获得的技能/能力
TArray<TSubclassOf<UGameplayAbility>> DefaultAbilities;
// 蓝图可调用的方法，用于授予角色 DefaultAbilities 中的所有能力
UFUNCTION(BlueprintCallable, Category = "Abilities")
void AddCharacterAbilities();
```

`完善输入系统`
```
DoMove(float Right, float Forward);   // 移动输入的统一处理入口 根据传入的 Right（左右方向值）和 Forward（前后方向值），让角色按照「控制器面向的方向」进行平移移动
DoLook(float Yaw, float Pitch);       // 视角输入的统一处理入口 根据传入的 Yaw（左右视角值）和 Pitch（上下视角值），调整控制器的视角旋转，实现镜头 / 角色朝向的转动
DoJumpStart();                        // 跳跃开始的统一处理入口
DoJumpEnd();                          // 跳跃结束的统一处理入口
```

`保留并规范化第三人称相机组件`
```
// 相机吊杆（控制相机与角色的距离、跟随方式，避免穿墙）
USpringArmComponent* CameraBoom;
// 跟随相机（角色的第三人称视角相机）
UCameraComponent* FollowCamera;
```
# 动作系统
动作系统参照 Lyra 的框架做的，完成了单朝向的四向移动，角色的单向移动（目前设计如此，因为要全方向跑动，所以没有对退后和左右移动做单独的区分），没有做急停pivot(因为没有对应素材)，没有做跳跃相关的动画（设计如此）
## 基本框架
### 动画实例蓝图父类 HeroGASAnimInstance
`FGameplayTagBlueprintPropertyMap GameplayTagPropertyMap`
- 这是 GAS 中用于 将 Gameplay Tag 自动映射到蓝图可读布尔变量 的机制。
- 你可以在 动画蓝图（AnimBlueprint） 中通过该结构体绑定若干 Gameplay Tags（如 Tag 的 State.State.Run 绑定 动画蓝图的布尔变量 GameplayTag_isCombat 、 Tag 的 State.State.Walk 绑定 动画蓝图的布尔变量 GameplayTag_isFree 、 Tag 的 State.State.RunFast 绑定 动画蓝图的布尔变量 GameplayTag_isRunFast ），并指定对应的布尔变量名。
- 当角色的 AbilitySystemComponent (ASC) 上 添加或移除这些 Tag 时，动画实例中对应的布尔变量会 自动更新为 true/false。

`float GroundDistance`
存储角色当前 脚下到地面的距离（单位：厘米），由自定义移动组件 UHeroCharacterMovementComponent 提供。
用于动画系统判断角色是否 即将落地、处于空中、贴地滑行 等状态。每帧在 NativeUpdateAnimation() 中从移动组件同步最新值。

### 动画蓝图继承HeroGASAnimInstance ABP_Mannequin_Base
ABP本应该是播放动画序列。但 Lyra 选择抽象出 LinkedAnimLayer，分别是 动画具体层（ABP_Anim）、动画层（ABP_ItemAnimLayersBase）、动画层接口（ALI_ItemAnimLayers）、动画状态层（ABP_Mannequin_Base）
Lyra 这样做的目的：1. 由动画状态层负责角色的通用基础动画，由动画层负责道具 / 角色专属的动画逻辑 2. 把这些 道具通用动画逻辑 抽象到动画层基类（ABP_ItemAnimLayersBase），新增道具时只需继承这个基类，修改具体动画资源即可，不用重写逻辑。 3. 将动画状态层通过重载 BlueprintThreadSafeUpdateAnimation 保证在动画状态机运行时各参数都从这个SafeThread中取保证线程安全
所以，比如动画图表中移动状态机的 Idle 里的 FullBody_IdleState 只是由 ALI_ItemAnimLayers 提供的一个接口，具体的实现要到 ABP_ItemAnimLayersBase 去找。
我的理解是动画状态层提供速度、位置、动画逻辑转化等并且保证线程安全，然后具体的动画应该怎么执行在动画层中，然后在动画层中把需要播放的动画置为变量然后在动画具体层进行选择。

`动画图表`
两个状态机（移动+主状态）→ Slot合并姿态 → Control Rig做IK/骨骼调整 → 输出最终动画姿态
locomotion 的状态机负责控制角色的站立、启动、行动、停止各状态的切换，MainStates 的状态机负责控制角色跳跃启动、跳跃浮空、跳跃着陆的状态切换。
“Slot 'DefaultSlot'” 是 UE 动画蓝图的插槽节点，作用是将多个动画源的姿态合并到同一个 “插槽组（DefaultGroup）”—— 这里把 “移动状态机” 和 “主状态机” 的姿态合并成一个统一的 “Source” 姿态，为后续处理做准备。
在执行完对角色执行一个ControlRig，负责对合并后的姿态做最终的骨骼 / IK 调整，使用UE第三人称的默认 CR_Mannequin_FootIK。
![alt text](picture/locomotionFSM.png)


`BlueprintThreadSafeUpdateAnimation`
需要 UpdateLocationData UpdateRotationData UpdateVelocityData UpdateAccelerationData UpdateRootYawOffset UpdateJumpData IsFirstUpdate
这几个函数的具体做法大多是从 getowneractor 或者 getmovementcomponet 里直接获得

### ABP_ItemAnimLayersBase
序列播放器：
- On Update（更新）：每帧动画更新阶段，在初始更新与变为相关之后，持续执行
- On Initial Update（初始更新）：节点首次被标记为相关（Relevant）后的第一个更新帧，仅执行一次
- On Become Relevant（变为相关）：节点从不相关（Irrelevant） 切换为相关（Relevant） 时立即触发（通常在 Update 前一帧或同一帧的前期阶段）

ALI_ItemAnimLayers 暴露的需要实现的函数有

`FullBody_IdleState`
- 通过序列播放器（在更新时UpdateIdleAnim）
- 通过去动画状态层取 GameplayTag 来判断播放哪种动画序列（Idle_Combat/Idle_Free）
- 通过 Set Sequence with Inertial Blending 切换动画序列时，用「惯性混合」替代传统的线性 / 曲线交叉淡入淡出（Cross-Fade），在保持运动连续性

`FullBody_Starttate`
- 通过序列播放器（在更新时UpdateStartAnim， 变相关时SetUpStartAnim）

`FullBody_CycleState`

`FullBody_StopState`


###  Set Sequence with Inertial Blending 
惯性混合的本质是用物理运动方程模拟骨骼的 “惯性”，在过渡期间生成符合动量的中间姿势，再平滑衔接至目标序列，而非直接在两个关键帧间插值。
- 触发与参数捕获：调用 Set Sequence with Inertial Blending 时，捕获当前帧的源序列最终姿势、骨骼速度（根运动 / 局部骨骼）、目标序列、混合时长
- 停止源序列采样（性能关键）：启用惯性混合后，源序列不再被采样和计算；传统混合仍需持续采样源序列
- 惯性过渡生成：动画线程每帧按惯性方程计算中间姿势（位置 / 角度）。目标序列从起始帧开始采样，生成目标姿势。按当前过渡进度（t/t1​），将惯性中间姿势与目标序列姿势做加权融合（过渡前期以惯性为主，后期以目标序列为主）
- 过渡完成：当 t=t1​ 时，惯性混合结束，完全切换到目标序列的正常播放；此时序列播放器的累计时间（Accumulated Time）与目标序列同步
### Set Playerate to Match Speed
动态调整动画序列的播放速率（Play Rate），让动画的 “原生移动速度” 和角色的实际物理移动速度完全匹配，彻底消除滑步（比如动画是慢跑，角色实际是快跑，会出现脚跟不上身体的滑步）；同时支持匀速、加速、减速场景的平滑过渡。
- PlayRate=Character_Actual_Speed​ / Animation_Native_Speed Character_Actual_Speed：角色在世界空间的实际移动速度（从 Character Movement 组件获取） Animation_Native_Speed：动画序列的原生根运动速度（在动画编辑器中烘焙，单位 m/s）
- 每帧（动画线程On Update阶段）获取角色当前物理速度；
- 读取当前动画序列的原生根运动速度（提前烘焙或通过蓝图配置）；
- 按公式计算目标 PlayRate，通过Set Play Rate函数应用到序列播放器；
- 添加速率平滑（如 Exponential Interpolation），避免速率突变导致动画抖动；限制速率范围（如 0.5~2.0），防止极端速率导致动画变形
### Advance Time by Distance Matching
不按时间（DeltaTime）推进动画，而是按角色实际移动的距离推进动画时间，让动画的根运动位移和角色的物理位移精准绑定。核心解决 “时间推进和距离脱节” 问题（比如运动匹配中，角色轨迹和动画轨迹偏差导致滑步）。
- ΔTimeAdvance​=Animation_Stride_Length/(Animation_Frame_Duration/Character_Actual_Distance​)Character_Actual_Distance：角色当前帧实际移动的距离（物理位移）Animation_Stride_Length：动画序列中每一步的长度（烘焙值） Animation_Frame_Duration：动画序列每帧的时长（1 / 帧率）
- 触发时机：通常在On Update或运动匹配的姿态选择后执行；
- 计算当前帧角色的物理位移（从 Character Movement 获取，或通过轨迹计算）；
- 读取选中动画序列的根运动速度 / 步幅长度；
- 按公式计算需要推进的动画时间ΔTimeAdvance​；
- 调用序列播放器的Advance Time函数，直接推进动画时间（跳过默认的 DeltaTime 推进）；
- 同步动画的根运动到角色物理位置，完成闭环。
### Stride Warping
在不修改原始动画的前提下，动态调整角色的步幅长度和脚步落地位置，让脚的移动和地面、角色速度精准匹配，解决 “不同速度下步幅不一致”“地面不平整导致穿模 / 悬空”“转向时脚步方向偏差” 的问题。通常和 IK（反向运动学）结合使用.
分离 “上半身动画” 和 “腿部 / 脚步动画”，上半身保持原始动画，腿部按物理速度 + 地面高度 + 目标步幅重新计算 IK 目标位置，实现步幅扭曲。
- Target_Stride_Length=Character_Speed *Animation_Stride_Duration Character_Speed：角色实际速度 Animation_Stride_Duration：原始动画中每一步的时长（固定值）
- 采样原始动画，获取腿部骨骼的初始位置和关节角度；
- 计算目标步幅长度（按速度）和脚步落地位置（通过射线检测获取地面高度）；
- 计算腿部 IK 的目标偏移（从原始位置到目标落地位置的向量）；
- 应用 IK 求解，扭曲腿部骨骼的位置和角度，实现步幅调整；
- 混合上半身原始动画和腿部扭曲后的动画，输出最终姿态。
### Orientation Warping
在不修改原始动画的前提下，动态调整角色的根骨骼 / 上半身朝向，让角色的朝向和目标方向（如移动方向、瞄准方向） 精准匹配，同时保持下半身动画的流畅性，解决 “转向时动画生硬”“上半身需要瞄准但下半身保持移动” 的问题。
分离 “角色朝向” 和 “动画姿态”，通过旋转偏移计算，扭曲根骨骼或上半身骨骼的朝向，实现 “上半身转向，下半身保持移动动画” 的效果，本质是对骨骼旋转的局部调整。
- 采样原始动画，获取角色根骨骼 / 上半身骨骼的原始旋转；
- 获取目标方向（从 Character Movement 或控制器获取）；
- 计算 Yaw 偏移角度，限制在合理范围（如 ±90°，避免过度扭曲）；
- 对目标骨骼应用旋转偏移（根骨骼或上半身骨骼，可通过权重控制影响范围）；
- 混合扭曲后的骨骼姿态和原始动画姿态，输出最终姿态。