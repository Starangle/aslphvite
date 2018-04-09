## 任务报告

目标文件内容格式如下：

```
~o
<VecSize> 39
<MFCC_0_D_A>
~v "varFloor1"
<Variance> 39
0.0012 0.0003 ...
<BeginHMM>
<NumStates> 5
<State> 2
<Mean> 39
0.0 0.0 0.0 ...
<Variance> 39
1.0 1.0 1.0 ...
<State> 3
<Mean> 39
0.0 0.0 0.0 ...
<Variance> 39
1.0 1.0 1.0 ...
<State> 4
<Mean> 39
0.0 0.0 0.0 ...
<Variance> 39
1.0 1.0 1.0 ...
<TransP> 5
0.0 1.0 0.0 0.0 0.0
0.0 0.6 0.4 0.0 0.0
0.0 0.0 0.6 0.4 0.0
0.0 0.0 0.0 0.7 0.3
0.0 0.0 0.0 0.0 0.0
<EndHMM>
```

1. ~o，~v标记舍弃，形如\<name>的标记也被舍弃，程序实际使用数据类型来对应这些标记

2. VecSize的值：`hset->vecSize`

3. MFCC_0_D_A：`hset->pkind`

4. ~v 的值：

   `hset->mtab[Hash[id->name]]->structure`，同时在`hset->pmap`中保存以下信息

   ```C++
   m=hset->mtab[Hash[id->name]]
   hashval=(unsigned long int)m->structure % PTRHASHSIZE
   hset->pmap[hashval]->ptr = m->structure;
   hset->pmap[hashval]->m = m;
   hset->pmap[hashval]->next = hset->pmap[hashval];
   ```

5. 一个完整的hmm定义，将会被存储以下信息：

   1. NumStates：`hset->mtab[Hash[id->name]]->numStates`，其中程序每次读到一个~h macro，就会产生一个id对象，然后通过该对象索引地址。

   2. State i：

      Mean存储于

      ```C++
      StateElem*(hset->mtab[Hash[id->name]]->structure)[i-2].info->pdf->
        spdf.cpdf->mpdf->mean
      ```

      Variance存储于

      ```C++
      StateElem*(hset->mtab[Hash[id->name]]->structure)[i-2].info->pdf->spdf.cpdf->mpdf->cov
      ```

   3. TransP：`hset->mtab[Hash[id->name]]->transP`

## 代码理解

1. HTK之所以要建立自己的堆栈类型，推测是要用来做垃圾回收。

## 附录A:查找过程

1. 从`main()`出发，检查参数被存储到全局变量`savedCommandLine`中。

2. 查看参数解析部分，发现MMF文件用HModel.c中的`AddMMF()`进行处理。`AddMMF()`传递了变量`hset`的指针，因此可以推断MMF文件将被加载到`hset`中。

3. `AddMMF()`首先进行了去重。对于还未被加载的MMF文件，做以下工作：

   1. 创建了一个`MMFInfo`用来记录文件本身的信息。

   2. `hset->numFiles`加1，显然`numFiles`指的是加载的MMF文件的个数。

   3. `MMFInfo`记录了MMF文件名称；hvite为文件的编号，编号等于当前文件数量；一个标识成员`isLoaded`，取值为`FALSE`用来表示文件内容还未被真正加载（推测加载后将改为`TRUE`)。

   4. 最后将新的`MMFInfo`追加到`hset`的成员`mmfNames`的尾部，根据下面的代码来看，`mmfNames`是一个单向链表。

      ```C
      if (hset->mmfNames == NULL)
        hset->mmfNames = p;
      else {  /* store in order of arrival */
        for (q=hset->mmfNames; q->next != NULL; q=q->next);
        q->next = p;
      }
      ```

   5. `AddMMF()`的返回值没有被接收，暂时不用注意其作用。

4. 因为上面没有真正加载MMF文件，下一步应当查找`isLoaded`为`TRUE`的位置，查找结果如下

   ```None
   查找全部 "isLoaded = TRUE", 查找结果 1, 整个解决方案, ""
     D:\Projects\htk\source code\HTKLib\HModel.c(6814):            mmf->isLoaded = TRUE;
     D:\Projects\htk\source code\HTKLib\HModel.c(8836):   p = AddMMF(hset,fname); p->isLoaded = TRUE;
     D:\Projects\htk\source code\HTKLib\HModel.c(8882):         p = AddMMF(hset,orphanMacFile); p->isLoaded = TRUE;
    匹配行: 3  匹配文件数: 1  已搜索文件总数: 514
   ```

