#输入: tiedtri.pho  
里面是一些三音素的描述。有logic hmm 和 physical hmm之分，每行的第一个是logic hmm 第二个是 physical hmm 。 如果某一行只有一个，则logic hmm 和 physical hmm 都是它。

#逻辑和物理的解释:
三音素状态种类非常多，实际建模的时候这样会太大，于是用决策树把一些状态进行绑定，把相同类似的三音素绑定在一起，进行一些抽象，于是就会出现一对多的情况。

#调用函数：InitHMMSet  (HModel.c)
主要函数调用关系: InitHMMSet -> CreateHMM -> NewMacro

#InitHMMSet:
	主要用到的结构体：Source 控制文件读取，对回车换行空白等加了一些特定的处理。
	执行流程:从tiedtri.pho中 一次读取一个三音素字串，遇到空格或者换行停止。每行第一个字串当做logic hmm，如果这一行没有第二个字串,则这个字串也是 physical hmm。如果有第二个字串，则第二个字串作为physical hmm。
	三音素字串存储在一个NameCell的结构体中,这个结构体组成的一个hash数组,是一个链式hash表。NameCell里面存放的只有三音素字串和链式hash节点的一个链表。

#CreateHMM:
	
