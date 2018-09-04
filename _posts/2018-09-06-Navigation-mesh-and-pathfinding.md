---
layout:     post
title:      "Navigation mesh and pathfinding"
date:       2018-09-06 00:00:00+0800
categories: [study]
tags:       [navmesh,pathfinding,recast,astar]
---

# NavMesh的生成

Unity的navmesh很好用，但是因为其代码不可控的原因，项目希望找一个代码可控的库，于是找到了`A*`，在其pro版本中提供了一种recast graph，这正是Unity所使用的库。

Recast navmesh的生成过程：
1. 体素化（voxelization）
1. 生成高度场（heightfield）
1. 划分区域（region）
1. 生成轮廓（contour）
1. 生成polymesh
1. *生成detailmesh

![](http://www.critterai.org/projects/nmgen_study/media/images/gen_01_conservvox.jpg)

体素就是3d的像素。一个矩形空间x和z以单位为cellsize、y以单位为cellheight转化为体素空间。
物体由顶点和三角形组成，每个三角形在体素空间里可以获得其三个顶点的体素坐标，由此完成体素化。

![](http://www.critterai.org/projects/nmgen_study/media/images/ohfg_01_ohfgen.jpg)

当地面、楼梯等地形完成体素化后，会用高度场来表示。
遍历体素空间里的每一个x、z坐标，从体素空间y最小开始向上观测直到碰到非实体部分（作为floor），继续向上碰到实体部分（作为ceiling），这样一段用span表示，如果有多层楼，则同一个x、z坐标会有多个span，其中最高的一个span若没有ceiling，则ceiling为无限高。
高度场的遍历，即遍历每一个x、z坐标，然后自底往上遍历span并检测span的region（后续会加入），由此准确取得该span的高度信息，也即场景中某一个位置的准确信息。

![](http://www.critterai.org/projects/nmgen_study/media/images/stage_regions.gif)

区域通过分水岭算法进行划分，然后为每个span分配region的id

![](http://www.critterai.org/projects/nmgen_study/media/images/cont_09_nullregionsimp02.png)

完成区域划分后对区域进行勾轮廓，因为在体素空间里，每一个位置都是体素，所以勾出的轮廓一定是锯齿的，于是需要对其进行简化。
一条轮廓为n个顶点的连线，初始为2个顶点，遍历轮廓上的顶点，如果所有顶点到该线的距离小于可接受距离，则生成最终轮廓；如果存在距离大于可接受距离，则找出最大差异的顶点，将其加入轮廓顶点，进行迭代。

![](http://www.critterai.org/projects/nmgen_study/media/images/dm_04_edges03.png)
![](http://www.critterai.org/projects/nmgen_study/media/images/dm_05_surface03.png)

勾完的轮廓生成polymesh可以直接当成navmesh来用，但是因为做过简化，所以会出现不贴合原始地形的情况。Recast额外会生成detailmesh去贴合，而`A*`没有额外处理。贴合的方式是分别在多边形的边（edge）进行等距采样、多边形内部进行网格（grid）采样，将采样点与高度场中对应的高度信息进行对比，如果超过可接受，则用高度场的y替换采样点的y，并加入新顶点，最后在xz平面用delaunay三角剖分将所有新老顶点一起生成detailmesh。

`A*`
~~~
CollectMeshes
VoxelizeInput
FilterLedges
FilterLowHeightSpans
BuildCompactField
BuildVoxelConnections
ErodeWalkableArea
BuildDistanceField
BuildRegions
BuildContours
BuildPolyMesh
CreateTile
~~~

Unity
~~~
rcRasterizeTriangles
rcFilterForceUnwalkableArea
rcFilterLowHangingWalkableObstacles
rcFilterLedgeSpans
rcFilterWalkableLowHeightSpans
rcBuildCompactHeightfield
rcMarkConvexPolyhedraArea
rcErodeWalkableArea
rcBuildLayerRegions
rcBuildContours
rcBuildPolyMesh
rcBuildPolyMeshDetail
~~~

|  Recast (rcConfig)   | `Unity` | `A*` |
|:---------------------|:------|:---|
|cs                    |`max(0.01f,cellSize)`|`max(cellSize,0.001F)`|
|ch                    |`max(cs*0.5f,(agentHeight+agentClimb)/255.0f,(bmax-bmin)/65536.0f)`|`max(forcedBoundsSize.y/64000, 0.001f)`|
|walkableSlopeAngle    |`clamp(agentSlope, 0.1f, 90.0f)`|`maxSlope`|
|walkableHeight        |`agentHeight/ch`|`max(walkableHeight,0)`|
|walkableClimb         |`agentClimb/ch`|`min(walkableClimb, walkableHeight)`|
|walkableRadius        |`ceilf(agentRadius/cs-cs*0.01f)`|`voxelCharacterRadius`<br>`=CharacterRadiusInVoxels`<br>`=ceil((characterRadius/cellSize)-0.1f)`|
|maxEdgeLen            |`0`|`maxEdgeLength/cellSize`|
|maxSimplificationError|`1.3f`|`contourMaxError`|
|minRegionArea         |`minRegionArea/(cs*cs))`|`minRegionSize/(cellSize*cellSize)`|
|tileSize              |`256*cs`|`editorTileSize`|
|mergeRegionArea       |`400`|`-`|
|maxVertsPerPoly       |`6`|`3`|
|detailSampleDist      |`cs*6.0f`|`-`|
|detailSampleMaxError  |`ch*1.0`|`-`|
|borderSize            |`walkableRadius+3`|`voxelCharacterRadius+3`|

tileSize(cs=0.15)实测:
+ scale=5的plane，`Unity`为16个顶点和8个面，`A*`设为166时一致，`A*`设为165时为30个顶点和14个面。
+ scale=10的plane，`Unity`为64个顶点和32个面，`A*`设为166时一致，`A*`设为165时为100个顶点和50个面。
+ scale=20的plane，`Unity`为144个顶点和72个面，`A*`设为222时一致，`A*`设为221时为196个顶点和98个面。
+ scale=40的plane，`Unity`为576个顶点和288个面，`A*`设为222时一致，`A*`设为221时为676个顶点和338个面。

<details><summary>NavMeshData</summary>

~~~
using UnityEngine;
using UnityEditor;

public class NavMeshData
{
	[MenuItem("NavMeshData/Dump")]
	static void Dump()
	{
		var data = UnityEngine.AI.NavMesh.CalculateTriangulation();
		Debug.Log("Unity NavMesh contains " + data.vertices.Length + " vertices, " + data.indices.Length / 3 + "faces");

		var nverts = 0;
		var ntris = 0;
		if (AstarPath.active.data != null) {
			if (AstarPath.active.data.recastGraph != null)
			{
				foreach (var tile in AstarPath.active.data.recastGraph.GetTiles())
				{
					nverts += tile.verts.Length;
					ntris += tile.tris.Length;
				}
			} else if (AstarPath.active.data.graphs != null) {
				foreach (var graph in AstarPath.active.data.graphs)
				{
					if (graph.GetType() == typeof(Pathfinding.RecastGraph) || graph.GetType().IsSubclassOf(typeof(Pathfinding.RecastGraph)))
					{
						var g = graph as Pathfinding.RecastGraph;
						foreach (var tile in g.GetTiles())
						{
							nverts += tile.verts.Length;
							ntris += tile.tris.Length;
						}
					}
				}
			}
		}
		Debug.Log("Astar NavMesh contains " + nverts + " vertices, " + (ntris / 3) + " triangles");
	}
}
~~~
</details>


### 参考文献：

1. http://www.critterai.org/projects/nmgen_study/overview.html
1. http://www.critterai.org/projects/nmgen_study/config.html
1. http://www.critterai.org/projects/nmgen_study/heightfields.html
1. http://www.critterai.org/projects/nmgen_study/contourgen.html
1. http://www.critterai.org/projects/nmgen_study/detailgen.html
1. https://github.com/recastnavigation/recastnavigation
1. https://unity3d.com/learn/tutorials/topics/navigation/baking-navmesh-runtime
1. https://docs.unity3d.com/ScriptReference/AI.NavMeshBuildDebugSettings.html