5. 根据查找结果转到函数`LoadMacroFiles()`,该函数的解释为：

   > LoadMacroFiles: scan file list of hset and load any macro files

   因此推测就是我们要查看的函数。

6. 可以看到`LoadMacroFiles()`中遍历了mmfNames并调用函数`LoadAllMacros()`进行读取，之后将`isLoaded`设置为`TRUE`。

7. `LoadAllMacros()`主要代码解析

   1. `Source`结构体用来管理文件信息，主要是文件流和一些选项信息。
   2. 函数每次调用`GetToken()`方法来获得信息，`GetToken()`的详细解释查看**附录D**。
   3. 程序的基本流程都是按照**附录C**给出的范式逐个解析，因此直接在函数中找到`GetHMMDef(&dset, &src, &tok, dhmm, nState) < SUCCESS` ，该语句加载HMM部分。

8. 查看`GetHMMDef()`的内容，为了方便，将该函数的定义在此显示

   ```C++
   /* GetHMMDef: get a hmm def from given source */
   static ReturnStatus GetHMMDef(HMMSet *hset, Source *src, Token *tok,
                                 HLink hmm, int nState)
   {
      short state;
      int N=0;
      StateElem *se;
      char buf[MAXSTRLEN];

      if (trace&T_PAR) printf("HModel: GetHMMDef\n");
      if (tok->sym != BEGINHMM){
         HMError(src,"<BeginHMM> symbol expected");
         return(FAIL);
      }
      if(GetToken(src,tok)<SUCCESS){
         HMError(src,"GetHMMDef: GetToken failed");
         return(FAIL);
      }
      if (tok->sym == USEMAC){      /* V1 style USE clause */
         if (!ReadString(src,buf)){
            HRError(7013,"GetHMMDef: cannot read USE macro name");
            return(FAIL);
         }
         if(GetToken(src,tok)<SUCCESS){
            HMError(src,"GetHMMDEf: GetToken failed");
            return(FAIL);
         }
      }
      while (tok->sym != STATE) {
         if(GetOption(hset,src,tok, &N)<SUCCESS){
            HMError(src,"GetHMMDef: GetOption failed");
            return(FAIL);
         }
         if (N>nState) nState = N;
      }
      FreezeOptions(hset);
      if (nState==0){
         HMError(src,"NumStates not set");
         return(FAIL);
      }
      hmm->numStates = N = nState;
      se = (StateElem *)New(hset->hmem,(N-2)*sizeof(StateElem));
      hmm->svec = se - 2;
      while (tok->sym == STATE) {
         if (!ReadShort(src,&state,1,tok->binForm)){
            HMError(src,"States index expected");
            return(FAIL);
         }
         if (state<2 || state >= N){
            HMError(src,"State index out of range");
            return(FAIL);
         }
         if(GetToken(src,tok)<SUCCESS){
            HMError(src,"GetHMMDef: GetToken failed");
            return(FAIL);
         }
         se = hmm->svec+state;
         if((se->info = GetStateInfo(hset,src,tok))==NULL){
            HMError(src,"GetStateInfo failed");
            return(FAIL);
         }     
      }
      if (tok->sym==TRANSP || (tok->sym==MACRO && tok->macroType=='t')){
         if((hmm->transP = GetTransMat(hset,src,tok))==NULL){
            HMError(src,"GetTransMat failed");
            return(FAIL);
         }     
         if (NumRows(hmm->transP) != N ||
             NumCols(hmm->transP) != N){
            HRError(7030,"GetHMMDef: Trans Mat Dimensions not %d x %d",N,N);
            return(FAIL);
         }
      }
      else{
         HMError(src,"Transition Matrix Missing");
         return(FAIL);
      }
      if (tok->sym==DURATION || (tok->sym==MACRO && tok->macroType=='d')){
         if((hmm->dur = GetDuration(hset,src,tok))==NULL){
            HMError(src,"GetDuration Failed");
            return(FAIL);
         }
      }
      else
         hmm->dur = NULL;
      if (tok->sym!=ENDHMM){
         HMError(src,"<EndHMM> symbol expected");
         return(FAIL);
      }
      if(GetToken(src,tok)<SUCCESS){
         HMError(src,"GetHMMDef: GetToken failed");
         return(FAIL);
      }
      return(SUCCESS);
   }
   ```

   对一个典型的hmmdef文件，主要内容结构如下：

   ```
   <BeginHMM>
   <NumStates> 5
   <State> 2
   <Mean> 39
   0.0 0.0 0.0 ...
   <Variance> 39
   1.0 1.0 1.0 ...
   <State> 3
   <Mean> 39
   0.0 0.0 0.0 ...
   <Variance> 39
   1.0 1.0 1.0 ...
   <State> 4
   <Mean> 39
   0.0 0.0 0.0 ...
   <Variance> 39
   1.0 1.0 1.0 ...
   <TransP> 5
   0.0 1.0 0.0 0.0 0.0
   0.0 0.6 0.4 0.0 0.0
   0.0 0.0 0.6 0.4 0.0
   0.0 0.0 0.0 0.7 0.3
   0.0 0.0 0.0 0.0 0.0
   <EndHMM>
   ```

   11-14行读取标记"\<BEGINHMM>"；

   15-18行读取标记"\<NUMSTATES>"；

   对于标准的文件，USEMAC不会为真，我们假设[19-28]不会被执行；

   29-35行读取\<NUMSTATES>后的5，并更新N；

   41行我们看到N被写入`hmm->numStates`；

   44-89行类似地读取了文件中的信息，然后将信息存储到`hmm`的成员中；

