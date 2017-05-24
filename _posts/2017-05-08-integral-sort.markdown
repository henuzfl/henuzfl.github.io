---
layout:     post
title:      "拾遗--使用红黑树实现用户积分的实时排名"
subtitle:   "使用红黑树实现用户积分的实时排名"
date:       2017-05-08 12:00:00
author:     "zfl"
header-img: "img/post_data_structure.jpg"
header-mask: 0.6
catalog:    true
tags:
    - 数据结构
    - 搜索树
    - 红黑树
    - 算法
    - python
    - 拾遗
---

# 拾遗-使用红黑树实现用户积分的实时排名
## 问题描述
当年写这个问题的代码应该是看到了一个面试题，时间过去太久原题已经忘却了，这几天，又重新梳理了代码逻辑，当然了又加上了自己新的一些思考。  
在好多系统中，都有用户积分这个东西，特别是各种网游，实时的显示着你当前的排名，还有就是排在榜首的多少多少位，这当然是一种满足玩家心理需求的一种伎俩，玩游戏的时候最需要的就是即时反馈，我花10分钟打死了这个怪，就要看到我积分的变化和排名的变化，要不然可就提不起玩下去的兴趣了；还要实时的查看各种榜单，给玩家一个冲击榜单的诱惑。  
由此，我暂时引出了以下需求：  
1. 用户当前的实时排名；  
2. 用户积分的变更；
3. 实时得到排名前N位的用户。  

## 问题实现  
针对于需求，首先模拟了尽可能真实的数据以方面后面进行测试，然后使用了两种解决方案来实现。
### 模拟数据   
本文采用了Mongodb数据库来持久化数据，使用python的pymongo来实现，模拟插入10w条数据，积分的最大值为5w，为了最大程度的模拟真实环境，根据2/8原则，有百分之二十的人在百分之八十的高分区，有百分之八十的人在百分之二十低分区，随机生成一些名字和积分插入到数据库中:  
```python
def dataInputDB(self):
    n = 100000
    global maxIntegral
    for i in range(n):
        integral = random.randint(int(maxIntegral*0.8),maxIntegral) if random.random() > 0.8 else random.randint(0,int(maxIntegral*0.2))
        user = User(self.getRandomUserName(),integral)
        self.insertUser(user)
```
### 直接使用SQL语句实现   
首先想到的即使直接通过SQL语句来实现，这个实现起来简单明了，获得TopN的用户，直接按照积分倒序查找前N条记录就OK了：  
```python
def getTopNUsers(self,n):
    topNUsers = []
    for u in self.UserColl.find({},{"name":1}).limit(n).sort("integral",-1):
        topNUsers.append(u.get("name"))
    return topNUsers
```  
查询用户的实时排名，这个排名包括两个部分，一部分是积分比他多的用户数，另一部分是积分和他相同用户中的排名（这部分就按照插入数据库的先后顺序）：
```python
def getUserRank(self,name):
    user = self.UserColl.find_one({"name":name})
    rank = 0
    for userObj in self.UserColl.find({"integral":user.get("integral")}):
        rank += 1
        if name == userObj.get("name"):
            break
    rank += self.UserColl.count({"integral":{"$gt":user.get("integral")}})
    return rank
```  
通过这种方式实现比较简单，针对于小规模的用户量不失为一种好的解决方案，具体的实现代码：[integralRankByDB](https://github.com/henuzfl/funcCode/blob/master/integralRankByDB.py),但是，在大规模大并发的情况下，当查询排名的同时有插入数据就会出现锁表的情况，虽然mongodb的粒度比较小，可能会影响小一点。另一方面，需求要求的是实时，每次读取硬盘效率也会不高，如果能放到内存中就好了。  

## 使用红黑树来实现  
既要使用内存来实现，使用何种的数据结构来组织数据就特别重要了，最好数据结构本身就可以满足需求，第一个就想到了一种**二叉搜索树**来实现，这个就非常友好了，内存占用率也不高，插入、删除、积分变更、排名、TopN的时间复杂度都为$$ O(log(h)) $$(h表示二叉树的高度)，要实现排名功能的话序我们还需要在每个节点上增加一个count的字段，这个count字段表示通过这个节点的所有节点数目，利用每个节点的右子树和父节点的右子树都比它大的原理就可以实现排序了，分两种情况：  
1. 如果是父节点的右儿子，则直接递归到父节点   
2. 如果是父节点的左儿子，则排名要加上父节点的右儿子的count值和父节点本身的个数1。如图实现节点3的排名所递归的路径：  

![tree1.png](http://upload-images.jianshu.io/upload_images/730879-878d5b1580b4dcc6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  

TopN问题的话就非常好解决了，首先找到最大值，然后再递归的找节点的前驱，直到找到N个节点。具体的二叉搜索树的代码参见[BinaryTreee](https://github.com/henuzfl/funcCode/blob/master/BinarySearchTree.py)，这个代码很简单，我参照的是《算法导论》第十二章。  
二叉搜索树的缺陷也显而易见，n个节点在最好的情况下高度h=log(n)，最坏的情况高度h=n，而插入、删除、变更、排名、TopN等操作都依赖h，由此呢，我们使用在最坏的情况下也保证h=log(n)的红黑树作为数据结构。  
红黑树的具体原理和实现这里不赘述，参见《算法导论》第十三章，因为真的很复杂，在没有参考的情况下很难写出完整的实现过程，据说《算法导论》的作者自己也说自己写不出来，足以见其变态程度。为了能够实现上述功能，以下我只重点说明一下我改造红黑树的部分：  
1. **节点的改造**  
为了实现排名功能增加了count字段，同时为了实现相同积分的用户将values设定成了list：
```python
class Node(object):

    def __init__(self,key,value):
        self.key = key
        self.values = []
        self.values.append(value)
        self.count = 1
        self.left = None
        self.right = None
        self.parent = None
        self.color = None

    def addNewValue(self,value):
        self.values.extend(value)

    # 当前用户在相同积分的用户中的排名
    def getValueRank(self,value):
        return self.values.index(value) + 1

    def length(self):
        return len(self.values)
```   
2. **左旋和右旋**  
![tree2.png](http://upload-images.jianshu.io/upload_images/730879-a245351415d8fee5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
左旋和右旋也伴随着count值的变化，主要是节点x，y，变化如下：
![tree3.png](http://upload-images.jianshu.io/upload_images/730879-c4d3999c66f443d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
3. **删除节点**  
删除节点要注意递归到root节点count-1   
以上详细的实现代码：[integralRank](https://github.com/henuzfl/funcCode/blob/master/integralRank.py)  
## 总结
在我的渣本上实现对10w数据对两种实现方法做了对比测试，效率提升还是很大的，毕竟是操作内存，使用红黑树的各个操作的时间基本上都在5毫秒以内，直接使用数据库方式都在二三百毫秒左右。  
可以看出使用红黑树在大数据量的情况下不失为一种良好的解决方案，**但是**，做什么时候永远都要说但是，缺点就是红黑树**不支持大并发**呀！每次对红黑树的操作都要锁定树，锁的粒度还是比较大的，有没有更加完美的解决方案呢？很显然是有的，噔噔噔噔，那就是跳表（SkipList），暂时还没有实现，有空搞一下。
