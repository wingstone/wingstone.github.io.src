---
title: 'Unity中序列化Asset的修改'
date: 2021-02-28 23:31:35
tags: [code snippet]
published: true
hideInList: false
feature: 
isTop: false
---
代码片段事例：
```
        Object obj = new Object();
        SerializedObject so = new SerializedObject(obj);     //这里obj为要进行序列化操作的asset对象
        SerializedProperty sp = so.FindProperty("property");    //这里property为要进行操作的property，需要查看asset文件获取
        sp.boolValue = true;
```