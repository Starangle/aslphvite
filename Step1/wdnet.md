#### 文件格式
	VERSION=1.0
	N=1106 L=2091
	I=0    W=!NULL             
	I=1    W=!NULL             
	I=2    W=SENT-START        
	I=3    W=\243\263\243\304\306\265\265\300
	...
	J=0     S=1105 E=1    
	J=1     S=0    E=2    
	J=2     S=2    E=3   
	J=3     S=3    E=4    
	J=4     S=5    E=4    
	J=5     S=12   E=4   
	J=6     S=24   E=4   
	J=7     S=25   E=4    
	J=8     S=26   E=4   
	J=9     S=27   E=4   
	J=10    S=28   E=4    
	J=11    S=29   E=4    
	J=12    S=30   E=4   
	J=13    S=31   E=4    
	J=14    S=32   E=4   
	J=15    S=41   E=4    
	...
#### 文件中符号含义

* VERSION:版本号<br>
* I:节点标号<br>
* W：节点名字<br>
* J：弧标号<br>
* S：弧开始节点的标号<br>
* E：弧结束节点的标号<br>
* 除W外其余数字的赋值即为编号<br>

#### 读文件流程
newlattice初始化lattice<br>
读文件在readlattice以及readonelattice函数中<br>
nn：lnode的数目,即节点数目<br>
na：larc的数目，即弧的数目<br>
并将填充一系列值，拨号系统中为默认，如utterance等<br>
读wdnet文件在readonelattice函数<br>
getnextfieldname函数读取name，getintfield函数读取数值<br>
N的数值存在nn，L数值存在na，即lat->nn,na;<br>
之后判断lat->foemat是否为shortarc，短弧，据情况分配空间并初始化<br>
向下依次读I值，读到n中，定位到lat->lnode+n,即ln，紧接着读W值，读信息到wordID，即在voc->wtab中的位置。<br>
wtab储存word结构体数组，地址通过vocabhash定位，其余字符主要修改format信息，没有用到<br>
读完I W行，即所有node信息之后，读J S E值到n,s,e;<br>
写入LARC结构体la，s对应start，e对应end，la->start和end对应lat->lnodes相应位置，线性<br>
l r对应lmlike和prlike，暂时没有值，其余字母基本是对应format信息，拨号系统中没有<br>

#### 文件读写相关结构体
   N，函数readonelattice中nn<br>
   L，函数readonelattice中na<br>
   I，对应lat->lnode+n,链接到ln，lat为lattice结构，函数返回本结构体，ln、la、nn、na等为局部变量<br>
   W，wordID，对应voc->wtab[]Word数组中的值，voc储存全局词典信息<br>
   J，即弧编号，S，E为弧起点和终点，储存到la->start和la->end，并链接到lat->lnodes对应位置节点<br>
   其余信息这个拨号系统中wdnet文件没有涉及，大致是改变lat中format类型限制变量，以及对lat结构体中其余成员进行赋值。<br>
