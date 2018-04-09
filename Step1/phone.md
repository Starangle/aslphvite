## 任务报告

### 1、 输入: `tiedtri.pho`  
里面是一些三音素的描述。有`logic hmm`和`physical hmm`之分，每行的第一个是`logic hmm`第二个是`physical hmm`。 如果某一行只有一个，则`logic hmm`和`physical hmm`都是它。

###### 逻辑和物理的解释:  
三音素状态种类非常多，实际建模的时候这样会太大，于是用决策树把一些状态进行绑定，把相同类似的三音素绑定在一起，进行一些抽象，于是就会出现一对多的情况,一个实体对应多个逻辑。

### 2. 调用函数：`InitHMMSet`  (HModel.c)  
主要函数调用关系: `InitHMMSet` -> `CreateHMM` -> `NewMacro`

#### `InitHMMSet`:

* 主要用到的结构体：`Source`控制文件读取，对回车换行空白等加了一些特定的处理。控制`source`的主要有两个函数:`GetCh` 和 `UnGetCh`。

	*  第一个GetCh是从source中读取一个信息，source中会有一个变量putback作为一个临时缓冲区，如果这个变量有内容，则GetCh会返回putback的内容，如果这个变量没有内容，会从文件中读取一个作为函数的返回。

	* 第二个UnGetCh，是把读取的内容进行回退。这么做是因为需要把字符一个一个读出来，然后对每个字符进行判断，判断是否为空，换行等符号。如果是空或者换行就不用把读取到的内容放到buffer里，然后对buffer里的内容进行处理，并且不会影响后面没有读取的部分，也方便一次跳过多个空格。同样，回退的内容会放在putback中，以便下一次的读取，如果putback有内容，则回退给文件中。
	* 实际上并不是判断putback是否为空，而是有另外一个bool变量作为一个开关。这里不是关键，这么说明只是为了方便理解，理解即可。


* 执行流程:

	从`tiedtri.pho`中一次读取一个三音素字串，遇到空格或者换行停止。每行第一个字串当做`logical hmm`，如果这一行没有第二个字串,则这个字串既是`logical hmm`又是`physics hmm`。如果有第二个字串，则第二个字串作为`physics hmm`。每读一行，就会对这一行的内容创建一个`macro`模型，在`CreateHMM`中进行处理，传入的就是`logical hmm`和`physics hmm`以及`hmmset`。

	三音素字串存储在一个`NameCell`的结构体中,里面存放的有三音素内容和一个指针，这个指针是`hash`链表节点的指针。以及一个`void* aux`(暂时没看到用法)。这个结构体组成的一个`hash`数组叫`hashTab`,是一个链式`hash`表。上面所说的`hash`表就是指的`hashTab`。
#### `CreateHMM`:

* 执行流程:
	1. `logicalhmm`不能重复使用。一上来进行`logicalhmm`的`macro`查重。如果存在则报错。
	2. 然后创建`physics hmm`对应的`macro`。如果`physics hmm`对应的`macro`不存在，则创建一个。如果存在,则把`macro`所对应的`hmm->nUse`加一。
	3. 最后创建一个新的`logical hmm`对应的`macro`。
	4. 创建`macro`的过程在`NewMacro`中实现。

### NewMacro:
* 主要用到的结构体：
`MacroDef`，包含了`Macro`对应的信息，包括三音素和对应的hmm。刚创建的时候，`MacroDef->hmm`设定为初始值，然后给`macro`赋上相应的三音素值。把`marco`放到`hset`的`mtab`哈希表中，也是一个链式hash。
* 执行流程：

	1. 如果是`physcis hmm`则对`hset->numPhyhmm`进行加一，如果是`logicalhmm`则对`hset->numLogHMM`进行加一。
	2. 创建`macro`结构体,并进行相应的初始化,然后用hash函数进行映射,把三音素名称映射成为一个数值,作为`mtab`的索引,并把`macro`放进到hash表中,返回`macro`。
