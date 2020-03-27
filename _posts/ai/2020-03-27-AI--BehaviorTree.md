---
layout: article
title:  "AI--BehaviorTree"
categories: ai
image:
    teaser: /in-post/ai/2020-03-27-AI--BehaviorTree/DefaultImg.jpg
---

# 目录

[TOC]

![GIF1](https://huskytgame.github.io/images/in-post/ai/2020-03-27-AI--BehaviorTree/BehaviorTreeExample.gif)

# 一、写在前面

本文实现了一个简易的行为树框架。工程地址：[GitHub](https://github.com/HuskyTGame/CustomBehaviorTree) and [网盘](https://pan.baidu.com/s/124-kkyyPO-2uBgs6UvzBsQ)（提取码：yu6b）

包含以下内容：

- **组合节点（CompositeNode）**
  - *并行节点（ParallelNode*
  
  - *选择节点（SelectorNode）*
  
  - *序列节点（SequenceNode）*
  
- **修饰节点（DecoratorNode）**
  - *循环节点（RepeatNode）*
  
  - *TOADD*
  
- **行为叶节点（ActonNode）**

- **前提条件（Precondition）**

- **黑板（Blackboard）**

行为树 较 FSM 有许多优势：

- 1.更容易实现复杂 AI 逻辑。（若使用 FSM 会因 State 数量的增加导致 Transition 数量骤增，很难管理维护）
- 2.行为树内部节点分为逻辑节点和行为节点：逻辑节点（组合节点 + 修饰节点）用于控制子节点运行逻辑；行为节点才是实现具体的 AI 行为。可以将行为节点模块化便于复用行为逻辑。（这一点 FSM 也可以办到）
- 3.在理解行为树原理后，通过行为树可以快速直观的理解 AI 逻辑。（FSM 会因状态过多很难轻易理解各个 State 之间的转换逻辑）

行为树也有他的不足之处：

- 1.实现较为复杂，不过很多平台都有前人实现好的行为树框架。
- 2.行为树需要好的设计，能让行为粒度在复用性和复杂性之间达到平衡。
- 3.行为树层深过大的时候，需要不小的性能开销。
- 4.行为树高度依赖于 AI 设计与编码，属于 “反应型 AI” ，在处理某些类型 AI 问题上不如 “协商型 AI”（GOAP、HTN） 方便。（不过 "协商型 AI" 也有不足：实现更复杂；AI 设计更考究；性能消耗与内部行为搜索算法和行动域大小高度相关）

# 二、BehaviorTree

## 1.行为树框架结构

如下图所示：

![picture0](https://huskytgame.github.io/images/in-post/ai/2020-03-27-AI--BehaviorTree/ScreenShot000.png)

包含以下内容：

（注意：每个节点都包含 3 种运行状态：Failed、Running、Finished）

- **组合节点（CompositeNode）：**

  用于控制多个子节点运行（遍历）逻辑。

  - ***并行节点（ParallelNode）***：子节点可同时运行。

    只有当子节点返回指定个 Finished 时，并行节点状态才为 Finished。

    若过多子节点返回 Failed 导致不可能有足够多子节点返回 Finished 时，并行节点返回 Failed。

    （注意：并行节点在 Running 状态下，状态为 Running 和 Failed 的子节点每帧都会更新状态，状态为 Finished 的子节点不再更新状态）

  - ***选择节点（SelectorNode）***：子节点由前到后按顺序依次运行。

    当出现第一个返回 Finished 的子节点时，选择节点才返回 Finished。

    若所有子节点均返回 Failed 时，选择节点返回 Failed。

  - ***序列节点（SequenceNode）***：子节点由前到后按顺序依次运行。

    当子节点依次返回 Finished 时，序列节点才返回 Finished。

    若子节点返回 Failed，则序列节点返回 Failed。

- **修饰节点（DecoratorNode）：**

  用于 “修饰” 单个子节点的运行逻辑。

  - ***循环节点（RepeatNode）***：子节点循环执行指定次数。

    当子节点返回 Finished 时，循环节点循环执行，知道达到指定的循环次数。

    当子节点返回 Failed 时，循环节点停止循环并返回 Failed。

  - ***TOADD***：

    还有其他的修饰节点，本文并未实现，不过很容易扩展出来。

    例如：逆变节点（InverterNode）、成功节点（SucceederNode）等。

- **行为叶节点（ActonNode）：**

  位于行为树叶子节点位置的节点，真正用于执行 AI 行为逻辑的节点。

  内部逻辑：

  - 内部条件检测是否通过：``InternalCondition()``
  - 执行``OnEnter()``（第一次执行当前节点时）
  - 执行``OnExcute()``
  - 执行``OnExit()``（退出当前节点时执行）

- **前提条件（Precondition）：**

  为节点添加额外的执行条件（外在前提条件）

- **黑板（Blackboard）：**

  用于各个节点之间数据通信。
  
  内部设置有数据的有效时间：若数据过期，则无法使用。

## 2.行为树执行流程

### （1）整体执行流程

![picture1](https://huskytgame.github.io/images/in-post/ai/2020-03-27-AI--BehaviorTree/ScreenShot001.png)

### （2）行为叶节点（ActionNode）

![picture2](https://huskytgame.github.io/images/in-post/ai/2020-03-27-AI--BehaviorTree/ScreenShot002.png)

### （3）并行节点（ParallelNode）——组合节点

![picture3](https://huskytgame.github.io/images/in-post/ai/2020-03-27-AI--BehaviorTree/ScreenShot003.png)

### （4）选择节点（SelectorNode）——组合节点

![picture4](https://huskytgame.github.io/images/in-post/ai/2020-03-27-AI--BehaviorTree/ScreenShot004.png)

### （5）序列节点（SequenceNode）——组合节点

![picture5](https://huskytgame.github.io/images/in-post/ai/2020-03-27-AI--BehaviorTree/ScreenShot005.png)

### （6）循环节点（RepeatNode）——修饰节点

![picture6](https://huskytgame.github.io/images/in-post/ai/2020-03-27-AI--BehaviorTree/ScreenShot006.png)

## 3.代码

### （1）BTStarter 行为树启动器

````csharp
/****************************************************
	文件：BTStarter.cs
	作者：HuskyT
	邮箱：1005240602@qq.com
	日期：2020/3/26 19:29:34
	功能：行为树启动器（在帧循环中开启行为树）
*****************************************************/

using HTUtility;

namespace AI.BehaviorTree
{
    public class BTStarter : HTSingleton<BTStarter>
    {
        public void UpdateBT(NodeBase rootNode, IAgent agent, Blackboard bb)
        {
            BTNodeStatus status = rootNode.Update(agent, bb);
            //HTLogger.Debug(string.Format("根节点为：{0}的行为树运行状态为：{1}", rootNode, status));
        }
    }
}
````

### （2）NodeBase 节点基类

````csharp
/****************************************************
	文件：NodeBase.cs
	作者：HuskyT
	邮箱：1005240602@qq.com
	日期：2020/3/25 22:47:16
	功能：行为树节点（基类）
*****************************************************/

using System.Collections.Generic;

namespace AI.BehaviorTree
{
    public abstract class NodeBase
    {
        /// <summary>
        /// 父节点
        /// </summary>
        protected NodeBase mParent;
        /// <summary>
        /// 子节点列表
        /// </summary>
        protected List<NodeBase> mChildren = new List<NodeBase>();
        /// <summary>
        /// 前提条件（外在前提）
        /// </summary>
        protected IPrecondition mPrecondition;

        public NodeBase()
        {
        }

        /// <summary>
        /// 添加子节点
        /// </summary>
        /// <param 子节点="node"></param>
        public NodeBase AddChild(params NodeBase[] nodeArray)
        {
            for (int i = 0; i < nodeArray.Length; i++)
            {
                nodeArray[i].mParent = this;
                mChildren.Add(nodeArray[i]);
            }
            return this;
        }
        /// <summary>
        /// 设置节点的前提条件（外在前提）
        /// </summary>
        /// <param 前提条件="precondition"></param>
        public NodeBase SetPrecondition(IPrecondition precondition)
        {
            mPrecondition = precondition;
            return this;
        }
        public BTNodeStatus Update(IAgent agent, Blackboard bb)
        {
            //没有通过（外在）前提条件
            if (mPrecondition != null && mPrecondition.IsTrue(agent) == false)
                return BTNodeStatus.Failed;
            return OnUpdate(agent, bb);
        }
        public void Reset(IAgent agent, Blackboard bb)
        {
            OnReset(agent, bb);
        }
        protected virtual BTNodeStatus OnUpdate(IAgent agent, Blackboard bb)
        {
            return BTNodeStatus.Finished;
        }
        protected virtual void OnReset(IAgent agent, Blackboard bb) { }
    }
}
````

### （3）ActionNode 行为叶节点

````csharp
/****************************************************
	文件：ActionNode.cs
	作者：HuskyT
	邮箱：1005240602@qq.com
	日期：2020/3/25 23:8:27
	功能：行为节点（行为树叶节点）
*****************************************************/

namespace AI.BehaviorTree
{
    public class ActionNode : NodeBase
    {
        /// <summary>
        /// 行为状态，在行为节点内部的状态
        /// </summary>
        public enum ActionStateEnum
        {
            /// <summary>
            /// 第一次执行当前行为
            /// </summary>
            FirstIn,
            /// <summary>
            /// 非第一次执行当前行为
            /// </summary>
            NotFirstIn,
        }

        /// <summary>
        /// 当前行为节点内部的状态，默认为 Ready 状态
        /// </summary>
        private ActionStateEnum mCurrentState = ActionStateEnum.FirstIn;

        protected override BTNodeStatus OnUpdate(IAgent agent, Blackboard bb)
        {
            //内部条件检测未通过
            if (InternalCondition(agent, bb) == false) return BTNodeStatus.Failed;

            //第一次执行此行为
            if(mCurrentState == ActionStateEnum.FirstIn)
            {
                OnEnter(agent, bb);
                mCurrentState = ActionStateEnum.NotFirstIn;
            }
            BTNodeStatus status = BTNodeStatus.Finished;
            //非第一次执行此行为
            if (mCurrentState==ActionStateEnum.NotFirstIn)
            {
                status = OnExcute(agent, bb);
            }
            //若当前行为为持续性行为
            if (status == BTNodeStatus.Running)
            {
                return status;//返回 Running
            }
            //当前行为已结束（非持续性行为）
            else
            {
                //善后工作（执行退出 + 标志位重置）
                OnReset(agent, bb);
                return status;//返回 Finished
            }
        }
        protected override void OnReset(IAgent agent, Blackboard bb)
        {
            if (mCurrentState == ActionStateEnum.NotFirstIn)
            {
                OnExit(agent, bb);//执行退出
                mCurrentState = ActionStateEnum.FirstIn;//标志位重置
            }
        }

        /// <summary>
        /// 内部条件
        /// </summary>
        /// <param AI 实体="agent"></param>
        /// <param 黑板数据="bb"></param>
        /// <returns></returns>
        protected virtual bool InternalCondition(IAgent agent, Blackboard bb)
        {
            return true;
        }

        #region 行为节点  生命周期函数
        protected virtual void OnEnter(IAgent agent, Blackboard bb) { }
        protected virtual BTNodeStatus OnExcute(IAgent agent, Blackboard bb)
        {
            return BTNodeStatus.Finished;
        }
        protected virtual void OnExit(IAgent agent, Blackboard bb) { }
        #endregion
    }
}
````

### （4）ParallelNode 并行节点

````csharp
/****************************************************
	文件：ParallelNode.cs
	作者：HuskyT
	邮箱：1005240602@qq.com
	日期：2020/3/25 23:45:37
	功能：并行节点（组合节点）
*****************************************************/

using System.Collections.Generic;
using UnityEngine;

namespace AI.BehaviorTree
{
    public class ParallelNode : NodeBase
    {
        /// <summary>
        /// 子节点需要完成多少个，并行节点才返回 Finished
        /// </summary>
        private int mNeedFinishedNumInChildren;
        /// <summary>
        /// 子节点运行状态列表
        /// </summary>
        private List<BTNodeStatus> mChildrenStatusList;
        /// <summary>
        /// 子节点中执行完成的个数
        /// </summary>
        protected int mFinishedNum;
        /// <summary>
        /// 子节点中执行失败的个数
        /// </summary>
        protected int mFailedNum;

        /// <summary>
        /// 并行节点（参数：子节点需要完成多少个，并行节点才返回 Finished）
        /// </summary>
        /// <param 子节点需要完成多少个，并行节点才返回Finished="needFinishedNumInChildren"></param>
        public ParallelNode(int needFinishedNumInChildren)
        {
            mNeedFinishedNumInChildren = needFinishedNumInChildren;
            mChildrenStatusList = new List<BTNodeStatus>();
            mFinishedNum = 0;
            mFailedNum = 0;
        }

        protected override BTNodeStatus OnUpdate(IAgent agent, Blackboard bb)
        {
            if (mChildren.Count == 0) return BTNodeStatus.Finished;
            mNeedFinishedNumInChildren = Mathf.Clamp(mNeedFinishedNumInChildren, 1, mChildren.Count);
            //第一次设置（初始化）子节点运行状态列表
            if (mChildrenStatusList.Count != mChildren.Count)
            {
                for (int i = 0; i < mChildren.Count; i++)
                {
                    mChildrenStatusList.Add(BTNodeStatus.Running);
                }
            }
            mFinishedNum = 0;//重置  子节点中执行完成的个数
            mFailedNum = 0;//重置  子节点中执行失败的个数
            BTNodeStatus status;
            for (int i = 0; i < mChildren.Count; i++)
            {
                status = mChildrenStatusList[i];
                //1--子节点行为为持续性行为
                if (status == BTNodeStatus.Running)
                {
                    status = mChildren[i].Update(agent, bb);//更新子节点行为状态
                }
                //2--子节点行为已结束（非持续性行为/恰好结束的持续性行为）
                if (status == BTNodeStatus.Finished)
                {
                    //更新  子节点行为状态  到  子节点运行状态列表
                    //此步骤的目的：
                    //并行节点在 Running 状态下的时候
                    //状态为 Running 和 Failed 的子节点每帧都会更新状态
                    //状态为 Finished 的子节点不再更新状态
                    mChildrenStatusList[i] = BTNodeStatus.Finished;
                    mFinishedNum += 1;
                    //满足子节点需要完成的次数
                    if (mFinishedNum == mNeedFinishedNumInChildren)
                    {
                        return BTNodeStatus.Finished;//返回 Finished
                    }
                }
                //3--子节点行为失败（第一次更新该子节点时可能出现）
                if (status == BTNodeStatus.Failed)
                {
                    //此处不更新  子节点运行状态列表
                    //原因：运行状态为 Failed 的子节点在下一帧中还需要重新更新运行状态
                    mFailedNum += 1;
                    //若失败次数足够多
                    if (mFailedNum > mChildren.Count - mNeedFinishedNumInChildren)
                    {
                        return BTNodeStatus.Failed;//返回 Failed
                    }
                }
            }
            return BTNodeStatus.Running;//返回 Running
        }
        protected override void OnReset(IAgent agent, Blackboard bb)
        {
            for (int i = 0; i < mChildren.Count; i++)
            {
                mChildren[i].Reset(agent, bb);
            }
            mChildrenStatusList.Clear();
        }
    }
}
````

### （5）DecoratorNode 修饰节点

````csharp
/****************************************************
	文件：DecoratorNode.cs
	作者：HuskyT
	邮箱：1005240602@qq.com
	日期：2020/3/26 1:37:9
	功能：修饰节点（基类）（修饰节点只有一个子节点）
*****************************************************/

namespace AI.BehaviorTree
{
    public class DecoratorNode : NodeBase
    {
        /// <summary>
        /// 当前修饰节点的子节点（修饰节点只有一个子节点）
        /// </summary>
        public NodeBase Child => mChildren[0];
        public DecoratorNode(NodeBase child)
        {
            AddChild(child);
        }
    }
}
````

### （6）IPrecondition 前提条件接口

````csharp
/****************************************************
	文件：IPrecondition.cs
	作者：HuskyT
	邮箱：1005240602@qq.com
	日期：2020/3/25 22:31:58
	功能：前提条件接口
*****************************************************/

namespace AI.BehaviorTree
{
    public interface IPrecondition
    {
        bool IsTrue(IAgent agent);
    }
}
````

### （7）Blackboard 黑板

````csharp
/****************************************************
	文件：BlackboardItem.cs
	作者：HuskyT
	邮箱：1005240602@qq.com
	日期：2020/3/25 18:50:33
	功能：黑板内的单个数据项
*****************************************************/

using UnityEngine;

namespace AI.BehaviorTree
{
    public class BlackboardItem
    {
        /// <summary>
        /// 黑板数据有效的终止时间（部分黑板数据可能有有效期，超过有效期将无效）
        /// </summary>
        private float mExpiredValidTime;
        private object mValue;
        public void SetValue(object value, float validTime = -1.0f)
        {
            if (validTime >= 0)//数据存在有效时间
                //更新 黑板数据有效的终止时间
                mExpiredValidTime = Time.time + validTime;
            else
                mExpiredValidTime = -1.0f;
            mValue = value;
        }
        /// <summary>
        /// 获取黑板数据
        /// </summary>
        /// <typeparam 数据类型="T"></typeparam>
        /// <param 默认数据="defaultValue"></param>
        /// <returns></returns>
        public T GetValue<T>(T defaultValue)
        {
            if (ValidValue())
                return (T)mValue;
            return defaultValue;
        }
        public bool ValidValue()
        {
            return mExpiredValidTime < 0 || mExpiredValidTime >= Time.time;
        }
    }
}
````



````csharp
/****************************************************
	文件：Blackboard.cs
	作者：HuskyT
	邮箱：1005240602@qq.com
	日期：2020/3/25 18:50:42
	功能：黑板（包含行为树各个节点之间需要传递的信息）
*****************************************************/

using System.Collections.Generic;

namespace AI.BehaviorTree
{
    public class Blackboard
    {
        /// <summary>
        /// 缓存所有黑板数据项
        /// </summary>
        private Dictionary<int, BlackboardItem> mItems;

        public Blackboard()
        {
            mItems = new Dictionary<int, BlackboardItem>();
        }

        /// <summary>
        /// 设置黑板数据，若设置了有效时间，则有效时间到了之后数据将失效
        /// </summary>
        /// <param name="key"></param>
        /// <param name="value"></param>
        /// <param 有效时间="validTime"></param>
        public void SetValue(BlackboardKey key, object value, float validTime = -1.0f)
        {
            SetValue((int)key, value, validTime);
        }
        /// <summary>
        /// 获取黑板数据
        /// </summary>
        public T GetValue<T>(BlackboardKey key, T defaultValue)
        {
            return GetValue<T>((int)key, defaultValue);
        }
        /// <summary>
        /// 删除黑板数据
        /// </summary>
        public bool DeleteValue(BlackboardKey key)
        {
            return DeleteValue((int)key);
        }
        /// <summary>
        /// 黑板中是否包含指定数据
        /// </summary>
        public bool ContainsValue(BlackboardKey key)
        {
            return ContainsValue((int)key);
        }
        /// <summary>
        /// 尝试从黑板中获取指定数据
        /// </summary>
        public bool TryGetValue<T>(BlackboardKey key, out T value)
        {
            return TryGetValue<T>((int)key, out value);
        }

        /// <summary>
        /// 设置黑板数据，若设置了有效时间，则有效时间到了之后数据将失效
        /// </summary>
        /// <param name="key"></param>
        /// <param name="value"></param>
        /// <param 有效时间="validTime"></param>
        public void SetValue(int key, object value, float validTime = -1.0f)
        {
            BlackboardItem item;
            if (mItems.TryGetValue(key, out item) == false)
            {
                item = new BlackboardItem();
                mItems.Add(key, item);
            }
            item.SetValue(value, validTime);
        }
        /// <summary>
        /// 获取黑板数据
        /// </summary>
        public T GetValue<T>(int key, T defaultValue)
        {
            BlackboardItem item;
            if (mItems.TryGetValue(key, out item) == false)
            {
                return defaultValue;
            }
            return item.GetValue<T>(defaultValue);
        }
        /// <summary>
        /// 删除黑板数据
        /// </summary>
        public bool DeleteValue(int key)
        {
            return mItems.Remove(key);
        }
        /// <summary>
        /// 黑板中是否包含指定数据
        /// </summary>
        public bool ContainsValue(int key)
        {
            if (mItems.ContainsKey(key) == false)
                return false;
            return mItems[key].ValidValue();
        }
        /// <summary>
        /// 尝试从黑板中获取指定数据
        /// </summary>
        public bool TryGetValue<T>(int key, out T value)
        {
            BlackboardItem item;
            if (mItems.TryGetValue(key, out item) == false || item.ValidValue() == false)
            {
                value = default(T);
                return false;
            }
            value = item.GetValue<T>(default(T));
            return true;
        }
    }
}
````

## 4.案例

### （1）运行效果

说明：

AI 有自己的需求：Food、Water、Energy、Mood、Money

不同的行为会影响不同需求的数值。

红色区域可以增加 Food；蓝色区域可以增加 Water；白色区域可以增加 Energy；绿色区域可以增加 Mood；黄色区域可以增加 Money。

![GIF2](https://huskytgame.github.io/images/in-post/ai/2020-03-27-AI--BehaviorTree/BehaviorTreeExample2.gif)

### （2）行为树图

![picture7](https://huskytgame.github.io/images/in-post/ai/2020-03-27-AI--BehaviorTree/ScreenShot007.png)

### （3）构建行为树

![picture8](https://huskytgame.github.io/images/in-post/ai/2020-03-27-AI--BehaviorTree/ScreenShot008.png)

````csharp
private void Update()
{
    if (mIsActive == false) return;
    UpdateMovement();
    UpdateTurnAround();
    BTStarter.Instance.UpdateBT(mRootNode, this, mBlackboard);
}
private void BuildBehaviorTree()
{
    mBlackboard = new Blackboard();
    mRootNode = new RepeatNode(
        new ParallelNode(1)
        .AddChild(new SelectorNode()
                  .AddChild(new SequenceNode()
                            .AddChild(new FeelBoring(), new WalkTo(), new HaveFun())
                            , new SequenceNode()
                            .AddChild(new FeelHungry(), new WalkTo(), new EatFood())
                            , new SequenceNode()
                            .AddChild(new FeelThirsty(), new WalkTo(), new DrinkWater())
                            , new SequenceNode()
                            .AddChild(new FeelTired(), new WalkTo(), new HaveRest())
                            , new SequenceNode()
                            .AddChild(new FeelPoor(), new WalkTo(), new Work())
                           )
                  , new Alive())
        , 0);
    HTLogger.Info("AI 行为树构建完成！");
}
````











Reference

[AI 行为树的工作原理](https://indienova.com/indie-game-development/ai-behavior-trees-how-they-work/)

[用800行代码做个行为树（Behavior Tree）的库](http://www.aisharing.com/archives/517)

[黑板和共享数据](http://www.aisharing.com/archives/801)

[游戏AI - 行为树Part2：框架](https://zhuanlan.zhihu.com/p/19891875)

