# UE游戏开发 – 游戏DEMO 
项目描述：实现俯视角下的迷宫冒险游戏 
1. 实现角色的移动，释放指向性技能和放置性技能，冲刺，交互。 
2. 通过行为树实现敌人的对主角的追逐、攻击。 
3. 实现WFC算法的地图、出生点位、敌人分布的随机生成。 
4. 实现进入游戏的简单关卡跳转UI。
演示视频
<video controls src="demo.mp4" title="Title"></video>

# 玩家角色
以UE默认的TopDown小人为基础进行调整，在角色 character 上实现角色能完成的各类动作，和各类数值初始化，在 controller 上实现各类技能的抽象逻辑
## 角色移动
### character
骨骼网格体：SKM_Manny_Simple
动画类：ABP_Unarmed
移动组件：character 继承的移动组件
### controller
EventBeginPlay -> 映射操作表 IMC
IA_SetDestination -> 获取鼠标位置 -> move to 鼠标位置
## 技能释放1
技能效果：往前挥击，向前发一个冲击波，冲击波碰撞到敌人造成伤害。长按和短按有技能指示器，技能指示器以角色为圆心
### character
CE_AttackForward -> 角色转向 -> 播放动画蒙太奇(角色动作为向前刺)
### controller
IA_Spell_WaveForward(triggered/started) -> 在角色位置生成技能Range指示器 -> 在鼠标位置方向生成扇形技能Location指示器
IA_Spell_WaveForward(Canceled) -> 摧毁指示器
IA_Spell_WaveForward(Completed) -> 摧毁指示器 -> 判断是否在CD -> CE_AttackForward -> 生成WaveForward
## 技能释放2
技能效果：施法，生成一个小飞弹飞到指定位置，指定位置生成一个魔法阵，敌人踩到魔法阵，魔法阵发生爆炸，对敌人造成减速
### character
CE_SpellBoom -> 播放动画蒙太奇(角色动作为举高手臂)
### controller
IA_Spell_Boom(triggered/started) -> 在角色位置生成技能Range指示器 -> 在鼠标位置生成圆形技能Location指示器
IA_Spell_Boom(Canceled) -> 摧毁指示器
IA_Spell_Boom(Completed) -> 摧毁指示器 -> 判断是否在CD -> CE_SpellBoom -> 生成 Projectile (后续的技能逻辑在技能实体中实现)
## 冲刺
技能效果：角色向指定方向冲刺
这块做的不是很理想，我设置了角色移动基于附件根，所以只做了指定方向冲刺，而没有做指定位置冲刺
### character
CE_Movement_Rush -> 角色转向 -> 播放动画蒙太奇(角色动作为滑铲，没有找到冲刺资源)
### controller
IA_Movement_Rush(triggered/started) -> 在角色位置生成技能Range指示器 -> 在鼠标位置方向生成扇形技能Location指示器
IA_Movement_Rush(Canceled) -> 摧毁指示器
IA_Movement_Rush(Completed) -> 摧毁指示器 -> 判断是否在CD -> CE_Movement_Rush
## 交互
在该demo中，在传送阵中施法，然后，传送到传送阵指定位置
### controller
IA_Movement_Interactive -> 在角色位置 生成球形扫描器收集球形内所有的Objects MultiSphereTraceForObjects -> 检测是否是传送阵类型 -> 是，判断是传送阵的 start/end 端 -> 得到最终要传送的目的地 -> CE_SpellBoom（用于播放动作）-> SetActorLocation 将角色传送 
如果传送阵是终点传送阵，则宣布游戏结束

值得注意的点：是选择 MultiSphereTraceForObjects 而不是在传送阵那里用碰撞体做检测，是为了减少开销，后者只要传送阵actor存在相当于在每个event tick都在执行检测，开销很大且没必要，换成角色每次进行 Interactive 才进行检测就开销很小了
## 数值初始化
### character
EventBeginPlay -> set HP_current, HP_max, HP_dead
## 小地图绑定
### character
EventBeginPlay -> 给 Player0 生成一个 MiniMap_Widget
## 血量显示
### character
EventBeginPlay -> 给 Player0 生成一个 HP_Widget

# 敌人角色
以UE默认的TopDown小人为基础进行调整，通过行为树自动追踪玩家，攻击玩家，各类数值初始化
## 追踪玩家
### controller
添加AI感知组件 AIPerception: 感知到目标更新时触发->检查被感知的 Actor 是否拥有 Player 标签 -> 如果看到玩家：清楚丢失玩家的计时器，在黑板中设置 HasLineOfSight 为 True。
                                                                                    | -> 如果丢失玩家：设置一个计时器，计时器触发后，在黑板中设置 HasLineOfSight 为 False。
### 行为树
如果能看到玩家->执行【追逐分支】:
        1. 转向面对玩家
        2. ChasePlayer_BTT 任务：改变敌人角色移动速度
        3. 冲向玩家
否则 -> 执行【巡逻分支】:
        1. FindRandomPatrol_BTT 任务：找个随机点 & 慢速移动
        2. 走到那个点
        3. wait ，然后重复
## 数值初始化
### character
EventBeginPlay -> set HP_current, HP_max, HP_dead
## 攻击玩家
### character
设置碰撞组件 -> Box碰撞体积（发生Overlap）-> 播放动画蒙太奇 -> 生成ShortWave -> 短暂延迟，摧毁Actor

