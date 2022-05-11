---
title: fbx sdk使用总结
author: wingstone
date: 2022-01-15
categories:
- 总结
tags:
- 总结
- fbx
metaAlignment: center
coverMeta: out
---

记录使用fbx sdk过程中所踩得一些坑；
<!--more-->

## 版本问题

不同的fbx sdk版本，要配合不同的vs studio版本使用；sdk版本对应的VS studio版本在下载时会有显示，所有的fbx sdk版本下载点击[fbx sdk archives](https://www.autodesk.com/developer-network/platform-technologies/fbx-sdk-archives);

## Mesh的创建流程小结

vertex与index的创建流程相对固定，如下：

```c++
FbxNode* CreateTriangle(FbxScene* pScene, const char* pName)
{
    FbxMesh* lMesh = FbxMesh::Create(pScene, pName);

    // The three vertices
    FbxVector4 lControlPoint0(-50, 0, 50);
    FbxVector4 lControlPoint1(50, 0, 50);
    FbxVector4 lControlPoint2(0, 50, -50);

    // Create control points.
    lMesh->InitControlPoints(3);
    FbxVector4* lControlPoints = lMesh->GetControlPoints();

    // 这里数组的赋值也可以预先生产好后，直接使用memcpy进行拷贝
    lControlPoints[0] = lControlPoint0;
    lControlPoints[1] = lControlPoint1;
    lControlPoints[2] = lControlPoint2;

    // Create the triangle's polygon    //如果添加4个顶点就是四边形
    lMesh->BeginPolygon();
    lMesh->AddPolygon(0); // Control point 0
    lMesh->AddPolygon(1); // Control point 1
    lMesh->AddPolygon(2); // Control point 2
    lMesh->EndPolygon();

    FbxNode* lNode = FbxNode::Create(pScene,pName);
    lNode->SetNodeAttribute(lMesh);

    return lNode;
}
```

uv、color、normal、tangent、binormal的创建会多一个环节，首先创建对应的Element，然后设置MappingMode与ReferenceMode，最后再设置对应的DirectArray与IndexArray；

```c++
// create UVset
FbxGeometryElementUV* lUVElement1 = lMesh->CreateElementUV("UVSet1");
FBX_ASSERT( lUVElement1 != NULL);
lUVElement1->SetMappingMode(FbxGeometryElement::eByPolygonVertex);
lUVElement1->SetReferenceMode(FbxGeometryElement::eIndexToDirect);
for (int i = 0; i <4; i++)
    lUVElement1->GetDirectArray().Add(FbxVector2(lUVs[i][0], lUVs[i][1]));

for (int i = 0; i<24; i++)
    lUVElement1->GetIndexArray().Add(uvsId[i%4]);
```

MappingMode常用的为FbxGeometryElement::eByControlPoint与FbxGeometryElement::eByPolygonVertex；以uv为例，若使用FbxGeometryElement::eByControlPoint，获取顶点uv的索引，与controlpoint一致；若使用FbxGeometryElement::eByPolygonVertex，获取顶点的索引单独递增即可，与controlpoint无关；

ReferenceMode常用的为FbxGeometryElement::eDirect与FbxGeometryElement::eIndexToDirect；以uv为例，若使用FbxGeometryElement::eDirect，则表示没有IndexArray，前面获取的索引可直接获取DirectArray；若使用FbxGeometryElement::eIndexToDirect，则表示有IndexArray，前面获取的索引是IndexArray的索引，从IndexArray获取的索引才可用来获取DirectArray;

从实际使用来看，对于vertex color的创建，需要绑定到FbxLayer0上，才能有效的创建与修改，即：
```c++
// normal to color
auto LayerZero = pMesh->GetLayer(0);
FbxGeometryElementVertexColor* lVertexColorElement = pMesh->CreateElementVertexColor();
lVertexColorElement->SetMappingMode(FbxGeometryElement::eByPolygonVertex);
lVertexColorElement->SetReferenceMode(FbxGeometryElement::eIndexToDirect);

lVertexColorElement->GetDirectArray().Resize(controlPointCount);
for (size_t i = 0; i < controlPointCount; i++)
{
    lVertexColorElement->GetDirectArray().SetAt(i, normals[i]);
}

lVertexColorElement->GetIndexArray().Resize(indexCount);
int currentIndex = 0;
for (size_t i = 0; i < polygonCount; i++)
{
    int lPolygonSize = pMesh->GetPolygonSize(i);

    for (size_t j = 0; j < lPolygonSize; j++)
    {
        int lControlPointIndex = pMesh->GetPolygonVertex(i, j);
        lVertexColorElement->GetIndexArray().SetAt(currentIndex, lControlPointIndex);
        currentIndex++;
    }
}

LayerZero->SetVertexColors(lVertexColorElement);
```

## Mesh的读取与显示

在fbx sdk中有大量的Sample，其中Sample中的ImportScene展示了如何完整显示fbx的所有信息；这里以显示一个mesh的uv信息为例：
```c++
void LoadUVInformation(FbxMesh* pMesh)
{
    //get all UV set names
    FbxStringList lUVSetNameList;
    pMesh->GetUVSetNames(lUVSetNameList);

    //iterating over all uv sets
    for (int lUVSetIndex = 0; lUVSetIndex < lUVSetNameList.GetCount(); lUVSetIndex++)
    {
        //get lUVSetIndex-th uv set
        const char* lUVSetName = lUVSetNameList.GetStringAt(lUVSetIndex);
        const FbxGeometryElementUV* lUVElement = pMesh->GetElementUV(lUVSetName);

        if(!lUVElement)
            continue;

        // only support mapping mode eByPolygonVertex and eByControlPoint
        if( lUVElement->GetMappingMode() != FbxGeometryElement::eByPolygonVertex &&
            lUVElement->GetMappingMode() != FbxGeometryElement::eByControlPoint )
            return;

        //index array, where holds the index referenced to the uv data
        const bool lUseIndex = lUVElement->GetReferenceMode() != FbxGeometryElement::eDirect;
        const int lIndexCount= (lUseIndex) ? lUVElement->GetIndexArray().GetCount() : 0;

        //iterating through the data by polygon
        const int lPolyCount = pMesh->GetPolygonCount();

        if( lUVElement->GetMappingMode() == FbxGeometryElement::eByControlPoint )
        {
            for( int lPolyIndex = 0; lPolyIndex < lPolyCount; ++lPolyIndex )
            {
                // build the max index array that we need to pass into MakePoly
                const int lPolySize = pMesh->GetPolygonSize(lPolyIndex);
                for( int lVertIndex = 0; lVertIndex < lPolySize; ++lVertIndex )
                {
                    FbxVector2 lUVValue;

                    //get the index of the current vertex in control points array
                    int lPolyVertIndex = pMesh->GetPolygonVertex(lPolyIndex,lVertIndex);

                    //the UV index depends on the reference mode
                    int lUVIndex = lUseIndex ? lUVElement->GetIndexArray().GetAt(lPolyVertIndex) : lPolyVertIndex;

                    lUVValue = lUVElement->GetDirectArray().GetAt(lUVIndex);

                    //User TODO:
                    //Print out the value of UV(lUVValue) or log it to a file
                }
            }
        }
        else if (lUVElement->GetMappingMode() == FbxGeometryElement::eByPolygonVertex)
        {
            int lPolyIndexCounter = 0;
            for( int lPolyIndex = 0; lPolyIndex < lPolyCount; ++lPolyIndex )
            {
                // build the max index array that we need to pass into MakePoly
                const int lPolySize = pMesh->GetPolygonSize(lPolyIndex);
                for( int lVertIndex = 0; lVertIndex < lPolySize; ++lVertIndex )
                {
                    if (lPolyIndexCounter < lIndexCount)
                    {
                        FbxVector2 lUVValue;

                        //the UV index depends on the reference mode
                        int lUVIndex = lUseIndex ? lUVElement->GetIndexArray().GetAt(lPolyIndexCounter) : lPolyIndexCounter;

                        lUVValue = lUVElement->GetDirectArray().GetAt(lUVIndex);

                        //User TODO:
                        //Print out the value of UV(lUVValue) or log it to a file

                        lPolyIndexCounter++;
                    }
                }
            }
        }
    }
}
```

关于fbx sdk应用上的文章，可以阅读[Working with FBX SDK](https://www.cnblogs.com/clayman/archive/2010/12/11/1902782.html)；