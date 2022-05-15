---
title: "Mesh Normal Smooth"
author: wingstone
date: 2022-01-22
categories:
- Unity
tags:
- code snippet
metaAlignment: center
coverMeta: out
---

Unity平滑法线的一些实现方法；
<!--more-->

## Version1

只对顶点索引重叠的顶点进行法线平滑，即ReCalculateNormal：

```c#
using UnityEngine;
using UnityEditor;
using System.Collections.Generic;

public class MeshNormalSmoothWindow : EditorWindow
{
    [MenuItem("Window/MeshNormalSmoothWindow")]
    static void Init()
    {
        MeshNormalSmoothWindow window = (MeshNormalSmoothWindow)EditorWindow.GetWindow(typeof(MeshNormalSmoothWindow));
        window.Show();
    }

    Mesh mesh;

    void OnGUI()
    {
        EditorGUILayout.BeginVertical();
        mesh = EditorGUILayout.ObjectField("mesh", mesh, typeof(Mesh), true) as Mesh;
        if (GUILayout.Button("convert"))
        {
            Vector3[] vertices = mesh.vertices;
            int[] triangles = mesh.triangles;

            int polygonNum = mesh.triangles.Length / 3;
            Vector3[] polyNormals = new Vector3[polygonNum];
            for (int i = 0; i < polygonNum; i++)
            {
                int vid0 = triangles[i * 3];
                int vid1 = triangles[i * 3 + 1];
                int vid2 = triangles[i * 3 + 2];

                // 乘以1000避免数值过小时计算结果为0
                polyNormals[i] = Vector3.Cross((vertices[vid1] - vertices[vid0]) * 1000, (vertices[vid2] - vertices[vid0]) * 1000).normalized;
            }

            int vertexNum = mesh.vertices.Length;
            Vector3[] normals = new Vector3[vertexNum];
            int[] normalsNum = new int[vertexNum];
            for (int i = 0; i < vertexNum; i++)
            {
                normals[i] = Vector3.zero;
            }
            for (int i = 0; i < polygonNum; i++)
            {

                int vid0 = triangles[i * 3];
                int vid1 = triangles[i * 3 + 1];
                int vid2 = triangles[i * 3 + 2];

                normals[vid0] += polyNormals[i];
                normals[vid1] += polyNormals[i];
                normals[vid2] += polyNormals[i];
                normalsNum[vid0]++;
                normalsNum[vid1]++;
                normalsNum[vid2]++;
            }
            for (int i = 0; i < vertexNum; i++)
            {
                normals[i] /= normalsNum[i];
                normals[i].Normalize();
            }

            Mesh newmesh = new Mesh();
            newmesh.vertices = mesh.vertices;
            newmesh.triangles = mesh.triangles;
            newmesh.normals = normals;
            newmesh.uv = mesh.uv;
            newmesh.uv2 = mesh.uv2;
            newmesh.boneWeights = mesh.boneWeights;
            newmesh.bindposes = mesh.bindposes;

            AssetDatabase.CreateAsset(newmesh, "Assets/test.asset");
            AssetDatabase.Refresh();
        }
        EditorGUILayout.EndVertical();
    }
}
```

## Version2

只对顶点重叠的顶点进行法线平滑，即对Lowpoly添加光滑组：

```c#
using UnityEngine;
using UnityEditor;
using System.Collections.Generic;

public class MeshNormalSmoothWindow : EditorWindow
{
    [MenuItem("Window/MeshNormalSmoothWindow")]
    static void Init()
    {
        MeshNormalSmoothWindow window = (MeshNormalSmoothWindow)EditorWindow.GetWindow(typeof(MeshNormalSmoothWindow));
        window.Show();
    }

    Mesh mesh;

    void OnGUI()
    {
        EditorGUILayout.BeginVertical();
        mesh = EditorGUILayout.ObjectField("mesh", mesh, typeof(Mesh), true) as Mesh;
        if (GUILayout.Button("convert"))
        {
            Vector3[] vertices = mesh.vertices;
            int[] triangles = mesh.triangles;

            int polygonNum = mesh.triangles.Length / 3;
            Vector3[] polyNormals = new Vector3[polygonNum];
            for (int i = 0; i < polygonNum; i++)
            {
                int vid0 = triangles[i * 3];
                int vid1 = triangles[i * 3 + 1];
                int vid2 = triangles[i * 3 + 2];

                // 乘以1000避免数值过小时计算结果为0
                polyNormals[i] = Vector3.Cross((vertices[vid1] - vertices[vid0])*1000, (vertices[vid2] - vertices[vid0])*1000).normalized;
            }

            int vertexNum = mesh.vertices.Length;
            Vector3[] normals = new Vector3[vertexNum];
            for (int i = 0; i < vertexNum; i++)
            {
                normals[i] = Vector3.zero;
            }
            for (int i = 0; i < polygonNum; i++)
            {

                int vid0 = triangles[i * 3];
                int vid1 = triangles[i * 3 + 1];
                int vid2 = triangles[i * 3 + 2];

                // 这里可以使用Dictionary来进行优化，那么不用进行整个顶点的遍历；
                // 因此需要将Vector3作为Dictionary的Key来使用；
                // Dictionary中添加自定义Key的方法为：
                // https://stackoverflow.com/questions/6999191/use-custom-object-as-dictionary-key
                // https://www.codeproject.com/Articles/23610/Dictionary-with-a-Custom-Key
                // 因为Vector3的Equals方法是精确匹配，只有“==”的实现是非精确匹配；因此不适合使用Vector3来作为Key使用；
                // 参考https://docs.unity3d.com/ScriptReference/Vector3.Equals.html
                for (int j = 0; j < vertexNum; j++)
                {
                    if(vertices[j] == vertices[vid0])
                    {
                        normals[j] += polyNormals[i];
                    }
                    if(vertices[j] == vertices[vid1])
                    {
                        normals[j] += polyNormals[i];
                    }
                    if(vertices[j] == vertices[vid2])
                    {
                        normals[j] += polyNormals[i];
                    }
                }

            }
            for (int i = 0; i < vertexNum; i++)
            {
                normals[i].Normalize();
            }

            Mesh newmesh = new Mesh();
            newmesh.vertices = mesh.vertices;
            newmesh.triangles = mesh.triangles;
            newmesh.normals = normals;
            newmesh.uv = mesh.uv;
            newmesh.uv2 = mesh.uv2;
            newmesh.boneWeights = mesh.boneWeights;
            newmesh.bindposes = mesh.bindposes;

            AssetDatabase.CreateAsset(newmesh, "Assets/test.asset");
            AssetDatabase.Refresh();
        }
        EditorGUILayout.EndVertical();
    }
}
```

