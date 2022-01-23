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

Unity计算平滑法线的简单功能实现；
<!--more-->

如果Unity中Vector3的Equals方法的实现与“==”的实现一致，那么代码将会简洁高效很多，参考[Vector3.Equals](https://docs.unity3d.com/ScriptReference/Vector3.Equals.html);

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
