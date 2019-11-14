---
layout: article
title:  "AI基础--Behavior"
categories: ai
image:
    teaser: /in-post/ai/2019-11-07-AI基础--Behavior/DefaultImg.jpg
---

# 目录

[TOC]

## 一、常规的移动控制

首先来实现一个最简单的通过点击鼠标控制对象移动的方式：

创建地面Ground，并新建Layer层--9:Ground；创建红色Sphere作为Player，并为其添加显示移动路径的*Trail Renderer*组件，为移动路线添加绿色材质。

- ***Step1***：获取鼠标点击位置。

  创建``GameManager``类来处理鼠标点击位置的获取。对外提供``Pos``属性--鼠标点击的位置信息。

  ````csharp
      public class GameManager : MonoBehaviour
      {
          private Ray mRay => Camera.main.ScreenPointToRay(Input.mousePosition);
          private Vector3 mPos;
  
          public Vector3 Pos => mPos;
  
          private void Awake()
          {
              mPos = Vector3.zero;
          }
          private void Update()
          {
              if (Input.GetMouseButton(0))
              {
                  RaycastHit raycastHit;
                  if (Physics.Raycast(mRay, out raycastHit, float.MaxValue, 1 << 9))
                  {
                      mPos = raycastHit.point;
                  }
              }
          }
      }
  ````

- ***Step2***：创建``PlayerCtrl``脚本，处理对象移动控制逻辑。

  ````csharp
  public class PlayerCtrl : MonoBehaviour
      {
          public GameManager Manager;
          public float MaxVelocity;
  
          private void Start()
          {
              mCurrentVelocity = new Vector3(1 * MaxVelocity, 0, 0);//初始速度
          }
          private void FixedUpdate()
          {
              //不使用Steer引导力，对象直接转向移动
              NormalSeek();
          }
          private void NormalSeek()
          {
              mCurrentVelocity = Vector3.Normalize(Manager.Pos - transform.position) * MaxVelocity;
              transform.position += mCurrentVelocity * Time.fixedDeltaTime;
          }
      }
  ````

结果展示：