## Normal Smooth Method

在前面的version1与version2中，使用的都是直接针对当前点进行面法线平均的算法；代码如下：

```c#
for (int i = 0; i < polygonNum; i++)
{
    int vid0 = triangles[i * 3];
    int vid1 = triangles[i * 3 + 1];
    int vid2 = triangles[i * 3 + 2];

    for (int j = 0; j < vertexNum; j++)
    {
        if(vertices[j] == vertices[vid0])
        {
            normals[j] += polyNormals[i];
            normalsNum[j]++;
        }
        if(vertices[j] == vertices[vid1])
        {
            normals[j] += polyNormals[i];
            normalsNum[j]++;
        }
        if(vertices[j] == vertices[vid2])
        {
            normals[j] += polyNormals[i];
            normalsNum[j]++;
        }
    }
}
for (int i = 0; i < vertexNum; i++)
{
    normals[i].Normalize();
}
```

这样会导致平滑法线偏向面数比较多的方向，如果三角面数量在同一点上分布不均匀，就会产生比较大的瑕疵；因此可以考虑使用**三角形的面积来进行加权平均**；代码如下：

```c#
for (int i = 0; i < polygonNum; i++)
{
    int vid0 = triangles[i * 3];
    int vid1 = triangles[i * 3 + 1];
    int vid2 = triangles[i * 3 + 2];

    for (int j = 0; j < vertexNum; j++)
    {
        if(vertices[j] == vertices[vid0])
        {
            normals[j] += polyNormals[i]*polyArea[i];
            normalsNum[j]++;
        }
        if(vertices[j] == vertices[vid1])
        {
            normals[j] += polyNormals[i]*polyArea[i];
            normalsNum[j]++;
        }
        if(vertices[j] == vertices[vid2])
        {
            normals[j] += polyNormals[i]*polyArea[i];
            normalsNum[j]++;
        }
    }
}
for (int i = 0; i < vertexNum; i++)
{
    normals[i].Normalize();
}
```

三角形的面积来进行加权平均仍然会有同样的问题，就是如果三角面面积在同一点上分布不均匀，同样会产生比较大的瑕疵，有时甚至会比非面积加权的更加严重；
最终的解决方案，应该是针对同一点，使用对应三面在该点的夹角来进行平均，这样就将法线的影响锁定到了局部，不会受其他非有效因素的影响；代码如下：

```c#
for (int i = 0; i < polygonNum; i++)
{
    int vid0 = triangles[i * 3];
    int vid1 = triangles[i * 3 + 1];
    int vid2 = triangles[i * 3 + 2];

    for (int j = 0; j < vertexNum; j++)
    {
        if(vertices[j] == vertices[vid0])
        {
            normals[j] += polyNormals[i]*polyAngle[i];
            normalsNum[j]++;
        }
        if(vertices[j] == vertices[vid1])
        {
            normals[j] += polyNormals[i]*polyAngle[i];
            normalsNum[j]++;
        }
        if(vertices[j] == vertices[vid2])
        {
            normals[j] += polyNormals[i]*polyAngle[i];
            normalsNum[j]++;
        }
    }
}
for (int i = 0; i < vertexNum; i++)
{
    normals[i].Normalize();
}
```

## Reference

1. [Weighted Vertex Normals](http://www.bytehazard.com/articles/vertnorm.html)
2. [use custom object as dictionary key](https://stackoverflow.com/questions/6999191/use-custom-object-as-dictionary-key)
3. [Dictionary with a Custom Key](https://www.codeproject.com/Articles/23610/Dictionary-with-a-Custom-Key)
4. [Vector3.Equals](https://docs.unity3d.com/ScriptReference/Vector3.Equals.html)
5. [General method for calculating Smooth vertex normals with 100% smoothness](https://stackoverflow.com/questions/45477806/general-method-for-calculating-smooth-vertex-normals-with-100-smoothness)