# 技能实体
## Indicator
Decal: 提供材质接口
Scale: 在平面上的默认值
各子类的区别主要是材质上的不同
### Indicator的材质
材质域：延迟贴花
混合模式：半透明
生成两个圆，两个相减得到圆环。再生成一个圆，设置 hardness 改变边缘模糊程度，与最大的圆相减。得到外面有实线中间带模糊的图案。作为材料的半透明度。
选择基础颜色为蓝色，作为材料自发光。
### Indicator_Range
材质:上述默认材质
Scale:500
### Indicator_MouseLocation
材质:上述默认材质，基础颜色调整为红色，通过调整上述圆的半径、hardness、自发光强度等，改得更锐利一点
Scale:150
### Indicator_Fan
材质:用2D向量/UV坐标转换为径向值（Radial Coordinates），再取只取 1/4 圆周的角度范围，然后用1-x做另外一般，在不透明度设置之前与之前的圆环相乘，形成扇形的效果
Scale:450
## WaveForward
粒子效果通过 Niagara 粒子系统实现，冲击波的特效
EventBeginPlay -> 时间轴(Update) Lerp -> 从角色位置到 角色位置+MaxDistance SetActorLocation
                    | (Finish) -> 摧毁Actor
Sphere碰撞体积（发生Overlap）-> 检测是否是敌人 -> 短暂延迟-> SetCollsionEnable -> ApplyDamage -> 短暂延迟 ->SetCollsionDisable
值得注意的点：选择短暂开启碰撞检测的原因是，避免飞行道具穿过敌人产生多次判定，只能先用这种土办法解决
## Projectile
粒子效果通过 Niagara 粒子系统实现，小火苗的特效
EventBeginPlay -> 时间轴(Update) Lerp -> 从角色位置到 TargetLocation
                    | (Finish) ->在 TargetLocation 生成 Boom -> 摧毁Actor
## Boom
粒子效果通过 Niagara 粒子系统实现, 法阵的特效，粒子的 Lifetime 跟随系统
EventBeginPlay -> 在 Boom 的持续时间结束后 摧毁Actor
Sphere碰撞体积（发生Overlap）-> 检测是否是敌人 -> 关闭Niagara效果 -> 生成Blast
## Blast
粒子效果通过 Niagara 粒子系统实现，爆炸的特效
EventBeginPlay -> 短暂延迟-> SetCollsionEnable  -> 短暂延迟 ->SetCollsionDisable
Sphere碰撞体积（发生Overlap）-> 检测是否是敌人 -> ApplyDamage -> 摧毁Actor
## ShortWave(敌人用)
Box碰撞体积（发生Overlap）-> 检测是否是玩家 -> ApplyDamage

# 地图生成器
使用 WFC（波坍缩函数）随机生成迷宫地图
## WFC生成地图
生成逻辑比较复杂，选择使用C++类来实现，通过13个基础的组件，生成随机的迷宫地图
### 核心数据结构
- EProtoType - 原型类型枚举，定义了13种不同的瓦片类型（VE_0到VE_12）
- FProto - 瓦片原型结构体，定义了四个方向的邻居约束（PX/NX/PY/NY方向允许的瓦片类型）
- FWFCCell - 波函数单元格，PotentialProto：当前单元格可能的所有瓦片原型，bCollapse：是否已坍缩（确定最终类型），bDirty：传播标记，防止重复计算
### 核心算法
1. 初始化WFC系统
2. 随机或按顺序选择单元格调用 CollapseCellByIndex
3. 算法会自动传播约束，生成符合规则的图案
4. 通过 GetCollapseProtoCopy 获取每个单元格的最终类型
## 传送点生成
因为WFC生成的地图是随机的，连通性无法保证，为了让随机地图保证可用性，需要对不同的连通区域进行连接
### 核心算法
1. 将每个波函数单元块，抽象为2*2的空间，高耸的部分为墙，凹陷的部分为路，在WFC初始化时，生成抽象的路网地图
2. 通过并查集获得连通区域
3. 每个连通集的最后一个和下一个连通集的第一一个分别设置为传送点的起始。保证所有区域连通
4. 在起始区域，生成两个碰撞体，玩家进行互动时，传送至里玩家更远的那个点
## 初始点生成
选择第一个连通集的第一个点
值得注意的是：因为并查集的算法是行或列+1这样遍历的，从（0,0）开始，所以基本还是可以保证传送点是不断向迷宫内部填充的，初始点和终点都会是逻辑上比较远的距离
## 终点生成
选择最后一个连通集的最后一个点
## 敌人出生点
选择连通集超过一定数额的区域，随机在该区域放置敌人

# UI
## Menu
在进入游戏时，默认StartMap的关卡，展示开始界面，在开始以后跳转到GameMap的关卡
## MiniMap
在Widget放置一个图像，图像来自在玩家角色头顶正上方放置一个场景捕获组件SceneCaptureComponent2D，场景捕获的TextureRender作为用户界面材质的最终颜色。
## HP
Widget，放置进度条，通过HP_cruent/HP_max实现

# 自我评价
我感觉我这个demo算是到头了，我做了地图的生成，玩家角色的移动、发波、冲刺，敌人追过来打我，一些基础UI。
因为最开始只是熟悉UE，所以是纯基础的开发，没有用GAS的框架，很多角色能力的实现就很粗糙，现在想扩展已经有点难了。
然后最开始没有做游戏设计，游戏性没有考虑，也没有继续优化的动力了。
目前还没有做联机。
后续考虑重新做一个用GAS框架来设计角色的demo