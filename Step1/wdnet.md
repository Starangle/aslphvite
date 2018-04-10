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
 除W外其余数字的赋值即为编号<br>

#### 读文件流程
　主要对应代码如下:<br>
 
 

	  
``` C++      
do {
     if ((ptr=GetNextFieldName(nbuf,&del,src)) == NULL) 
         break;
			/* Recognised line types have only one character names */
      if (strlen(ptr)==1) 
         ntype=*ptr;
      else 
         ntype=0;
      if (ntype == '.') {
         ptr = NULL;
         break;
      }
      switch(ntype) {
      case '\n': break;
      case 'I':
         n=GetIntField('I',del,vbuf,src);
         if (n < 0 || n >= lat->nn){
            Dispose(heap, lat);
            HRError(8251,"ReadLattice: Lattice does not contain node %d",n);
            return(NULL);
         }
         ln=lat->lnodes+n;
         if (ln->hook!=NULL){
            Dispose(heap, lat);
            HRError(8251,"ReadLattice: Duplicate info info for node %d",n);
            return(NULL);
         }
         time=0.0;wordId=voc->nullWord;tag=NULL;v=-1;
         while((ptr=GetNextFieldName(nbuf,&del,src)) != NULL) {
            if (nbuf[0]=='\n') break;
            else {
               if (strlen(ptr)>=1) 
                  ntype=*ptr;
               else 
                  ntype=0;
               switch(ntype) {
               case 't':
                  time=GetFltField('t',del,vbuf,src);
                  time *= tscale;
                  lat->format |= HLAT_TIMES;
                  break;
               case 'W':
                  GetFieldValue(vbuf,src,0);
                  wordId=GetWord(voc,GetLabId(vbuf,add2Dict),add2Dict);
                  if (wordId==NULL){
                     Dispose(heap, lat);
                     HRError(8251,"ReadLattice: Word %s not in dict",vbuf);
                     return(NULL);
                  }
                  break;
               case 's':
                  GetFieldValue(vbuf,src,0);
                  tag=CopyString(heap,vbuf);
                  lat->format |= HLAT_TAGS;
                  break;
               case 'L':
                  GetFieldValue(vbuf,src,0);
                  wordId=voc->subLatWord;
                  if((subLatId=AdjSubList(lat,GetLabId(vbuf,TRUE),NULL,+1))==NULL) {
                     HRError(8251,"ReadLattice: AdjSubLat failed");
                     return(NULL);
                  }

                  break;
               case 'v':
                  lat->format |= HLAT_PRON;
                  v=GetIntField('v',del,vbuf,src);
                  break;
               default:
                  GetFieldValue(0,src,0);
                  break;
               }
            }
         }
         if (wordId != voc->nullWord){
            lat->format &= ~HLAT_ALABS;}
         ln->time=time;/*time=0.0;tag=NULL;v=-1;*/
         ln->word=wordId;
         ln->tag=tag;
         ln->v=v;
         if (wordId == voc->subLatWord)
            ln->sublat = subLatId;
         else
            ln->sublat = NULL;
         ln->hook=ln;
         nn--;
         break;
      case 'J':
         n=GetIntField('I',del,vbuf,src);
         if (n<0 || n>=lat->na){
            Dispose(heap, lat);
            HRError(8251,"ReadLattice: Lattice does not contain arc %d",n);
            return(NULL);
         }
         la=NumbLArc(lat,n);
         if (la->start!=NULL){
            Dispose(heap, lat);
            HRError(8251,"ReadLattice: Duplicate info for arc %d",n);
            return(NULL);
         }
         s=e=v=-1; wordId=NULL; aclike=lmlike=0.0;
         prlike=0.0;
         while ((ptr=GetNextFieldName(nbuf,&del,src))) {
            if (nbuf[0]=='\n') break;
            else {
               if (strlen(ptr)>=1) ntype=*ptr;
               else ntype=0;
               switch(ntype)
                  {
                  case 'S':
                     s=GetIntField('S',del,vbuf,src);
                     if (s<0 || s>=lat->nn){
                        Dispose(heap, lat);
                        HRError(8251,"ReadLattice: Lattice does not contain start node %d",s);
                        return(NULL);
                     }
                     break;
                  case 'E':
                     e=GetIntField('E',del,vbuf,src);
                     if (e<0 || e>=lat->nn){
                        Dispose(heap, lat);
                        HRError(8251,"ReadLattice: Lattice does not contain end node %d",e);
                        return(NULL);
                     }
                     break;
                  case 'W':
                     GetFieldValue(vbuf,src,0);
                     wordId=GetWord(voc,GetLabId(vbuf,add2Dict),add2Dict);
                     if (wordId==NULL || wordId==voc->subLatWord){
                        Dispose(heap, lat);
                        HRError(8251,"ReadLattice: Word %s not in dict",
                                vbuf);
                        return(NULL);
                     }
                     break;
                  case 'v':
                     lat->format |= HLAT_PRON;
                     v=GetIntField('v',del,vbuf,src);
                     break;
                  case 'a':
                     lat->format |= HLAT_ACLIKE;
                     aclike=GetFltField('a',del,vbuf,src);
                     aclike = ConvLogLikeFromBase(logbase, aclike);
                     break;
                  case 'l':
                     lat->format |= HLAT_LMLIKE;
                     lmlike=GetFltField('l',del,vbuf,src);
                     lmlike = ConvLogLikeFromBase(logbase, lmlike);
                     break;
                  case 'r':
                     lat->format |= HLAT_PRLIKE;
                     prlike=GetFltField('r',del,vbuf,src);
                     prlike = ConvLogLikeFromBase(logbase, prlike);
                     break;
                  case 'd':
                     lat->format |= HLAT_ALIGN;
                     GetFieldValue(dbuf,src,DBUFLEN);
                     if (!shortArc)
                        la->nAlign=ReadAlign(lat,la,dbuf);
                     break;
                  default:
                     GetFieldValue(0,src,0);
                     break;
                  }
            }
         }
         if (s<0 || e<0 ||(wordId==NULL && (lat->format&HLAT_ALABS))){
            Dispose(heap, lat);
            HRError(8250,"ReadLattice: Need to know S,E [and W] for arc %d",n);
            return(NULL);
         }
         la->start=lat->lnodes+s;
         la->end=lat->lnodes+e;
         la->lmlike=lmlike;
           
         if ((lat->format&HLAT_ALABS) && la->end->word == voc->nullWord){
            la->end->word=wordId;
	    la->end->v = v;
	 }
         if (wordId != NULL && la->end->word != wordId){
            Dispose(heap, lat);
            HRError(8251,"ReadLattice: Lattice arc (%d) W field (%s) different from node (%s)",  n,wordId->wordName->name,la->end->word->wordName->name);
            return(NULL);
         }

         la->farc=la->start->foll;
         la->parc=la->end->pred;
         la->start->foll=la;
         la->end->pred=la;
         if (!shortArc) {
            la->aclike=aclike;
            la->prlike=prlike;
         }
         na--;
         break;
      default:
         GetFieldValue(0,src,0);
         while ((ptr=GetNextFieldName(nbuf,&del,src))) {
            if (nbuf[0]=='\n') break;
            else GetFieldValue(0,src,0);
         }
         break;
    }
    }
    while(ptr != NULL);
 ```

