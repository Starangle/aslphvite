I:节点标号
W：节点名字
J：弧标号
S：弧开始节点的标号
E：弧结束节点的标号
构造lnode，保存lnode的连接关系，最终实现构造lattice

newlattice初始化lattice
读文件在readlattice以及readonelattice函数中
nn：lnode的数目,即节点数目
na：larc的数目，即弧的数目
并将填充一系列值，拨号系统中为默认，如utterance等
读wdnet文件在readonelattice函数
getnextfieldname函数读取name，getintfield函数读取数值
N的数值存在nn，L数值存在na，即lat->nn,na;
之后判断lat->foemat是否为shortarc，短弧，据情况分配空间并初始化
向下依次读I值，读到n中，定位到lat->lnode+n,即ln，紧接着读W值，读信息到wordID，即在voc->wtab中的位置。
wtab储存word结构体数组，地址通过vocabhash定位，其余字符主要修改format信息，没有用到
读完I W行，即所有node信息之后，读J S E值到n,s,e;
写入LARC结构体la，s对应start，e对应end，la->start和end对应lat->lnodes相应位置，线性
l r对应lmlike和prlike，暂时没有值，其余字母基本是对应format信息，拨号系统中没有

文件读写：N，函数readonelattice中nn
         L，函数readonelattice中na
         I，对应lat->lnode+n,链接到ln，lat为lattice结构，函数返回本结构体，ln、la、nn、na等   	为局部变量
         W，wordID，对应voc->wtab[]Word数组中的值，voc储存全局词典信息
         J，即弧编号，S，E为弧起点和终点，储存到la->start和la->end，并链接到lat->lnodes对应位置节点
         其余信息这个拨号系统中wdnet文件没有涉及，大致是改变lat中format类型限制变量，以及对lat结构体中其余成员进行赋值。
