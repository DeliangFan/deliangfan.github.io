---
layout: post
title:  "《人月神话》贵族专制和民主政治章节读后感"
categories: 编程随想
---

>This great church is an incomparable work of art. There is neither aridity nor confusion in the tenets it sets forth. . . . It is the zenith of a style, the work of artists who had understood and assimilated all their predecessors ' successes, in complete possession of the techniques of their times, but using them without indiscreet display nor gratuitous feats of skill.       
>&nbsp;&nbsp;&nbsp;    
>  
>It was Jean d'Orbais who undoubtedly conceived the general plan of the building, a plan which was respected, at least in its essential elements, by his successors. This is one of the reasons for the extreme coherence and unity of the edifice.
>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ———《人月神话》贵族专制和民主政治章节序

大型的软件往往非常复杂，概念之多，功能之多，模块之多，代码之多。如果把大型软件比作成为一座城市，这座城市应该是大而有序，繁而不乱，美观和谐，而不是城中有村，村中有城，建筑风格，囊括中外，道路交通，曲值相交！和构建一个美观有序的大城市一样，软件开发也应当适当的追求一致性，化繁为简，整理归纳，概念应当通俗易懂，功能应当简洁易用，整体布局得当，模块之间和谐共生，研发遵守一定的规则和工具。适当的最求一致性具有多种优点：

- 产品易懂
- 功能易用
- 降低沟通成本
- 提升研发效率
- 降低维护成本，减少不可预知的问题

在追求一致性和产品规模之间应当有所取舍，规模越小，一致性要求降低；规模越大，一致性的要求越高。

从产品的角度降低复杂度

- 类比，贴近生活
- 规律，纲举目张
- 高度抽象的概念
- 功能划分


从研发的角度降低复杂度

- 一致的内核和操作系统
- 尽可能采用一致的编程语言，至少模块内应该统一编程语言，并且注意版本的一致性，同种语言不同版本间的库函数可能会有很大差异；不同的高级语言之间的调用请走网络协议，尽量不要直接调用，如 Python 代码调用 shell 脚本。
- 尽可能一致的数据库，如 mysql 和 postgres 功能类似，但是对一些细节处理不一致，容易引起问题。
- 一致的编码风格，每种编程语言在构建 UT 时加入代码规范检查。
- 统一的文档存放，风格一致的文档说明