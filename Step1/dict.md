### 1. 参数说明
在`dict`加载当中用到了两个比较关键的数据结构`Vocab`和`Word`，其中`Vocab`作为全局变量，记载着`dict`信息，而`Word`作为`dict`加载过程中的一个中间变量，只记录一个word及其对应的phones

### 2.函数调用
* `InitVocab(&vocab):`  初始化处理全局变量vocab
* `ReadDict(dictFn,&vocab):`   加载dict文件（具体实现dict加载的函数）
    * `ReadDictWord(&src,labels,&prob, &nphones):`    读取dict中的一个word及其对应的phones，存储到lables中
    * `word = GetWord(voc,labels[0],TRUE):`      根据给定的wordName(labels[0]),通过VocabHash(wordName)查重之后，再进行存储
    * `NewPron(voc,word,nphones,labels+2,labels[1],prob):`     将lables当中存储的word的phones存储为对应word的pron。

### 3.主要代码
```
/* EXPORT->ReadDict: read and store a dictionary definition */
ReturnStatus ReadDict(char *dictFn, Vocab *voc)
{
   LabId labels[MAXPHONES+4];
   Source src;
   Word word;
   float prob;
   int nphones;
   ReturnStatus ret;

   if((ret=ReadDictWord(&src,labels,&prob, &nphones))<SUCCESS){
      CloseSource(&src);
      HRError(8013,"ReadDict: Dict format error in first entry");
      return(FAIL);
   }
   while(nphones>=0){
      word = GetWord(voc,labels[0],TRUE);
      if (labels[1]==NULL) labels[1]=labels[0];
      if (labels[1]->name[0]==0) labels[1]=NULL;
      if (voc->nullWord->wordName == word->wordName)
         HRError(-8013,"ReadDict: !NULL entry contains pronunciation");
      NewPron(voc,word,nphones,labels+2,labels[1],prob);
      if((ret=ReadDictWord(&src,labels,&prob, &nphones))<SUCCESS){
         HRError(8013,"ReadDict: Dict format error");
         return(FAIL);
      }
   }

   return(SUCCESS);
}
```


