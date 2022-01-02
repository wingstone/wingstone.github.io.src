---
title: 'Mathematics about camera in graphics（图形学中关于相机的数学）'
date: 2020-06-18 14:34:51
tags: [图形学,engine]
published: true
hideInList: false
feature: 
isTop: false
---

以OpenGL中的右手坐标系为例，介绍引擎中和各种应用中跟相机有关的数学；
<!--more-->

# 实现渲染中的相机

## 观察矩阵

首先理解观察矩阵的作用，观察矩阵是为了将相机位置和转向不同的情况进行统一，最合适的统一方式就是将相机移动到坐标原点，然后将相机朝向变为-Z轴，这样所有的世界在相机看来就是一致的，便于后续的处理；

实际就是将以(0, 0, 0)为原点的世界坐标系转变为以(cameraPos.x, cameraPos.y, cameraPos.z)为原点的坐标系，当然，这里不能少了旋转；

由于矩阵相乘的顺序影响最后的结果，由于旋转矩阵是相对原点进行旋转的，所以自然而然，我们应该先将相机移至原点（平移矩阵）再进行转向（旋转矩阵）；

在OpenGl中，假设相机坐标为cameraPos，朝向分别为Front，Up，Right，坐标为列向量，则相应的矩阵为：
![view矩阵](/assets/img/matrix.gif)

## 投影矩阵

