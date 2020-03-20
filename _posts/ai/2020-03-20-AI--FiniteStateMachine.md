---
layout: article
title:  "AI--FiniteStateMachine"
categories: ai
image:
    teaser: /in-post/ai/2020-03-20-AI--FiniteStateMachine/DefaultImg.jpg
---

# 目录

[TOC]

# 一、写在前面

本文着重实现一个高度可配置的有限状态机（FiniteStateMachine）；然后借鉴 [Unity官方教程](https://learn.unity.com/tutorial/5c515373edbc2a001fd5c79d?tab=overview) 将其运用于 Unity 中。

有限状态机的相关概念就不做介绍了，了解即可。

工程地址：[网盘](链接：https://pan.baidu.com/s/1NxqumpoqEve3ywMQdfP4ow)（提取码：79zj）

# 二、FiniteStateMachine

## 1.FSM 框架结构

如下图所示：

![picture0](https://huskytgame.github.io/images/in-post/ai/2020-03-20-AI--FiniteStateMachine/ScreenShot000.png)

``StateController`` 状态管理器：

- AI 的核心
- 将 AI 系统的 **初始状态** 注入
- 设置好 AI 的 **相关数据**（此后会将 ``StateController`` 注入到各个模块以提供 AI 相关数据）
- 状态管理器的帧函数 ``UpdateAI`` 由外部（继承自 Monobehavior）的类调用

``StateBase`` 状态基类：处理各个状态的公共逻辑：

- 存储 *状态* 的 **行动集合**（每个状态可能有多个行动组合而成）
- 存储 *状态* 的 **转换条件集合**（每个状态可能有多种转换条件）
- （必选）在子类中添加 行动集合 ``AddActions``
- 在外部添加 转换条件集合 ``AddTransition``
- 在内部处理状态帧函数``UpdateState``：执行动作``DoActions`` + 检验每个转换条件``CheckTransitions``
- （可选）实现状态的生命周期函数：进入状态的``DoOnBefore`` + 退出状态的``DoOnExit``

``ActionBase`` 动作基类：

- 将每个状态下的行为分解为各个动作的集合，有利于逻辑复用，更加模块化的处理也有利于 AI 行为的扩展。

``Transition`` 转换条件：

- 每个转换条件由：一个 **转换决策**``Decision`` 、一个 **转换成功后指向的状态**``TargetState`` 组成

``DecisionBase`` 转换决策基类：

- 转换条件 ``Transition`` 的核心组成部分
- 用于处理状态之间转换的核心逻辑

## 2.框架代码

### StateController

````csharp
/****************************************************
	文件：StateController.cs
	作者：HuskyT
	邮箱：1005240602@qq.com
	日期：2020/3/20 16:19:33
	功能：状态管理器（AI Brain）
*****************************************************/

using System;
using UnityEngine;

namespace FiniteStateMachine
{
    /*
     * 状态管理器
     * AI Brain
     * 作用：AI 的状态控制器（核心类）
     */
    public class StateController
    {
        private StateBase mCurrentState;
        /// <summary>
        /// AI 是否为激活状态
        /// </summary>
        private bool mAIActivate;
        /// <summary>
        /// 当前状态过去的时间（当前状态的计时器）
        /// </summary>
        private float mElapsedTime;

        public StateController(StateBase originalState, params object[] args)
        {
            mCurrentState = originalState;
            mAIActivate = true;
            mElapsedTime = 0.0f;
            //设置 AI 初始数据

            Debug.Log("AI 初始化完毕");
        }

        public void SetupAI(bool aiActivate, params object[] args)
        {
            mAIActivate = aiActivate;
            //设置 AI 数据
        }

        public void UpdateAI()
        {
            if (mAIActivate == false) return;
            mCurrentState.UpdateState(this);
        }
        public void TransitionToState(StateBase targetState)
        {
            mCurrentState.DoOnExit(this);
            mElapsedTime = 0.0f;//计时器清零
            mCurrentState = targetState;
            mCurrentState.DoOnBefore(this);
        }
        /// <summary>
        /// 检测当前状态的计时器是否到达指定时间
        /// </summary>
        /// <param 指定时间="time"></param>
        /// <returns></returns>
        public bool CheckElapsedTime(float time)
        {
            return mElapsedTime >= time;
        }
    }
}
````

### StateBase

````csharp
/****************************************************
	文件：StateBase.cs
	作者：HuskyT
	邮箱：1005240602@qq.com
	日期：2020/3/20 15:55:28
	功能：状态基类
*****************************************************/

using System.Collections.Generic;
using UnityEngine;

namespace FiniteStateMachine
{
    /*
     * 状态基类
     * 可创建不同 AI 对象特有的状态
     * 作用：划分 AI 状态（内部处理每个状态独有的逻辑）
     * 使用：在子类中设置 状态 包含的 动作集Actions ；转换条件集（Transitions）在外部设置
     * 可选：在子类中实现 DoOnBefore 和 DoOnExit （状态 的 生命周期函数）
     */
    public abstract class StateBase
    {
        protected List<ActionBase> mActions;
        protected List<Transition> mTransitions;

        public StateBase()
        {
            mActions = new List<ActionBase>();
            mTransitions = new List<Transition>();
            AddActions();
        }

        public virtual void DoOnBefore(StateController controller) { }
        public virtual void DoOnExit(StateController controller) { }
        /// <summary>
        /// 在子类中配置 状态 State 包含的 动作集（Actions）
        /// </summary>
        protected abstract void AddActions();
        /// <summary>
        /// 在子类中配置 状态 State 包含的 转换条件集（Transitions）
        /// </summary>
        public void AddTransition(Transition transition)
        {
            for (int i = 0; i < mTransitions.Count; i++)
            {
                if (mTransitions[i] == transition)
                {
                    Debug.LogErrorFormat("{0}状态中已包含转换条件：{1}", this, transition.Decision.ToString());
                    return;
                }
            }
            mTransitions.Add(transition);
            Debug.LogFormat("在{0}状态中添加转换条件：{1}", this, transition.Decision.ToString());
        }
        public void UpdateState(StateController controller)
        {
            DoActions(controller);
            CheckTransitions(controller);
        }
        private void DoActions(StateController controller)
        {
            for (int i = 0; i < mActions.Count; i++)
            {
                mActions[i].Act(controller);
            }
        }
        private void CheckTransitions(StateController controller)
        {
            bool succeed = false;
            //每次帧循环都会检测当前状态所有的转换条件
            //若存在多个可以成功转换的 转换条件 时，选取第一个为成功转换的状态（此处可以修改选取规则）
            for (int i = 0; i < mTransitions.Count; i++)
            {
                succeed = mTransitions[i].Decision.Decide(controller);
                if (succeed)
                {
                    controller.TransitionToState(mTransitions[i].TargetState);
                    break;
                }
            }
        }
    }
}
````

### ActionBase

````csharp
/****************************************************
	文件：ActionBase.cs
	作者：HuskyT
	邮箱：1005240602@qq.com
	日期：2020/3/20 16:23:4
	功能：AI 的最小行为单元（抽象基类）
*****************************************************/

namespace FiniteStateMachine
{
    /*
     *  AI 的最小行为单元（抽象基类）
     *  作用：将 AI 行为尽可能拆分、模块化，便于拓展、复用。
     */
    public abstract class ActionBase
    {
        public abstract void Act(StateController controller);
    }
}
````

### Transition

````csharp
/****************************************************
	文件：Transition.cs
	作者：HuskyT
	邮箱：1005240602@qq.com
	日期：2020/3/20 16:44:21
	功能：转换条件
*****************************************************/

namespace FiniteStateMachine
{
    /*
     * 转换条件
     * 每个转换条件由：一个转换决策、一个转换成功后指向的状态 组成
     */
    public class Transition
    {
        private DecisionBase mDecision;
        private StateBase mTargetState;
        public DecisionBase Decision { get => mDecision; }
        public StateBase TargetState { get => mTargetState; }
        public Transition(DecisionBase decision, StateBase targetState)
        {
            mDecision = decision;
            mTargetState = targetState;
        }
    }
}
````

### Decision

````csharp
/****************************************************
	文件：DecisionBase.cs
	作者：HuskyT
	邮箱：1005240602@qq.com
	日期：2020/3/20 16:23:26
	功能：转换决策（抽象基类）
*****************************************************/

namespace FiniteStateMachine
{
    /*
     * 转换决策（抽象基类）
     * 作用：为转换条件 Transition 的核心组成部分，用于处理状态之间转换的核心逻辑
     */
    public abstract class DecisionBase
    {
        public abstract bool Decide(StateController controller);
    }
}
````

## 3.案例

实现一个追踪者（Hunter）的 AI ：

- 包含两个状态：巡逻状态 + 追逐状态
- 巡逻状态 由一个 Action 构成：巡逻行为
- 追逐状态 由两个 Action 构成：追逐行为 + 攻击行为
- 默认的初始状态为：巡逻状态
- 当目标被发现时，Hunter 由 *巡逻状态* 切换为 *追逐状态*，追逐状态 下当满足 **攻击条件** 时可进行 *攻击行为* 。
- 直到目标被消灭，Hunter 才会由 *追逐状态* 切换回 *巡逻状态*

如下图所示：

![picture1](https://huskytgame.github.io/images/in-post/ai/2020-03-20-AI--FiniteStateMachine/ScreenShot001.png)

代码：（Action 和 Decision 的实现部分略）

**HunterPatrolState：**

````csharp
using System;
using UnityEngine;

namespace FiniteStateMachine.Example1
{
    public class HunterPatrolState : StateBase
    {
        protected override void AddActions()
        {
            mActions.Add(new PatrolAction());
            Debug.LogFormat("{0}的Action添加完毕", this);
        }
    }
}
````

**HunterChaseState：**

````csharp
using System;
using UnityEngine;

namespace FiniteStateMachine.Example1
{
    public class HunterChaseState : StateBase
    {
        protected override void AddActions()
        {
            mActions.Add(new ChaseAction());
            mActions.Add(new AttackAction());
            Debug.LogFormat("{0}的Action添加完毕", this);
        }
    }
}
````

**HunterController：**

````csharp
using System;
using UnityEngine;

namespace FiniteStateMachine.Example1
{
	public class HunterController : MonoBehaviour
	{
        private StateController mAI;
		private void Start()
		{
            HunterPatrolState hunterPatrolState = new HunterPatrolState();
            HunterChaseState hunterChaseState = new HunterChaseState();
            hunterPatrolState.AddTransition(new Transition(new LookDecision(), hunterChaseState));
            hunterChaseState.AddTransition(new Transition(new InactiveStateDecision(), hunterPatrolState));
            mAI = new StateController(hunterPatrolState, null);
		}
        private void Update()
        {
            mAI.UpdateAI();
        }
    }
}
````

# 三、SerializableFiniteStateMachine

## 1.说明

可序列化的有限状态机。可以在 Unity 编辑模式下进行 AI 的快速配置，有利于原型开发。

状态机构建原理和之前的基本一致（Transition 部分略有区别，本质相同）

## 2.框架代码

### StateController

````csharp
/****************************************************
	文件：StateController.cs
	作者：HuskyT
	邮箱：1005240602@qq.com
	日期：2020/3/18 22:6:21
	功能：状态管理器（AI Brain）
*****************************************************/

using System;
using UnityEngine;

namespace SerializableFiniteStateMachine
{
    /*
     * 状态管理器
     * AI Brain，直接挂载在 NPC 上
     * 作用：AI 的状态控制器（核心类）
     */
    public class StateController : MonoBehaviour
    {
        [Tooltip("当前状态")]
        public State CurrentState;
        [Tooltip("预设的虚拟状态")]
        public State RemainState;

        /// <summary>
        /// AI 是否为激活状态
        /// </summary>
        private bool mAIActivate = false;
        /// <summary>
        /// 当前状态过去的时间（当前状态的计时器）
        /// </summary>
        private float mElapsedTime = 0.0f;

        public void SetupAI(bool aiActivate, params object[] args)
        {
            mAIActivate = aiActivate;
            //设置 AI 数据
        }
        private void Update()
        {
            if (mAIActivate == false) return;
            CurrentState.UpdateState(this);
        }
        public void TransitionToState(State targetState)
        {
            if (targetState == RemainState) return;
            CurrentState = targetState;
            OnExitState();
        }
        /// <summary>
        /// 检测当前状态的计时器是否到达指定时间
        /// </summary>
        /// <param 指定时间="time"></param>
        /// <returns></returns>
        public bool CheckElapsedTime(float time)
        {
            mElapsedTime += Time.deltaTime;
            return mElapsedTime >= time;
        }
        private void OnExitState()
        {
            mElapsedTime = 0.0f;//计时器清零
        }
    }
}
````

### State

````csharp
/****************************************************
	文件：State.cs
	作者：HuskyT
	邮箱：1005240602@qq.com
	日期：2020/3/18 22:8:17
	功能：状态
*****************************************************/

using UnityEngine;

namespace SerializableFiniteStateMachine
{
    /*
     * 状态
     * 可创建不同 AI 对象特有的状态，可序列化为 Asset 资产对象，便于在 Unity 编辑模式下配置 AI
     * 作用：划分 AI 状态（内部处理每个状态独有的逻辑）
     */
    [CreateAssetMenu(fileName = "DefaultState", menuName = "SerializableFiniteStateMachine/States")]
    public class State : ScriptableObject
    {
        [Tooltip("该状态下的行动集合")]
        public ActionBase[] Actions;
        [Tooltip("该状态下的转换条件集合")]
        public Transition[] Transitions;

        public void UpdateState(StateController controller)
        {
            DoActions(controller);
            CheckTransitions(controller);
        }

        private void DoActions(StateController controller)
        {
            for (int i = 0; i < Actions.Length; i++)
            {
                Actions[i].Act(controller);
            }
        }
        private void CheckTransitions(StateController controller)
        {
            bool succeed = false;
            //每次帧循环都会检测当前状态所有的转换条件
            //若存在多个转换条件，且同时为 TrueState 时，选取最后一个 TrueState 为成功转换的状态（此处可以修改选取规则）
            for (int i = 0; i < Transitions.Length; i++)
            {
                succeed = Transitions[i].Decision.Decide(controller);
                if (succeed)
                {
                    controller.TransitionToState(Transitions[i].TrueState);
                }
                else
                {
                    controller.TransitionToState(Transitions[i].FalseState);
                }
            }
        }
    }
}
````

### ActionBase

````csharp
/****************************************************
	文件：ActionBase.cs
	作者：HuskyT
	邮箱：1005240602@qq.com
	日期：2020/3/18 22:7:18
	功能：AI 的最小行为单元（抽象基类）
*****************************************************/

using UnityEngine;

namespace SerializableFiniteStateMachine
{
    /*
     * AI 的最小行为单元
     * 抽象基类，子类可序列化为 Asset 资产对象
     * 作用：将 AI 行为尽可能拆分、模块化，便于拓展、复用。
     */
    public abstract class ActionBase : ScriptableObject
    {
        public abstract void Act(StateController controller);
    }
}
````

### Transition

````csharp
/****************************************************
	文件：Transition.cs
	作者：HuskyT
	邮箱：1005240602@qq.com
	日期：2020/3/18 22:8:44
	功能：转换条件
*****************************************************/

using UnityEngine;

namespace SerializableFiniteStateMachine
{
    /*
     * 转换条件
     * 每个转换条件由：一个转换决策、一个转换成功后指向的状态、一个转换失败后指向的状态  组成
     * 说明：加上可序列化标签是便于在 Unity 编辑模式下进行配置
     */
    [System.Serializable]
    public class Transition
    {
        [Tooltip("转换决策")]
        public DecisionBase Decision;
        [Tooltip("转换成功后指向的状态")]
        public State TrueState;
        [Tooltip("转换失败后指向的状态")]
        public State FalseState;
    }
}
````

### DecisionBase

````csharp
/****************************************************
	文件：DecisionBase.cs
	作者：HuskyT
	邮箱：1005240602@qq.com
	日期：2020/3/18 22:8:33
	功能：转换决策（抽象基类）
*****************************************************/

using UnityEngine;

namespace SerializableFiniteStateMachine
{
    /*
     * 转换决策
     * 抽象基类，子类可序列化为 Asset 资产对象
     * 作用：为转换条件 Transition 的核心组成部分，用于处理状态之间转换的核心逻辑
     */
    public abstract class DecisionBase : ScriptableObject
    {
        public abstract bool Decide(StateController controller);
    }
}
````

## 3.案例

### （1）说明

新增一个扫描者（Scanner）的 AI ：

- 包含三个状态：巡逻状态 + 追逐状态 + 警戒状态
- 巡逻状态 由一个 Action 构成：巡逻行为
- 追逐状态 由两个 Action 构成：追逐行为 + 攻击行为
- 警戒状态 没有行为
- 默认的初始状态为：巡逻状态
- 当目标被发现时，Scanner 由 *巡逻状态* 切换为 *追逐状态*，追逐状态 下当满足 **攻击条件** 时可进行 *攻击行为* 。
- 当失去目标踪迹时，Scanner 由 *追逐状态* 切换为 *警戒状态*
- 警戒状态下，Scanner 会进行 扫描搜索 目标：如果扫描结束依旧没有发现目标则由 *警戒状态* 切换回 *巡逻状态* ；如果扫描过程中发现了目标，则由 *警戒状态* 切换回 *追逐状态*

如下图所示：

![picture2](https://huskytgame.github.io/images/in-post/ai/2020-03-20-AI--FiniteStateMachine/ScreenShot002.png)

### （2）在 Unity 编辑模式下配置 AI

**Hierarchy：**

![picture3](https://huskytgame.github.io/images/in-post/ai/2020-03-20-AI--FiniteStateMachine/ScreenShot003.png)

**各个状态：**

*ScannerPatrolState*：

![picture4](https://huskytgame.github.io/images/in-post/ai/2020-03-20-AI--FiniteStateMachine/ScreenShot004.png)

*ScannerChaseState*：

![picture5](https://huskytgame.github.io/images/in-post/ai/2020-03-20-AI--FiniteStateMachine/ScreenShot005.png)

*ScannerAlertState*：

![picture6](https://huskytgame.github.io/images/in-post/ai/2020-03-20-AI--FiniteStateMachine/ScreenShot006.png)

### （3）代码

#### PatrolAction

````csharp
/****************************************************
	文件：PatrolAction.cs
	作者：HuskyT
	邮箱：1005240602@qq.com
	日期：2020/3/18 23:44:27
	功能：巡逻 Action
*****************************************************/

using System;
using UnityEngine;

namespace SerializableFiniteStateMachine.Example1
{
    [CreateAssetMenu(fileName = "PatrolAction", menuName = "SerializableFiniteStateMachine/Actions/PatrolAction")]
    public class PatrolAction : ActionBase
    {
        public override void Act(StateController controller)
        {
            Patrol(controller);
        }
        private void Patrol(StateController controller)
        {
            //巡逻逻辑
        }
    }
}
````

#### ScanDecision

````csharp
/****************************************************
	文件：ScanDecision.cs
	作者：HuskyT
	邮箱：1005240602@qq.com
	日期：2020/3/19 0:22:2
	功能：扫描 转换决策
*****************************************************/

using System;
using UnityEngine;

namespace SerializableFiniteStateMachine.Example1
{
    [CreateAssetMenu(fileName = "ScanDecision", menuName = "SerializableFiniteStateMachine/Decisions/ScanDecision")]
    public class ScanDecision : DecisionBase
    {
        public override bool Decide(StateController controller)
        {
            return Scan(controller);
        }
        private bool Scan(StateController controller)
        {
            //扫描逻辑
            return false;
        }
    }
}
````

# 四、写在最后

如果 AI 的状态不复杂，使用 FSM 来构建 AI 是一件很高效的事情。利用 Unity 的可序列化功能能使 AI 的配置更为方便直观。

在 FSM 的基础上也可以很容易得扩展出 分层有限状态机。





Reference

[Unity官方教程](https://learn.unity.com/tutorial/5c515373edbc2a001fd5c79d?tab=overview)

[Unity录播](https://v.qq.com/x/page/c0537cidpg9.html)