![GIF1](https://huskytgame.github.io/images/in-post/ai/2019-11-07-AI基础--Behavior/NormalSeekBehavior.gif)

可见移动转向不够平滑，太过突兀。

## 二、Seek寻求

可以使用力来控制对象移动：对象移动过程中，如果遇到方向改变，有两种做法。一种是直接在新的运动方向上赋值速度；另一种是，给对象赋值一个指向目标方向的Steer引导力。具体可见下图：

![picture0](https://huskytgame.github.io/images/in-post/ai/2019-11-07-AI基础--Behavior/ScreenShot000.png)

使用Steer引导力来改变对象移动方向的方式可以使得对象转向变得平滑。

代码如下：

````csharp
    public class PlayerCtrl : MonoBehaviour
    {
        public GameManager Manager;
        public float MaxVelocity;
        public float MaxForceValue;
        public float Mass;

        private Vector3 mDesiredVelocity;
        private Vector3 mCurrentVelocity;
        private Vector3 mSteerForce;
        private Vector3 mDesiredDir;
        private float mDesiredDis;

        private void Start()
        {
            mCurrentVelocity = new Vector3(1 * MaxVelocity, 0, 0);//初始速度
        }
        private void FixedUpdate()
        {
            //不使用Steer引导力，对象直接转向移动
            //NormalSeek();
            //寻求：通过Steer引导力使对象平滑转向移动
            Seek();
        }

        private void NormalSeek(){......}

        /// <summary>
        /// 寻求：通过Steer引导力使对象平滑转向移动
        /// </summary>
        private void Seek()
        {
            mDesiredDir = Manager.Pos - transform.position;//期望方向：指向鼠标点击方向
            mDesiredDis = mDesiredDir.magnitude;//期望距离
            mDesiredVelocity = Vector3.Normalize(Manager.Pos - transform.position) * MaxVelocity;
            mSteerForce = Vector3.ClampMagnitude(mDesiredVelocity - mCurrentVelocity, MaxForceValue);//引导力：由当前运动方向引导向期望方向
            mSteerForce /= Mass;//引导力=>加速度
            mCurrentVelocity = Vector3.ClampMagnitude(mCurrentVelocity + mSteerForce, MaxVelocity);//更新当前速度：受引导力影响后的速度
            transform.position += mCurrentVelocity * Time.fixedDeltaTime;//更新位置
        }
    }
````

如此一来会有新的问题产生：对象当到达目标位置后由于没有处理到达时的逻辑，``mSteerForce = Vector3.ClampMagnitude(mDesiredVelocity - mCurrentVelocity, MaxForceValue);``会导致对象到达目标点后受到反方向Steer引导力，从而出现震荡运动的问题。

## 三、Arrival达到

未解决Seek寻求行为中的问题，需要处理到达目标点的逻辑。如下图所示：

![picture1](https://huskytgame.github.io/images/in-post/ai/2019-11-07-AI基础--Behavior/ScreenShot001.png)

对象到达目标点前应该有一段平滑减速的区域，越接近目标点期望速度应该越小，直至为0。

![picture2](https://huskytgame.github.io/images/in-post/ai/2019-11-07-AI基础--Behavior/ScreenShot002.png)

````csharp
    public class PlayerCtrl : MonoBehaviour
    {
		......
        [Tooltip("到达目的地前的减速半径")]
        public float DecelerateRadius = 2f;
		......
        /// <summary>
        /// 寻求：通过Steer引导力使对象平滑转向移动
        /// </summary>
        private void Seek()
        {
            mDesiredDir = Manager.Pos - transform.position;//期望方向：指向鼠标点击方向
            mDesiredDis = mDesiredDir.magnitude;//期望距离
            if (mDesiredDis <= DecelerateRadius)//减速
            {
                mDesiredVelocity = Vector3.Normalize(Manager.Pos - transform.position) * MaxVelocity * mDesiredDis / DecelerateRadius;//到达目标点后速度为零
            }
            else//正常移动
            {
                mDesiredVelocity = Vector3.Normalize(Manager.Pos - transform.position) * MaxVelocity;
            }
            mSteerForce = Vector3.ClampMagnitude(mDesiredVelocity - mCurrentVelocity, MaxForceValue);//引导力：由当前运动方向引导向期望方向
            mSteerForce /= Mass;//引导力=>加速度
            mCurrentVelocity = Vector3.ClampMagnitude(mCurrentVelocity + mSteerForce, MaxVelocity);//更新当前速度：受引导力影响后的速度
            transform.position += mCurrentVelocity * Time.fixedDeltaTime;//更新位置
        }
    }
````

效果展示：

![GIF2](https://huskytgame.github.io/images/in-post/ai/2019-11-07-AI基础--Behavior/SeekBehavior.gif)

## 四、Flee逃离

Flee逃离行为：对象在接近目标（障碍物）时，会有一个引导力试图将对象推离障碍物。Flee的期望速度方向正好与Seek的期望速度方向相反。

![picture3](https://huskytgame.github.io/images/in-post/ai/2019-11-07-AI基础--Behavior/ScreenShot003.png)

在场景中创建一个灰色Cube命名为Obstacle，并创建同名脚本。

````csharp
    public class Obstacle : MonoBehaviour
    {
        [SerializeField]
        private float mRadius = 2f;
        public float Radius => mRadius;
        private void OnDrawGizmos()
        {
            Gizmos.color = new Color(0f, 1f, 1f, 0.2f);//蓝绿色
            //将Gizmos按照当前物体的Transform进行坐标转换，转换到世界空间
            Gizmos.matrix = transform.localToWorldMatrix;
            Gizmos.DrawSphere(Vector3.zero, mRadius);
        }
    }
````

``OnDrawGizmos()``中创建了只在Scene视图中显示的球形障碍半径。

![picture4](https://huskytgame.github.io/images/in-post/ai/2019-11-07-AI基础--Behavior/ScreenShot004.png)

*PlayerCtrl*脚本：

````csharp
    public class PlayerCtrl : MonoBehaviour
    {
		......
        [SerializeField]
        private Obstacle mObstacle = default;
        [Tooltip("最大逃离的力的值")]
        public float MaxFleeForceValue;
		......
        private void FixedUpdate()
        {
		......
            Flee();
        }
		......
        /// <summary>
        /// 逃离
        /// </summary>
        private void Flee()
        {
            if (Vector3.Distance(transform.position, mObstacle.transform.position) < mObstacle.Radius)
            {
                mDesiredVelocity = -Vector3.Normalize(mObstacle.transform.position - transform.position) * MaxVelocity;//逃离的力的方向与移动到目标方向相反
                mSteerForce = Vector3.ClampMagnitude(mDesiredVelocity - mCurrentVelocity, MaxFleeForceValue);//引导力：由当前运动方向引导向期望方向
                mSteerForce /= Mass;//引导力=>加速度
                mCurrentVelocity = Vector3.ClampMagnitude(mCurrentVelocity + mSteerForce, MaxVelocity);//更新当前速度：受引导力影响后的速度
                transform.position += mCurrentVelocity * Time.fixedDeltaTime;//更新位置
            }
        }
    }
````

效果展示：

![GIF3](https://huskytgame.github.io/images/in-post/ai/2019-11-07-AI基础--Behavior/FleeBehavior.gif)

## 五、Wander随机徘徊

![GIF4](https://huskytgame.github.io/images/in-post/ai/2019-11-07-AI基础--Behavior/WanderBehavior.gif)