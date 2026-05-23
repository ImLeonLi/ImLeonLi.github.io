---
layout: post
title: "Overpass API 文档笔记"
date: 2025-01-22 00:00:00 +0800
tags: [OSM, Overpass API, Overpass QL]
category: 技术分享
---

> ⚠️ **注意**：本文为学习笔记，内容较为零散，尚未系统整理。

# Overpass API 文档笔记

Overpass API 文档：<https://wiki.openstreetmap.org/wiki/Overpass_API>

The Overpass API is a read-only API that serves up custom selected parts of the OpenStreetMap (OSM) map data

Overpass API 调试UI：<https://overpass-turbo.eu/>

Overpass QL 查询语言文档：<https://wiki.openstreetmap.org/wiki/Overpass_API/Overpass_QL>

OpenStreetMap 编辑 API：<https://wiki.openstreetmap.org/wiki/API>

在线encodeURIComponent编码工具：<https://tool.oschina.net/encode?type=4>

## xml 查询

### around 标签

例：查询名为 Bristol 节点的附近10米的节点。

```xml
<query type="node">
  <has-kv k="name" v="Bristol"/>
</query>
<around radius="10"/>
<print/>
```

例：查询名为 Bristol 节点附近100米的公交站。

```xml
<query type="node">
  <has-kv k="amenity" v="pub"/>
  <has-kv k="name" v="Bristol"/>
</query>
<query type="node">
  <around radius="100"/>
  <has-kv k="highway" v="bus_stop"/>
</query>
<print/>
```

## Overpass QL 查询示例

### 基本查询

```overpass
[out:json];
node(around:100000,29.733275,-95.377293)["name"="bank"];
out;
```

```overpass
[out:json];
way["highway"](around:1000,29.747646,-95.365705);
out body;
>;
out skel qt;
```

### 查找附近加油站

```overpass
[out:json];
(
  node(around:1500,29.736376,-95.328491)[amenity=fuel];
  way(around:1500,29.736376,-95.328491)[amenity=fuel];
  relation(around:1500,29.736376,-95.328491)[amenity=fuel];
);
out body;
>;
out skel qt;
```

### 使用 id 查询位置

```overpass
[out:json];
way(364815924);
out geom;
```

### 查询 bbox 范围内的道路

```overpass
[out:json];
(
  way["highway"](29.7292746, -95.3270455, 29.7296956, -95.3265719);
);
out geom;
>;
out skel qt;
```

### 查询指定名称的街道

```overpass
[out:json];
(
  way["name"="Telephone Road"];
);
out geom;
>;
out skel qt;
```

### 查询某位置附近的指定名称街道

```overpass
[out:json];
way(around:1000,39.9042,116.4074)["name"="东长安街"];
out body;
>;
out skel qt;
```
