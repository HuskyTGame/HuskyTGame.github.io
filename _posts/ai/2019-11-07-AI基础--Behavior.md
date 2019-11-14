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

对象在没有目标的情况下，通常会进行随机移动，随机移动的实现方式有很多种。比如说以随机的时间间隔设置随机目标，但用此方法对象通常会进行大幅度的转向，很多时候看起来会很奇怪，就好像一个人一会儿向前走，一会儿又后退，这种交替前进后退如果太过频繁看起来就显得有违常理。

此方法的便捷之处在于实现简单，结合之前的*Seek*和*Arrival*行为可快速实现。

添加*RandomTarget*脚本来生成随机目标：

````csharp
    public class RandomTarget : MonoBehaviour
    {
        [Tooltip("随机目标的范围半径")]
        public float Radius;
        private Vector3 mPos = default;
        public Vector3 Pos
        {
            get
            {
                mPos = Random.insideUnitSphere * Radius;
                return mPos;
            }
            private set { mPos = value; }
        }
        private void OnDrawGizmos()
        {
            if (mPos == default) return;
            Gizmos.color = Color.black;
            Gizmos.DrawSphere(new Vector3(mPos.x, 0.01f, mPos.z), 0.1f);
        }
    }
````

随机移动的脚本：

````csharp
    public class RandomSeekBehavior : MonoBehaviour
    {
        public RandomTarget RandomTarget;
        public float MaxVelocity;
        public float MaxForceValue;
        public float Mass;
        [Tooltip("到达目的地前的减速半径")]
        public float DecelerateRadius = 2f;
        
        private Vector3 mDesiredVelocity;
        private Vector3 mCurrentVelocity;
        private Vector3 mSteerForce;
        private Vector3 mDesiredDir;
        private float mDesiredDis;
        private float mInterval;
        private float mTimer;
        private Vector3 mTargetPos = default;

        private void Start()
        {
            mInterval = UnityEngine.Random.Range(1f, 2f);
        }
        private void FixedUpdate()
        {
            mTimer += Time.fixedDeltaTime;
            if (mTimer >= mInterval)
            {
                mTimer -= mInterval;
                mInterval = UnityEngine.Random.Range(1f, 2f);
                Vector3 pos = RandomTarget.Pos;
                mTargetPos = new Vector3(pos.x, 0f, pos.z);
            }
            Seek(mTargetPos);
        }
        /// <summary>
        /// 寻求：通过Steer引导力使对象平滑转向移动
        /// </summary>
        private void Seek(Vector3 targetPos)
        {
            mDesiredDir = targetPos - transform.position;//期望方向：指向鼠标点击方向
            mDesiredDis = mDesiredDir.magnitude;//期望距离
            if (mDesiredDis <= DecelerateRadius)//减速
            {
                mDesiredVelocity = Vector3.Normalize(targetPos - transform.position) * MaxVelocity * mDesiredDis / DecelerateRadius;//到达目标点后速度为零
            }
            else//正常移动
            {
                mDesiredVelocity = Vector3.Normalize(targetPos - transform.position) * MaxVelocity;
            }
            mSteerForce = Vector3.ClampMagnitude(mDesiredVelocity - mCurrentVelocity, MaxForceValue);//引导力：由当前运动方向引导向期望方向
            mSteerForce /= Mass;//引导力=>加速度
            mCurrentVelocity = Vector3.ClampMagnitude(mCurrentVelocity + mSteerForce, MaxVelocity);//更新当前速度：受引导力影响后的速度
            transform.position += mCurrentVelocity * Time.fixedDeltaTime;//更新位置
        }
    }
````

效果如下：