9. 接下来我们需要知道HLink的结构以及刚才所用到的实例`hmm`如何从`hset`访问。看htk对HLink的定义：

   ```c++
   typedef struct {
      struct _HMMSet *owner;  /* owner of this model */
      short numStates;        /* includes entry and exit states */
      StateElem *svec;        /* array[2..numStates-1] of StateElem */  
      SVector dur;            /* vector of model duration params, if any */   
      SMatrix transP;         /* transition matrix (logs) */
      int tIdx;               /* Transition matrix index */
      int nUse;               /* num logical hmm's sharing this def */
      Ptr hook;               /* general hook */
   } HMMDef;

   typedef HMMDef * HLink;
   ```

   可以看到HLink是HMMDef的一个指针，根据刚才代码的行为我们可以确认以下信息：

   * HMMDef会保存一个指针owner，记录这个HMMDef属于哪个模型；
   * numStates保存hmm的状态个数（包含0状态和n状态）；
   * svec保存了指向一个向量的指针；
   * transP记录了状态转移矩阵；

   到这里我们仍然不清楚均值和方差是如何被存储的，因此还需要再进一步查看`GetStateInfo()`函数。我们反向查找**附录C**，得知通过L1可以查找到均值被加载到的地方，同理L2可以找到方差被加载到的地方。

   ```
   L1:GetStateInfo->GetStream->GetMixture->GetMixPDF->GetMean
   L2:GetStateInfo->GetStream->GetMixture->GetMixPDF->GetVariance
   ```

## 附录B:遇到的缩写含义

| 缩写   | 英文全称      | 中文全称 |
| ---- | --------- | ---- |
| Fn   | File Name | 文件名称 |

## 附录C:HMM定义语言

