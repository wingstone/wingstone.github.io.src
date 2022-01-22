---
title: "Mesh Normal Smooth"
author: wingstone
date: 2022-01-022
categories:
- Unity
tags:
- code snippet
metaAlignment: center
coverMeta: out
---

Unity计算平滑法线的简单功能实现
<!--more-->

```c++
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

                polyNormals[i] = Vector3.Cross(vertices[vid2] - vertices[vid0], vertices[vid1] - vertices[vid0]).normalized;
            }

            int vertexNum = mesh.vertices.Length;
            Vector3[] normals = new Vector3[vertexNum];
            List<Vector3> hashvertexs = new List<Vector3>();
            List<Vector3> hashnormals = new List<Vector3>();
            List<int> hasnormalsNums = new List<int>();
            for (int i = 0; i < vertexNum; i++)
            {
                normals[i] = Vector3.zero;
            }
            for (int i = 0; i < polygonNum; i++)
            {

                int vid0 = triangles[i * 3];
                int vid1 = triangles[i * 3 + 1];
                int vid2 = triangles[i * 3 + 2];

                bool have0 = false;
                bool have1 = false;
                bool have2 = false;
                for (int j = 0; j < hashvertexs.Count; j++)
                {
                    if (hashvertexs[j] == vertices[vid0])
                    {
                        have0 = true;
                        hasnormalsNums[j]++;
                        hashnormals[j] += polyNormals[i];
                    }
                    if (hashvertexs[j] == vertices[vid1])
                    {
                        have1 = true;
                        hasnormalsNums[j]++;
                        hashnormals[j] += polyNormals[i];
                    }
                    if (hashvertexs[j] == vertices[vid2])
                    {
                        have2 = true;
                        hasnormalsNums[j]++;
                        hashnormals[j] += polyNormals[i];
                    }
                }

                if (!have0)
                {
                    hashvertexs.Add(vertices[vid0]);
                    hashnormals.Add(polyNormals[i]);
                    hasnormalsNums.Add(1);
                }
                if (!have1)
                {
                    hashvertexs.Add(vertices[vid1]);
                    hashnormals.Add(polyNormals[i]);
                    hasnormalsNums.Add(1);
                }
                if (!have2)
                {
                    hashvertexs.Add(vertices[vid2]);
                    hashnormals.Add(polyNormals[i]);
                    hasnormalsNums.Add(1);
                }

            }
            for (int i = 0; i < hashnormals.Count; i++)
            {
                hashnormals[i] /= hasnormalsNums[i];
                hashnormals[i].Normalize();
            }
            for (int i = 0; i < vertexNum; i++)
            {
                for (int j = 0; j < hashvertexs.Count; j++)
                {
                    if (vertices[i] == hashvertexs[j])
                        normals[i] = -hashnormals[j];
                }
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