使用TVINSERTSTRUCT结构插入节点：

示例：插入根节点

如果要根节点显示出加减号，需要设置HasButton和LinesAtRoot为TRUE，然后cChildren = 1。

```c
TVINSERTSTRUCT tvInsert;
tvInsert.hParent = NULL; //1
tvInsert.hInsertAfter = NULL; //2
tvInsert.item.mask = TVIF_TEXT | TVIF_CHILDREN; //3 为了显示出加号
tvInsert.item.pszText = szTmp;
tvInsert.item.cChildren = 1; //4 I_CHILDRENCALLBACK据说用户没有扩展过节点的时候，设置cChildren为1，用户有扩展节点的时候，用实际子节点数代替——但我试着没用

HTREEITEM hItem = this->InsertItem(&tvInsert);
```