关于投影矩阵的推导，看[这里](http://www.songho.ca/opengl/gl_projectionmatrix.html);

## 相机中非线性0-1的深度转为线性0-1深度

设非线性深度值0-1为depth，线性深度值0-1为lineardepth，Zn表示-1到1的NDC深度，Ze表示观察空间下的深度，n表示近裁剪面，f表示远裁剪面；
则几者之间的关系为：

![depth与NDC](/assets/img/d_zn.gif)

![lineardepth与view](/assets/img/ld_ze.gif)

![NDC与view](/assets/img/zn_ze.gif)

**其中由公式2可以看出，线性的0-1范围并不是指近裁剪面对应0、远裁剪面对应1，而是相机位置对应0，远裁剪面对应1；**

其中公式3可以在上面投影矩阵那篇文章中看到，跟进上面三式，可以得到depth与lineardepth的关系为：

![lineardepth与depth](/assets/img/ld_d.gif)

## 第一人称视角相机的实现

第一人称视角相机，其实就是FPS类游戏中常用的相机，即相机所看的就是游戏中的人眼所看到的，可以自由的前后左右移动，以及左右上下旋转视角；

第一人称视角相机需要存储一些额外的变量，一个是移动的速度MovementSpeed，二是Pitch，Yaw角度，Pitch表示俯仰角，Yaw表示偏航角；

相机自带的变量为Front，Up，Right，cameraPos，worldUp，worldUp表示世界的正上方向；

### 相机的移动实现

```c++
void ProcessKeyBoard(CEMERA_MOVEMENT direction, float deltaTime)
{
    float offset = MovementSpeed * deltaTime;
    switch (direction)
    {
    case FORWARD:
        Position += Front * offset;
        break;
    case BACKWARD:
        Position -= Front * offset;
        break;
    case LEFT:
        Position -= Right * offset;
        break;
    case RIGHT:
        Position += Right * offset;
        break;
    default:
        break;
    }
}

```

### 相机的旋转实现

```c++
void ProcessMouseMovement(float xOffset, float yOffset, bool focusCenter = true, GLboolean constrainPitch = true)
{
    xOffset *= MouseSensitivity;
    yOffset *= MouseSensitivity;

    Yaw += xOffset;
    Pitch += yOffset;

    if (constrainPitch)
    {
        if (Pitch > 89.0f)
        {
            Pitch = 89.0f;
        }
        if (Pitch < -89.0f)
        {
            Pitch = -89.0f;
        }
    }

    {
        glm::vec3 front;
        front.x = sin(glm::radians(Yaw))*cos(glm::radians(Pitch));
        front.y = sin(glm::radians(Pitch));
        front.z = -cos(glm::radians(Yaw))*cos(glm::radians(Pitch));
        Front = glm::normalize(front);

        Right = glm::normalize(glm::cross(Front, WorldUp));
        Up = glm::normalize(glm::cross(Right, Front));
    }
}
```

## 绕固定物体的第三人称相机的实现

绕固定物体旋转相机，有点类似于unity或maya中，Alt+左键移动移动镜头的那种实现，只不过镜头是绕着所看到的物体旋转的，不是自由旋转的，所以相机要跟根据旋转角度计算相机方向同时计算相机位置；

实际上第三人称相机就是这样实现的，只不过此时固定物体是角色而已；

### 绕固定物体相机的旋转实现

```c++
void ProcessMouseMovement(float xOffset, float yOffset, glm::vec3 focusPosition = glm::vec3(0.0f), float focusDistance = 5.0f, GLboolean constrainPitch = true)
{
    xOffset *= MouseSensitivity;
    yOffset *= MouseSensitivity;

    Yaw += xOffset;
    Pitch += yOffset;

    if (constrainPitch)
    {
        if (Pitch > 89.0f)
        {
            Pitch = 89.0f;
        }
        if (Pitch < -89.0f)
        {
            Pitch = -89.0f;
        }
    }

    {
        glm::vec3 front;
        front.x = sin(glm::radians(Yaw))*cos(glm::radians(Pitch));
        front.y = sin(glm::radians(Pitch));
        front.z = -cos(glm::radians(Yaw))*cos(glm::radians(Pitch));
        Front = glm::normalize(front);

        Right = glm::normalize(glm::cross(Front, WorldUp));
        Up = glm::normalize(glm::cross(Right, Front));
    }

    Position = focusPosition - Front * focusDistance;            //区别在在这里
}
```

# 离线渲染中的相机

离线渲染中并不涉及到View矩阵与Project矩阵，主要涉及到如何使用相机参数生成相应的射线；关于射线的计算与具体的相机类型有关；

## 透视相机/针孔相机（Pin Hole）

针孔相机就是传统的透视相机，由于针孔无限小，这样从针孔投射的光线是唯一的，针对投影平面来说；投射光线的唯一也就导致无法多采样产生景深效果；

透视相机的射线，只需要将相机坐标作为起点，相机与屏幕像素的连线作为方向即可；考虑到抗拒齿问题，一般还需要在像素区域内进行随机采样；
```C++
//归一化屏幕坐标，范围：0~1
Ray getRay(float screenX, float screenY, float randx, float randy)
{
    vector<float> dir = _up * std::tan(_fov / 2) *(screenY*2.0f-1.0f) + _right * std::tan(_fov / 2)*_aspect*(screenX*2.0f-1.0f) + _front;		//射线从origin发射
    dir = Normalize(dir);
    return Ray(_origin, dir);
}
```
## 正交相机

正交相机的射线，只需要将相机坐标作为起点，起点的坐标沿屏幕像素进行偏移即可；考虑到抗拒齿问题，一般还需要在像素区域内进行随机采样；
```C++
//归一化屏幕坐标，范围：0~1
Ray getRay(float screenX, float screenY, float randx, float randy)
{
    point origin = _origin + _right * (_aspect*(screenX*2.f - 1.f)) + _up * (screenY*2.f - 1.f);		//射线从像素发射
    return Ray(origin, _front);
}
```

## 环境相机

环境相机实际就是将像素的x、y坐标映射到体积角上，然后计算出相应体积角下的射线；整个像素映射下来刚好覆盖整个4pi体积角；
```C++
Ray getRay(float screenX, float screenY, float randx, float randy)
{
    float theta = PI*(1.f - screenY);
    float phi = 2*PI * screenX;
    vector<float> dir = _right * (std::sin(theta)*std::cos(phi)) + _front * (std::sin(theta)*std::sin(phi)) + _up * std::cos(theta);
    return Ray(_origin, dir);
}
```

## 薄透镜相机（Thin Lens）

薄透镜相机是对传统透镜的模拟，只不过忽略了透镜的实际厚度影响；为了模拟真实的相机，需要加入**光圈**、**焦距**、**焦平面距透镜距离（像距）**；整个相机的[示意图](http://www.360doc.com/content/18/0104/17/50354283_719051589.shtml)，光圈所在位置实际就是透镜所在位置；关键的问题是如何根据相机模型，以及屏幕（像平面）上坐标获取所发射出来的射线；过程如下：

 - 在透镜（光圈）上均匀采样，作为光线的起点，来获取所有经过光圈的光线；
 - 为了模拟对焦，像平面上坐标点与透镜中心的连线，相交于焦平面一点；此点即为所有采样射线必须经过的焦点；
 - 前面两个步骤所获取的光圈采样点、焦平面焦点即可决定采样的光线；

```C++
//透镜描述
float _apertureRedius;	//光圈半径
float _focusDistance;		//焦距
float _imageDistance;		//像距

//画布描述
float _height;
float _aspect;

//针孔成像后为倒立图像，此处并没有对倒立过程进行纠正
Ray getRay(float screenX, float screenY, float randx, float randy)
{
    point screenPoint = _origin + _right * (_aspect*(screenX*2.f - 1.f)*_height) + _up * ((screenY*2.f - 1.f)*_height);
    point lensPoint = _origin + _front * _imageDistance;
    point focusPoint = lensPoint + (lensPoint - screenPoint) * (_focusDistance / _imageDistance);

    point origin = lensPoint + _right * (std::sqrt(randx) * std::cos(randy*2.f*PI)) + _up * (std::sqrt(randx) * std::sin(randy*2.f*PI)) *_apertureRedius;	//射线从透镜发射
    vector<float> dir = Normalize(focusPoint - origin);
    return Ray(origin, dir);
}
```

## 猫眼效应（Cat's Eye Effect）与暗角（vignette）

猫眼效应与暗角形成的原因，要从实际的相机模型来思考；实际的相机镜头是圆筒状的，这就导致光圈中心的光通量比较充足，而沿着光圈边缘的光线有很大概率会被镜筒遮挡住；从而导致屏幕边缘的亮度较弱，且会产生轻微畸变；

知道原理后，模拟方法就比较简单了：

1. 我们先设定镜筒的长度（镜头头部距光圈的距离）、以及镜筒的半径，实际相机的长度不会很长，但可以设置很长来产生明显的效果；
2. 计算投射光线与镜头头部平面的交点，然后判断该交点是否在镜筒半径范围内，如果大于该范围，则舍弃该光线；

## 任意形状的虚化光斑

实际的相机是放置相应的透光图案至于镜头前产生该效果（及镜筒的头部）

模拟方法一：我们只需要在光圈采样时，能在相应图案的范围内进行采样即可；
模拟方法二：根据实际产生的原理，去计算采样光线与镜头头部平面的交点，然后判断该交点是否在相应图案内，舍弃图案外的光线即可；

## 鱼眼镜头（Fisheye Lens）

一般的投射相机在广角很大时，会产生很大的透视畸变；采用鱼眼镜头可以处理超过后180度广角的情况，和一般的镜头不一样，鱼眼镜头拍摄的相片允许很重的Barrel Distortion，也就是说现实中的一条直线在相片中允许被扭曲成弯曲的形状。

鱼眼镜头类似于环境相机，只不过同样是将像素坐标映射到球坐标系，两者的映射方法不同；环境相机是将像素坐标x、y作为球坐标系下的两个夹角维度；而鱼眼镜头的映射要看具体的模拟方法；

鱼眼镜头的一种模拟方法叫做等立体角投影（Equisolid Angle Projection）；等立体角投影的定义是，图像上每个像素位置到图像中心位置的距离与此像素出射光线方向和相机方向间的夹角大小成正比。也就是说，每个像素所覆盖的立体角大小是相等的。

 模拟方法为：

1. 先根据与相机方向夹角随着到图像中心距离线性变换的定义，由像素到图像中心的距离计算出出射光线在球坐标系中的theta（0-pi）。
2. 接着再根据像素位置距离图像中心的水平距离和垂直距离求出出射光线在极坐标系中的phi（0-2pi）；


## 参考文章

[learnopengl-cn](https://learnopengl-cn.github.io/01%20Getting%20started/09%20Camera/)
[Unity Shader入门精要](https://book.douban.com/subject/26821639/)
[OpenGL Projection Matrix](http://www.songho.ca/opengl/gl_projectionmatrix.html)
[离线渲染中的相机](https://zhuanlan.zhihu.com/behindthepixels)，
