## 任务报告



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

7. `LoadMacroFiles()`主要代码解析

   1. `Source`结构体用来管理文件信息，主要是文件流和一些选项信息。
   2. ​

## 附录B:遇到的缩写含义

| 缩写   | 英文全称      | 中文全称 |
| ---- | --------- | ---- |
| Fn   | File Name | 文件名称 |
|      |           |      |