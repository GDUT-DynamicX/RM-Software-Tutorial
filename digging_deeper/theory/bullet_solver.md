# 简介
在实际中，要打击的目标往往是会移动的，并且子弹在空气中也会受到阻力以及枪管的发射会存在延时。我们需要做一个枪管角度解算器，用于预测枪管的发射角度，从而实现更加精准的打击。

目的:
- 考虑空气阻力(与速度成正比)
- 考虑目标速度
- 考虑发射延时

难点：
- 无解析
- 目标解算频率大于1Khz

两种算法：
- 迭代法
  - 优点：精度较高(迭代次数多时)
  - 缺点：速度慢
- 速度叠加法
  - 优点：速度快(20us内)
  - 缺点：存在固有误差
<br/>
- 算法框图：

![BE2uFg.png](https://s1.ax1x.com/2020/10/24/BE2uFg.png)

# 模型推导
空气阻力与速度成正比，目标点以恒定速度运动
2D子弹运动模型：在xz平面内，假设以子弹子弹发射点为坐标原点，则子弹的实际位置相当于在xoz平面内的一个矢量。
![BEgFbT.png](https://s1.ax1x.com/2020/10/23/BEgFbT.png)
用x表示子弹位置矢量在x轴方向上的分量，用z表示子弹位置矢量在z轴方向上的分量；$v_0$表示子弹发射初速度，$v_x、v_z$分别表示子弹速度在x轴、z轴的分量，$v_{x_0}、v_{z_0}$分别表示$v_0$在x轴、z轴的分量，k为空气阻力系数，g为重力加速度，m表示子弹重量，$f_x$表示子弹所受空气阻力在X方向上的分量，根据物理知识知：
$$f_x=-kv_x=m\frac{dv_x}{dt}\tag{1}$$
将(1)式整理得到：
$$-\frac kmdt=\frac{dv_x}{v_x}$$
以子弹发射的时刻为0时刻,发射初速度在 x 方向上的分量为$v_x$，对上式积分：
$$\int_0^t-\frac kmdt=\int_{v_{x_0}}^{v_x}\frac{dv_x}{v_x}\tag{2}$$
由(2)式求解得到：
$$v_x=\frac{dx}{dt}=v_{x_0}e^{-\frac kmt}$$
对上式两边积分可得：
$$x=\frac mk v_{x_0}(1-e^{-\frac kmt})\tag{3}$$
同时，可求子弹在z轴方向上受到的阻力为：
$$f_z=-kv_z-mg=m\frac{dv_z}{dt}$$
整理得到微分方程：
$$\frac{dv_z}{dt}+\frac kmv_z+g=0\tag{4}$$
解得：
$$z=\frac km\left(v_{z_0}+\frac{mg}k\right)\left(1-e^{-\frac kmt}\right)-\frac{mg}kt\tag{5}$$
目标点运动模型：设目标点的起始位置为$\left(x_{t_0} , z_{t_0}\right)$，目标点在 x 轴方向上的速度为$v_{t_x}$，在 z 轴方向上的速度为$v_{t_z}$，目标点实际位置在 x 轴方向的分量为$x_t$，在 z 轴方向的分量为$z_t$。跟据匀速直线运动的公式可得：
$$x_t=x_{t_0}+v_{t_x}t\\z_t=z_{t_0}+v_{t_z}t$$   
基于上述2D模型的推导，用y表示子弹位置矢量在y轴方向上的分量，$v_{y_0}$表示$v_0$在y轴的分量；目标点实际位置在 y 轴方向的分量为$y_t$，起始位置的 y 轴坐标为$y_{t_0}$，在 y 轴方向上的速度为$v_{t_y}$，可以得到3D模型下子弹和目标点在y方向的运动模型：
$$ y = \frac mkv_{y_0}(1-e^{-\frac kmt})$$  $$ y_t = y_{t_0} + v_{t_y}t$$

# 代码实现 
## 1、基类
```cpp
template<typename T>
class BulletSolver {
 public:
  BulletSolver(T resistance_coff, T g, T delay, T dt, T timeout) :
      resistance_coff_(resistance_coff),
      dt_(dt), g_(g), delay_(delay),
      timeout_(timeout) {};
  virtual ~BulletSolver() = default;
  virtual void setTarget(const T *pos, const T *vel) = 0;
  virtual void setBulletSpeed(T speed) { bullet_speed_ = speed; };
  virtual void solve(const T *angle_init) = 0;
  virtual void output(T *angle_solved) = 0;
 protected:
  T bullet_speed_{};
  T resistance_coff_, g_, dt_, timeout_, delay_;
};
```
BulletSolver类是所有模型以及算法的基类，定义了实现求解子弹发射角度的算法函数接口，和空气阻力系数、重力加速度、发射延时、子弹速度等成员变量。

### 函数功能
* setTarget() 

   设置目标点，用于初始化目标点初始位置及速度

* setBulletSpeed() 

   设置子弹速度，用于初始化子弹发射速度

* solve() 

   计算，用于解算最终的枪管角度

* output()

  输出，用于输出最终枪管角度

### 变量说明
* bullet_speed_(`T`)

   子弹初速度

* resistance_coff_(`T`)

   空气阻力系数

* g_(`T`)

   重力加速度

* dt_(`T`)

   循环迭代时间间隔

* timeout_(`T`)

   退出循环的条件，当计算到子弹在空中飞行的时间超过此值时将退出循环

* delay_(`T`)

   枪管发射延时

## 2、子弹运动模型
```cpp
rt_bullet_rho = (1 / this->resistance_coff_) * bullet_v_rho
        * (1 - std::exp(-this->fly_time_ * this->resistance_coff_));

rt_bullet_z = (1 / this->resistance_coff_)
      * (bullet_v_z + this->g_ / this->resistance_coff_)
      * (1 - std::exp(-this->fly_time_ * this->resistance_coff_))
      - this->fly_time_ * this->g_ / this->resistance_coff_;        
```
### 变量说明
* bullet_v_rho(`T`)

   子弹速度$v_x$与$v_y$的叠加

* rt_bullet_rho(`T`)

  子弹实际位置在x轴分量和在y轴分量的叠加

* rt_bullet_z(`T`)

   子弹实际位置在z轴分量

* fly_time_(`T`)

   子弹的飞行时间

## 3、目标点运动模型
```cpp
rt_target_x += this->target_dx_ * this->dt_;
rt_target_y += this->target_dy_ * this->dt_;
```

### 变量说明
* rt_target_x(`T`)

   目标点实际位置在x轴分量

* rt_target_y(`T`)

   目标点实际位置在y轴分量

* target_dx_(`T`)

   目标点速度在x轴分量

* target_dy_(`T`)

   目标点速度在y轴分量

所有算法具体实现请参考[bullet_solver.cpp]()

# 测试程序
## 1、包含头文件
```cpp
#include <iostream>
#include "bullet_solver.h"
```
头文件中包含所有类、函数的定义。

## 2、创建类对象实例
```cpp
int main(int argc, char **argv) {
  Iter2DSolver<double> iter2d(0.1, 9.8, 0.01, 0.0001, 3.);
  Approx2DSolver<double> approx2d(0.1, 9.8, 0.01, 0.01, 3.);
  Iter3DSolver<double> iter3d(0.1, 9.8, 0.01, 0.0001, 3.);
  Approx3DSolver<double> approx3d(0.1, 9.8, 0.01, 0.0001, 3.);
  ```

### 变量说明
* iter2d(`Iter2DSolver`)

   2D模型的迭代算法类对象实例

* approx2d(`Approx2DSolver`)

   2D模型的速度叠加算法类对象实例
   
* iter3d(`Iter3DSolver`)

   3D模型的迭代算法类对象实例

* approx3d(`Approx3DSolver`)

   3D模型的速度叠加算法类对象实例

## 3、设置参数
这里以3D模型的迭代算法为例
```cpp
  double angle_init[2]{}, angle_solved[2]{};
  double bullet_speed = 18.;
  double pos_3d[3] = {7, 0, 1};
  double vel_3d[3] = {0, 1, 0};
  iter3d.setBulletSpeed(bullet_speed);
  iter3d.setTarget(pos_3d, vel_3d);
  ```

  ### 参数说明
  * angle_init(`double`)

     自定义的初始发射角度

* angle_solved(`double`)

   计算得出的发射角度

* bullet_speed(`double`)

   子弹发射初速度

* pos_3d(`double[]`)

   目标点初始坐标

* vel_3d(`double[]`)

   目标点在x、y、z方向上的速度

## 4、计算并输出发射角
```cpp
  iter3d.solve(angle_init);
  iter3d.output(angle_solved);
  std::cout << "yaw:" << angle_solved[0] << " pitch:" << angle_solved[1] << std::endl;
```