![GIF4](https://huskytgame.github.io/images/in-post/ai/2019-11-07-AI基础--Behavior/RandomSeekBehavior.gif)

对象随机移动在每一次目标改变的时候都显得很突兀，改进的方式是加快对象移动方向改变的频率，使得对象在每一帧都改变移动方向，且每一次改变的方向大小均在45度以内。如此一来，对象的随机移动就会显得十分平滑。

如何控制对象移动方向的改变呢？方法和上述一样，通过力来改变对象运动方向，称此力为*WanderForce*随机徘徊力。可以想象对象前方有一个圆圈，圆心就在对象当前移动方向上，而对象与圆上随机一点的连线就是随机徘徊力。如下图所示：

![picture5](https://huskytgame.github.io/images/in-post/ai/2019-11-07-AI基础--Behavior/ScreenShot005.png)

代码：

````csharp
    /*
     * 在对象前增加一个指示运动方向的圆
     * 限制了对象每一帧运动方向改变的大小，防止对象随机徘徊中运行方向突变超过45度。
     */
    public class WanderBehavior : MonoBehaviour
    {
        /*
         * 如果mCircleDistance>>mCircleRadius时，
         * mWanderForce对速度的影响可忽略不计，
         * 即对象不改变移动方向
         */
        [SerializeField, Tooltip("圆心到对象的距离"), Range(1f, 5f)]
        private float mCircleDistance = 2f;
        [SerializeField, Tooltip("圆半径")]
        private float mCircleRadius = 1f;
        [SerializeField, Tooltip("质量")]
        private float mMass = 1f;
        private Vector3 mCurrentVelocity;
        [SerializeField, Tooltip("最大速度"), Range(0.1f, 8f)]
        private float mMaxVelocity = 5f;
        /// <summary>
        /// 指向圆心方向
        /// </summary>
        private Vector3 mCircleCenterVector3;
        /// <summary>
        /// 位移的力（与圆半径正比例）
        /// </summary>
        private Vector3 mDisplacementForce;
        /// <summary>
        /// 位移力的方向
        /// </summary>
        private Vector3 mDisplacementDir;
        /// <summary>
        /// 顺时针方向的夹角
        /// </summary>
        private float mCurrentAngle;
        private Vector3 mWanderForce;
        //[SerializeField, Tooltip("最大随机徘徊力")]
        //private float mMaxWanderForce = 5f;
        private SpawnZone Spawner;

        private void Start()
        {
            Vector3 randomV3 = Random.onUnitSphere;
            //Vector3 randomV3 = new Vector3(0f, 0f, 1f);
            mCurrentVelocity = Vector3.ClampMagnitude(new Vector3(randomV3.x, 0f, randomV3.z), mMaxVelocity);
            mDisplacementDir = new Vector3(0f, 0f, 1f);//默认指向Z轴正方向
            mCurrentAngle = 0f;
            Spawner = GameObject.FindWithTag("SpawnZone").GetComponent<SpawnZone>();
        }
        private void Update()
        {
            if (Vector3.Distance(transform.position, Vector3.zero) > Spawner.Radius)
            {
                Spawner.Recycle(transform);
            }
        }
        private void FixedUpdate()
        {
            Wander();
        }
        private void Wander()
        {
            //指向圆心方向：
            mCircleCenterVector3 = mCurrentVelocity.normalized * mCircleDistance;
            //位移力：
            SetAngle(out mDisplacementDir, mCurrentAngle);
            mDisplacementForce = mDisplacementDir * mCircleRadius;
            //随机徘徊力：
            //随机徘徊力不受最大随机徘徊力影响，只受mCircleDistance、mCircleRadius、mDisplacementDir影响
            //mWanderForce = Vector3.ClampMagnitude(mCircleCenterVector3 + mDisplacementForce, mMaxWanderForce);
            mWanderForce = mCircleCenterVector3 + mDisplacementForce;
            mWanderForce /= mMass;//随机徘徊力=>加速度
            mCurrentVelocity = Vector3.ClampMagnitude(mCurrentVelocity + mWanderForce, mMaxVelocity);//更新当前速度：受随机徘徊力影响后的速度
            transform.position += mCurrentVelocity * Time.fixedDeltaTime;//更新位置
            //更新
            mCurrentAngle = Random.Range(0f, 2f * Mathf.PI);
        }
        private void SetAngle(out Vector3 vec, float angle)
        {
            vec = new Vector3(Mathf.Sin(angle), 0f, Mathf.Cos(angle)) * mCircleRadius;
        }
    }
````

创建多个小球来展示随机徘徊，创建代码：（超出生成半径会回收并在原点重新创建）

````csharp
    public class SpawnZone : MonoBehaviour
    {
        [SerializeField, Tooltip("生成数量")]
        private int mSpawnCount = 3;
        [SerializeField, Tooltip("生成半径")]
        private float mRadius = 5f;
        [SerializeField, Tooltip("待生成对象")]
        private GameObject mPrefab = default;
        /// <summary>
        /// 缓存生成出来的对象
        /// </summary>
        private List<Transform> ObjectList;
        /// <summary>
        /// 生成半径
        /// </summary>
        public float Radius => mRadius;

        private void Start()
        {
            ObjectList = new List<Transform>();
            for (int i = 1; i <= mSpawnCount; i++)
            {
                Spawn();
            }
        }
        public void Spawn()
        {
            GameObject go = Instantiate(mPrefab, Vector3.zero, Quaternion.identity, transform);
            ObjectList.Add(go.transform);
        }
        public void Recycle(Transform obj)
        {
            ObjectList.Remove(obj);
            Spawn();
            Destroy(obj.gameObject);
        }
    }
````

效果如下：

![GIF5](https://huskytgame.github.io/images/in-post/ai/2019-11-07-AI基础--Behavior/WanderBehavior.gif)

## 六、Pursuit追踪

