---
title: Cs61b Proj2
date: 2023-10-20T21:42:22+08:00
draft: false
categories:
- one
tags:
- two
- three
- four
---

[project2的代码在这里！](https://github.com/whxhlgy/cs61b-sp18/tree/main/proj2/byog)

## Lab5: Getting started

[lab5](https://github.com/whxhlgy/cs61b-sp18/tree/main/proj2/byog/lab5)

推荐先做一下lab5，可以熟悉一个tile engine，并且会给很多proj2的提示。

## Project2：phase1

目标：

创造一个充满room和hallway的迷宫！

![](https://pic1.zhimg.com/80/v2-7bdfa899de8a07cefad7037a88b85f69_1440w.png?source=d16d100b)

添加图片注释，不超过 140 字（可选）

我的思维过程如下

First：

从proj2得到的灵感是，既然六边形可以对齐然后堆叠，那么正方形也可以用同样的思路，所以我

1. 先在随机位置创建一个随机大小的方块
  
2. 设定下一个方块的大小（随机）
  
3. 于随机的方向放置下一个方块
  
4. 将当前位置移动到下一个方块的左下角
  
5. 重复过程2～4，直到迷宫增长到一定上限
  

因为空间的size是随机的，所以要么是通道（一条边为1）要么是房间，那就满足了实验要求。

但是从结果来看，并没有很理想，因为我没有考虑到不同房间之间的堆叠，而且因为是从一个点出发，所以很容易在点的附近创建出很大的不规则空洞，最后迷宫很丑。

![](https://pic1.zhimg.com/80/v2-fc1f6c329a8c1504df5be1768d8383b0_1440w.png?source=d16d100b)

map1.0

Second：

这次，我在每次增长一个房间时，都把它们放在一个List<Room>中，然后每次增长时，额外判断一下room和room之间有没有重叠，这样，就避免了room的重叠问题。

这样做有一个弊端，就是容易陷入死胡同，也就是如果当前位置周围没有足够空间时，需要重新resize，然后再判断，如果还不行就一直循环。所以在这里会有一直循环的风险，我的做法是在循环时加入一个loopCnt，循环到一定次数就放弃这个分支，去找另一个分支去（也就是在room集合中再找一个）

```
while (!hasSpaceOnAllDirection(p, s, nextSize, null)) {
		nextSize = randomSize(random);
         loopCnt++;
         if (loopCnt > 4) {
	            p = randomRoom(random).p;
			            loopCnt = 0;
                }
        }
}
```


最后的成品如下，感觉还是不很理想，问题应该出在我的算法始终还是基于随机的room大小，所以很容易出现多个room堆在一起组合成一个怪物的情况。

![](https://pica.zhimg.com/80/v2-2f0f47c44b041c575beb6b81ebcbbc63_1440w.png?source=d16d100b)

map2.0

Third：

这一次，我将之前的方法全部推翻，该用“先生成rooms，再生成hallways”的方法，也就是先在地图上随机生成一定数量的rooms，然后用宽度为1的path去连接他们。

1. room的随机生成非常简单，每一个room的大小、位置都是随机的，如果在该位置不能放置，那么就再生成一次，直到生成的room数量达到上限（roomMax）
  
2. 路径的连接比较麻烦，但是不难
  

首先就是要解决，如何确定两个房间的连接点。[我参考了这篇回答](https://gamedev.stackexchange.com/questions/50570/creating-and-connecting-rooms-for-a-roguelike)，就是根据两个room的相对位置，如果是左右型，那就水平方向上，用Z字型连接，如果是上下型，就上下方向上用垂直连接。连接点的确立就非常直觉化了，就在它们相对的那个面上，随机取一个连接点即可。

![](https://pica.zhimg.com/80/v2-d5203c3b4d25dd435241cbd6513e0b5a_1440w.png?source=d16d100b)

添加图片注释，不超过 140 字（可选）

3. 最后，将生成的rooms按照顺序连接起来即可，因为我们的rooms的生成是散落在各处，所以会导致路径重叠在一起很丑，我用了一个非常简陋的解决办法，就是根据room的横坐标大小排序一下，然后根据排好的顺序连接，这样就能避免两个很远的room进行连接。

![](https://picx.zhimg.com/80/v2-717ba4cb945a477397a43f7dc47baed3_1440w.png?source=d16d100b)

map3.0

4. 最后把locked door加上即可。

  

## Project2：phase2

### Starting

先完成Lab6，了解一下StdDraw的原理，因为phase2中会直接地大量用到。

### Main：Overview

我的文件结构如下，也是我构思整个实验的基础

```
-Core
|_____Main.java
|_____Game.java
|_____World.java
|_____...
```

Main：根据输入的参数启动游戏（是否使用input string）

Game：运行游戏的主题，会包含一个循环，每次循环会根据用户键入的key和鼠标的位置来重新画每一帧。在游戏刚开始时，会初始化整个世界，这个世界可以是新创建的，或者是从上次世界恢复的。

World：这个类提供构建一个世界的主体所需要的所有属性，比如tiles，random，以及player之类的实体的位置，它还提供一系列方法，包括初始化一个世界，以及让player做一些动作（主要是移动）

### Main：Interactivity

根据用户的输入来对世界作出一些改变，这些都在lab6中涉及，我仿照lab6也写了一个drawFrame方法，每一次循环调用一次这个方法，就能画出每一帧的画面，这样能实现交互性。

```
    private void drawFrame(TETile[][] tiles, String info) {
        ter.renderFrame(tiles); // draw the world firstly

        TETile t = tiles[mouseX][mouseY];
        StdDraw.setPenColor(Color.WHITE);

        StdDraw.textLeft(0, HEIGHT, t.description()); // display the info of the tile under the cursor
        if (info != null) {
            StdDraw.textRight(WIDTH, HEIGHT, info);
        }
        StdDraw.show();
    }
```

（注：我对renderFrame做了一些小修改）

### Main：Load and save

关于保存，我将World类implements到serializable接口，然后在每次用:q退出时，将World类保存下来。有一些field是不需要保存的，因为可能只在初始化时被用到，这样的field要加上transient修饰。

### Main：others

让砖块看起来更抽象

![img](https://picx.zhimg.com/80/v2-35df4c6e3f82385fdc6c382fc8154349_1440w.png?source=d16d100b)



![image-20231020220525904](/Users/zhongjunjie/Documents/project/my_hugo_site/content/posts/image-20231020220525904.png)

很多功能还可以添加（但是现在没有，因为一些未知的原因🥺）

比如说

- 捡起一个钥匙然后打开门，然后进入下一个世界等等，以此获得一个积分
  
- 在钥匙旁边放上障碍，可以推箱子移开障碍（可能会无解）
  
- 捡起一个强力道具，可以开辟道路！