##### 主要调用函数
    `newlattice`函数初始化lattice<br>
  　读wdnet文件在`readlattice()`以及`readonelattice()`函数中<br>
    getnextfieldname函数读取name，getintfield函数读取数值<br>
    
##### 一些关键变量   
  　nn：局部变量，lnode的数目,即节点数目<br>
  　na：局部变量，larc的数目，即弧的数目<br>
    la、ln：局部变量，链接到lat->lnode节点
  　除上述两个变量外`lat`结构体还填充一系列值，拨号系统中为默认，如utterance等<br>
##### 大致流程  　
  　先读N和L的值，N的数值存在nn，L数值存在na，即lat->nn,na;<br>
 　 之后判断lat->foemat是否为shortarc，短弧，据情况分配空间并初始化<br>
 　 向下依次读I值，读到n中，定位到lat->lnode+n,即ln，紧接着读W值，读信息到wordID，即在voc->wtab中的位置。<br>
 　 wtab储存word结构体数组，地址通过vocabhash定位，其余字符主要修改format信息，没有用到<br>
　  读完I W行，即所有node信息之后，读J S E值到n,s,e;<br>
　  写入LARC结构体la，s对应start，e对应end，la->start和end对应lat->lnodes相应位置，位置为线性对应<br>
 　 l r对应lmlike和prlike，暂时没有值，其余字母基本是对应format信息，拨号系统中没有<br>

#### 文件读写相关结构体
  　 N，函数readonelattice中nn<br>
 　  L，函数readonelattice中na<br>
   　I，对应lat->lnode+n,链接到ln，lat为lattice结构，函数返回本结构体，ln、la、nn、na等为局部变量<br>
  　 W，wordID，对应voc->wtab[]Word数组中的值，voc储存全局词典信息<br>
   　J，即弧编号，S，E为弧起点和终点，储存到la->start和la->end，并链接到lat->lnodes对应位置节点<br>
   　其余信息这个拨号系统中wdnet文件没有涉及，大致是改变lat中format类型限制变量，以及对lat结构体中其余成员进行赋值。<br>
