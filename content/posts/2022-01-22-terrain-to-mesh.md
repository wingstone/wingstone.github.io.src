---
title: "Terrain To Mesh"
author: wingstone
date: 2022-01-22
categories:
- Unity
tags:
- code snippet
metaAlignment: center
coverMeta: out
---

Unity转换Terrain为mesh的简单功能实现；
<!--more-->

```c#
using UnityEngine;
using UnityEditor;

public class TerrainToMeshWindow : EditorWindow
{

    [MenuItem("Window/TerrainToMeshWindow")]
    static void Init()
    {
        TerrainToMeshWindow window = (TerrainToMeshWindow)EditorWindow.GetWindow(typeof(TerrainToMeshWindow));
        window.Show();
    }
    int row = 5;
    int column = 5;
    Terrain terrain = null;

    void OnGUI()
    {
        EditorGUILayout.BeginVertical();
        terrain = EditorGUILayout.ObjectField("terrain", terrain, typeof(Terrain), true) as Terrain;
        row = EditorGUILayout.IntField("row", row);
        column = EditorGUILayout.IntField("column", column);
        if (GUILayout.Button("convert"))
        {
            if( row * column * 6 > 65535)
            {
                Debug.LogError("row or column is too big! index buffer can't use 16 bit");
                return;
            }

            TerrainData terrainData = terrain.terrainData;
            int vertexCount = (row + 1) * (column + 1);

            // vertex && uv
            Vector3[] vertices = new Vector3[vertexCount];
            Vector2[] uvs = new Vector2[vertexCount];
            for (int i = 0; i <= row; i++)
            {
                for (int j = 0; j <= column; j++)
                {
                    float width = terrainData.size.x;
                    float length = terrainData.size.z;

                    float x = (float)i / row * width;
                    float z = (float)j / column * length;
                    float y = terrain.SampleHeight(new Vector3(x, 0, z));

                    vertices[i * (column + 1) + j] = new Vector3(x, y, z);

                    uvs[i * (column + 1) + j] = new Vector2((float)i / row, (float)j / column);
                }
            }

            // index
            int indexCount = row * column * 6;
            int[] indices = new int[indexCount];
            for (int i = 0; i < row; i++)
            {
                for (int j = 0; j < column; j++)
                {
                    indices[(i * column + j) * 6] = i * (column + 1) + j;
                    indices[(i * column + j) * 6 + 1] = i * (column + 1) + j + 1;
                    indices[(i * column + j) * 6 + 2] = (i + 1) * (column + 1) + j;

                    indices[(i * column + j) * 6 + 3] = i * (column + 1) + j + 1;
                    indices[(i * column + j) * 6 + 4] = (i + 1) * (column + 1) + j + 1;
                    indices[(i * column + j) * 6 + 5] = (i + 1) * (column + 1) + j;
                }
            }

            // normal
            Vector3[] normals = new Vector3[vertexCount];
            int[] normalcounts = new int[vertexCount];
            for (int i = 0; i < vertexCount; i++)
            {
                normals[i] = Vector3.zero;
                normalcounts[i] = 0;
            }
            for (int i = 0; i < row; i++)
            {
                for (int j = 0; j < column; j++)
                {
                    int vid0 = i * (column + 1) + j;
                    int vid1 = i * (column + 1) + j + 1;
                    int vid2 = (i + 1) * (column + 1) + j;
                    int vid3 = (i + 1) * (column + 1) + j + 1;

                    Vector3 normal0 =Vector3.Cross(vertices[vid2]-vertices[vid0], vertices[vid2]-vertices[vid0]).normalized;

                    Vector3 normal1 =Vector3.Cross(vertices[vid3]-vertices[vid1], vertices[vid2]-vertices[vid1]).normalized;

                    normals[vid0] += normal0;
                    normals[vid1] += normal0;
                    normals[vid2] += normal0;
                    normalcounts[vid0]++;
                    normalcounts[vid1]++;
                    normalcounts[vid2]++;

                    normals[vid1] += normal1;
                    normals[vid2] += normal1;
                    normals[vid3] += normal1;
                    normalcounts[vid1]++;
                    normalcounts[vid2]++;
                    normalcounts[vid3]++;
                }
            }
            for (int i = 0; i < vertexCount; i++)
            {
                normals[i] /= normalcounts[i];
                normals[i].Normalize();
            }

            Mesh mesh = new Mesh();
            mesh.name = terrain.name;
            mesh.vertices = vertices;
            mesh.uv = uvs;
            mesh.triangles = indices;
            mesh.normals = normals;

            AssetDatabase.CreateAsset(mesh, "Assets/test.asset");
            AssetDatabase.Refresh();
        }
        EditorGUILayout.EndVertical();
    }
}
```