```ini
hmmdef = [~h macro] <BeginHMM> [globalOpts] <NumStates> short state {state} transP [duration] <EndHMM>
optmacro = ~o globalOpts
globalOpts = option {option}
option = <HmmSetId> string|<StreamInfo> short { short }|<VecSize> short|<ProjSize> short|<InputXform> inputXform|<ParentXform> ~a macro|covkind|durkind|parmkind
covkind = <DiagC>|<InvDiagC>|<FullC>|<LLTC>|<XformC>
durkind = <nullD>|<poissonD>|<gammaD>|<genD>
parmkind = <basekind{_D|_A|_T|_E|_N|_Z|_O|_V|_C|_K}>
basekind = <discrete>|<lpc>|<lpcepstra>|<mfcc>|<fbank>|<melspec>|<lprefc>|<lpdelcep>|<user>
state = <State:Exp> short stateinfo
stateinfo = ~s macro|[mixes][weights] stream { stream } [duration]|<TargetSource> layer <TargetIndex> short <TargetPenalty> float
macro = string
mixes = <NumMixes> short {short}
weights = ~w macro | <SWeights> short vector
vector = float {float}
stream = [<Stream> short](mixture {mixture}|tmixpdf|discpdf)
mixture = [<Mixture> short float] mixpdf
tmixpdf = <TMix> macro weightList
weightList = repShort { repShort }
repShort = short [* char]
discpdf = <DProb> weightList
mixpdf = ~m macro|mean cov [ <GConst> float]
mean = ~u macro|<Mean> short vector
cov = var|inv|xform
var = ~v macro|<Variance> short vector
inv = ~i macro|(<InvCovar>|<LLTCovar>) short tmatrix
xform = ~x macro|<Xform> short short matrix
matrix = float{float}
tmatrix = matrix
duration = ~d macro|<Duration> short vector
transP = ~t macro|<TransP> short matrix
baseClass = ~b macro baseopts classes
baseopts = <MMFIdMask> string <Parameters> baseKind <NumClasses> int
baseKind = MIXBASE|MEANBASE|COVBASE
classes = <Class> int itemlist{classes}
regTree = ~r macro <BaseClass> baseclasses node
baseclasses = ~b macro|baseopts classes
node = (<Node> int int int{int}|<TNode> int int int{int}){node}
adaptXForm = ~a macro adaptOpts <XformSet> xformset
adaptOpts = <AdaptKind> adaptkind <BaseClass> baseclasses [<ParentXForm> parentxform]
parentxform = ~a macro|adaptOpts <XformSet> xformset
adaptKind = TREE|BASE
xformset = <XFormKind> xformKind <NumXForms> int{linxform}
xformKind = MLLRMEAN|MLLRCOV|MLLRVAR|CMLLR|SEMIT
linxform = <LinXForm> int <VecSize> int [<OFFSET> xformbias] <BlockInfo> int int {int} block {block}
xformbias = ~y macro|<Bias> short vector
block = <Block> int xform
inputXform = ~j macro|inhead inmatrix
inhead = <MMFIdMask> string parmkind [<PreQual>]
inmatrix = <LinXform> <VecSize> int <BlockInfo> int int {int} block {block}
block = <Block> int xform
anndef = [~N macro] <BeginANN> <NumLayers> int layer{layer}<EndANN>
layer = <Layer:Exp> int layerinfo
layerinfo = ~L macro|<BeginLayer> <LayerKind> string <InputFeature> feamix [parameters] activation <EndLayer>
feamix = ~F macro|[ <NumFeatures> int int ] feature{feature}
feature = <Feature:Exp> int <Source> feasrc [<DimRange> int int] <ContextShift> int int {int}
feasrc = parmKind|stream|~L macro
parameters = <Weight> ~M macro <Bias> ~V macro
matrix = ~M macro|[<TargetMacro> ~M targetmacro] <Matrix> int int float{float}
vector = ~V macro|[ <TargetMacro> ~V targetmacro] <Vector> int float {float}
activation = <Activation> string [<NumParameters> int] <Parameter:Exp> int ~V macro{<Parameter:Exp> int ~V macro}
```

观察上述HMM定义语言，我们需要用到如下要点来理解代码：

1. 所有选项~c，当`c!='o'`时，其后必定跟随macro；

## 附录D:GetToken()做了什么？

| MMF文件内容                   | sym被设置为      | marcoType被设置为       |
| ------------------------- | ------------ | ------------------- |
| `~ c`                     | MACRO        | $c$                 |
| `#!MMF!#`                 | MACRO        | 'h'                 |
| `.`                       | MACRO        | 'h'                 |
| `<para>`                  | para对应的标识    | 未设置                 |
| `.[\s]+EOF`               | EOFSYM       | 未设置                 |
| `[BEGINHMM,PARMKIND)`的字符值 | 强制转换为Symbol类 | 未设置，但设置binForm为TRUE |
| 暂不清楚                      | PARMKIND     | 未设置                 |

## 附录E:一些简单的函数的功能记录

* TermScanner() 终止输入流。
* GetOption() 用来读取一些选项的值。
* RawReadFloat() 从文件src中连续读取n个float，放置于x中。
* ReadFloat()从文件src中连续读取n个float，放置于x中。该函数仅调用了RawReadFloat()。
* ReadVector() 从src中读取一个向量v。
