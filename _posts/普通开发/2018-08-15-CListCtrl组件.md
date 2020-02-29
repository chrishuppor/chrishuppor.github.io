---
layout: post
title: "CListCtrl组件"
date: 2018-08-15 8:8:8
categories: Program
tags: MFC
---
MFC中CListCtrl组件常用的功能介绍及使用代码。*之所以会用到这个组件，主要是觉得windows的快速启动目录不好使，想自己做一个快速启动目录工具，于是就用这个组件打底做了一个*

[TOC]

# 初始化

1. 设置样式SetExtendedStyle
2. 添加列及标题InsertColumn(列号，标题)
3. 添加行InsertItem
4. 设置行列内容显示SetItemText(行号，列号，文字)

```c
//1. 设置扩展样式
m_FastDirList.SetExtendedStyle(LVS_EX_FULLROWSELECT | LVS_EX_GRIDLINES);//选中一行，显示网格

//2. 添加列及标题
m_FastDirList.InsertColumn(NAME_COL, TEXT("FILE NAME"));
m_FastDirList.InsertColumn(1, TEXT("FULL PATH"));
m_FastDirList.InsertColumn(2, TEXT("DIR STATUS"));
m_FastDirList.InsertColumn(3, TEXT("REMARK"));

//3.
m_FastDirList.InsertItem(m_iRow, szName);

//4.
m_FastDirList.SetItemText(m_iRow, PATH_COL, szPath);
m_FastDirList.SetItemText(m_iRow, STATUS_COL, (PathFileExists(szPath) ? TEXT("可用") : TEXT("不可用")));
m_FastDirList.SetItemText(m_iRow, REMARK_COL, szRemark);
```

# 修改列宽

SetColumnWidth(列号，列宽)

```c
#define TOTAL 15.0
/*****************************************
功能：根据比例调整CListCtrl空间列宽
*****************************************/
BOOL ChangeListTitleSize(CListCtrl *cListCtrl) {
    
    //1. 首先获取整个list的宽度
	CRect rect;
	GetClientRect(cListCtrl->m_hWnd, &rect);
    
    //2. 按照比例分配宽度
	cListCtrl->SetColumnWidth(NAME_COL, (int)(3.0 *rect.Width() / TOTAL));
	cListCtrl->SetColumnWidth(1, (int)(7.0*rect.Width() / TOTAL));
	cListCtrl->SetColumnWidth(2, (int)(2.0*rect.Width() / TOTAL));
	cListCtrl->SetColumnWidth(3, rect.Width() - (int)(12.0*rect.Width() / TOTAL));
	return TRUE;
}
```

# 获得点击位置的行列号

处理鼠标点击事件时：

1. 获取当前消息发生的位置，并转换为客户端位置
2. 构建LVHITTESTINFO
3. 获取点击信息SubItemHitTest

```c
//点击事件处理函数参数为(NMHDR *pNMHDR, LRESULT *pResult)
//1.
DWORD dwPos = GetMessagePos();
CPoint cPoint = {LOWORD(dwPos), HIWORD(dwPos)};
m_FastDirList.ScreenToClient(&cPoint);
//2.
LVHITTESTINFO lvinfo;
lvinfo.pt = cPoint;
lvinfo.flags = LVHT_ABOVE;
//3.
m_FastDirList.SubItemHitTest(&lvinfo);
int nItem = lvinfo.iItem; //行号
int nCol = lvinfo.iSubItem; //列号
```

# 点击列标题进行排序

专用的排序函数SortItems(回调函数，组件指针);

* 函数功能：根据回调函数中的规则对items进行排序
* 操作说明
  * 排序前需要使用SetItemData将每个item绑定一个标志（一般情况下，这个标志设定为行号），这个标志就是回调函数的前两个参数
  * 组件指针：参数指针

回调函数原型 int CALLBACK CompareItem(LPARAM lParam1, LPARAM lParam2, LPARAM lParamThisDialog)

* 函数功能：设定策略比较两个标志代表的item的大小，lParam1大则返回1，lParam2大则返回0
* 操作说明 
  * lParam1和lParam2是为每一行绑定的标志，eg：要比较第一行和第二行，绑定的标志分别为1和2，则传进的参数为 lParam1 =1, lParam2=2
  * lParamThisDialog是传进来的参数

（总之就是：排序由SortItems完成，排序过程中如何判断两个值之间的大小由回调函数决定。）

```c
/*****************************************
功能：排序回调函数，用于比较两个相邻项的大小;回调函数不能是类成员
*****************************************/
int CALLBACK Compare2Items(LPARAM lParam1, LPARAM lParam2, LPARAM lParamThisDialog){

	TCHAR szColData1[DEFAULT_SIZE] = {0};
	TCHAR szColData2[DEFAULT_SIZE] = { 0 };
    
    //1. 确定排序依据的列
	int iCol = ((CFastDirOpenDlg *)lParamThisDialog)->m_iCurrentCol;
    //2. 获得该列lParam1和lParam2对应的值
	((CFastDirOpenDlg *)lParamThisDialog)->m_FastDirList.GetItemText((int)lParam1, iCol, szColData1, DEFAULT_SIZE);
	((CFastDirOpenDlg *)lParamThisDialog)->m_FastDirList.GetItemText((int)lParam2, iCol, szColData2, DEFAULT_SIZE);
    //3. 返回比较结果
	return _tcscmp(szColData1, szColData2) > 0 ? 1 : 0;
}


/*****************************************
功能：点击某一列，按该列的值进行排序
*****************************************/
void CFastDirOpenDlg::OnLvnColumnclickListFastdir(NMHDR *pNMHDR, LRESULT *pResult)
{
	LPNMLISTVIEW pNMLV = reinterpret_cast<LPNMLISTVIEW>(pNMHDR);
	// TODO: 在此添加控件通知处理程序代码

	//1. 获得点击的列号
	m_iCurrentCol = pNMLV->iSubItem;//点击的列

	//2. 设置item绑定的data(其实不知道为什么这样做)
	for(int i = 0; i < m_iRow; i++){
		m_FastDirList.SetItemData(i, i);
	}

	//3. 根据回调函数的比较结果进行排序
	m_FastDirList.SortItems((PFNLVCOMPARE)Compare2Items, (DWORD_PTR)this);

	*pResult = 0;
}
```

# 最小化到系统托盘

这件事总共需要做以下几件事：

1. 将自身图标添加到系统托盘
2. 为了这个图标添加一个自定义消息，将该消息与图标绑定
3. 为这个自定义消息添加消息处理函数

具体代码示例如下：

```c
//1. 首先定义一个消息
#define WM_MY_SHOWTASK (WM_USER + 100) //[最小化]1.定义一个消息

//2. 定义该消息的处理函数
afx_msg LRESULT OnMyShowTask(WPARAM wParam, LPARAM lParam);//对函数的返回值和参数都有要求
//2.2 消息函数实现
LRESULT CFastDirOpenDlg::OnMyShowTask(WPARAM wParam, LPARAM lParam){
	
	if (wParam != IDR_MAINFRAME)
		return 1;
    
	switch (lParam){

	case WM_RBUTTONUP:{ //PS.加了大括号才能在这里定义变量      // 右键起来时弹出菜单 
		LPPOINT lpoint = new tagPOINT;
		::GetCursorPos(lpoint);                    // 得到鼠标位置，鼠标位置必须和点击消息一起，否则鼠标位置不准确
		CMenu menu;
		menu.CreatePopupMenu();                    // 声明一个弹出式菜单
		menu.AppendMenu(MF_STRING, WM_DESTROY, TEXT("关闭"));//右键菜单还可以这样
		menu.AppendMenu(MF_STRING, WM_CLOSE, TEXT("取消"));//右键菜单还可以这样
		menu.TrackPopupMenu(TPM_LEFTALIGN, lpoint->x, lpoint->y, this);
		HMENU hmenu = menu.Detach();
		menu.DestroyMenu();
		delete lpoint;
		break;
	}

	case WM_LBUTTONUP:  // 单击左键的处理
        iIsMainWindowShow = this->IsWindowVisible();
		if(FALSE == iIsMainWindowShow){//如果窗口没有显示就显示出来
			this->ShowWindow(SW_SHOWNORMAL);         // 显示主窗口
		}
		else{//如果窗口显示就隐藏
			this->ShowWindow(SW_HIDE);         // 隐藏主窗口
			//SendMessage(WM_SYSCOMMAND, SC_MINIMIZE, 0); //使用SendMessage发消息也可以
		}
		break;
	}
	return 0;
}

//3. 在MESSAGE_MAP中定义注册消息
ON_MESSAGE(WM_MY_SHOWTASK, OnMyShowTask)

//4. 在dialog类中添加一个系统托盘需要的变量
NOTIFYICONDATA m_nid;

//5. 调用Shell_NotifyIcon添加图标到系统托盘
//5.1 参数配置
m_nid.cbSize = (DWORD)sizeof(NOTIFYICONDATA);
m_nid.hWnd = this->m_hWnd;
m_nid.uID = IDR_MAINFRAME;
m_nid.uFlags = NIF_ICON | NIF_MESSAGE | NIF_TIP;//程序最小化到系统托盘时要有icon,可以相应消息，可以显示tips
m_nid.uCallbackMessage = WM_MY_SHOWTASK;             // 自定义的消息名称
m_nid.hIcon = LoadIcon(AfxGetInstanceHandle(), MAKEINTRESOURCE(IDR_MAINFRAME));
_tcscpy(m_nid.szTip, TEXT("FAST DIR OPEN"));                // 信息提示条为"服务器程序"，VS2008 UNICODE编码用wcscpy_s()函数
//5.2 调用函数
Shell_NotifyIcon(NIM_ADD, &m_nid);                // 在托盘区添加图标

//6. 重载窗口消息处理函数，增加：当窗口最小化将任务栏图标隐藏，如果关闭窗口则删除系统任务栏中图标
LRESULT CFastDirOpenDlg::WindowProc(UINT message, WPARAM wParam, LPARAM lParam)
{
	// TODO: Add your specialized code here and/or call the base class
	switch (message) //判断消息类型
	{
	case WM_SYSCOMMAND:
		//如果是系统消息 
		if (wParam == SC_MINIMIZE)//最小化窗口消息
		{
			//接收到最小化消息时主窗口隐藏 
			AfxGetApp()->m_pMainWnd->ShowWindow(SW_HIDE);
			return 0;
		}
		if (wParam == SC_CLOSE)
		{
			::Shell_NotifyIcon(NIM_DELETE, &m_nid); //关闭时删除系统托盘图标
		}
		break;
	}

	return CDialog::WindowProc(message, wParam, lParam);
}
```

# 两次单击进入编辑状态

这里使用了一个有意思的方式实现列表item文字编辑效果：借用CEdit

1. 创建一个CEdit
2. 这个CEdit平时处于隐藏且不可用状态
3. 触发编辑时将这个CEdit移动到item对应的位置并设置为与该item的一样大小，然后设置为可用状态
4. 触发编辑
   1. 两次单击触发：需要记录上一次点击的行号，如果本次点击的行号与之前点击的行号一致则触发编辑
   2. 过时消除：第一次点击后设置一个定时器，如果时间到了还没有进行下一次点击则消除上一次点击的记录

```c
//定时器定时响应：需要添加ON_WM_TIMER消息，函数原型void Func(UINT_PTR uIDEvent)
void CFastDirOpenDlg::OnTimer(UINT_PTR uIDEvent){
	KillTimer(REMARK_TIMER);//删除定时器
	if(FALSE == m_CEditRemark.IsWindowEnabled())//没有在编辑状态则过时消除（如果在编辑状态，还要使用m_iCurRemark，所以不能清除)
		m_iCurRemark = -1;
}


/****************************************************
功能：响应单击事件-具体是指响应remark修改的单击事件
****************************************************/
void CFastDirOpenDlg::OnNMClickListFastdir(NMHDR *pNMHDR, LRESULT *pResult)
{
	LPNMITEMACTIVATE pNMItemActivate = reinterpret_cast<LPNMITEMACTIVATE>(pNMHDR);
	*pResult = 0;

    //1. 获得点击的行列号
	DWORD dwPos = GetMessagePos();
	CPoint cPoint = {LOWORD(dwPos), HIWORD(dwPos)};

	m_FastDirList.ScreenToClient(&cPoint);

	LVHITTESTINFO lvinfo;
	lvinfo.pt = cPoint;
	lvinfo.flags = LVHT_ABOVE;

	int nItem = m_FastDirList.SubItemHitTest(&lvinfo);

	do{
		if(nItem == -1){//没有获得行号
			break;
		}

		if (lvinfo.iSubItem != REMARK_COL){//点击的不是备注列
			break;
		}
        
        //2. 根据点击的行号与上一次点击的行号比对结果进行不同的操作
        
        //2.1上一次点击的不是这一行，则是第一次点击这行——记录行号，设置定时器
		if (m_iCurRemark != nItem) {
			m_iCurRemark = nItem;
			SetTimer(REMARK_TIMER, RM_WAIT_TIME, NULL);
		}
		else {//2.2 上一次点击的就是这行
            
			//0. 已经再次点击进入了修改状态就不必再计时了
			KillTimer(REMARK_TIMER);

			//1.设定编辑框的位置
			CRect cEditRect;
			CRect cListRect;
			m_FastDirList.GetSubItemRect(nItem, REMARK_COL, LVIR_LABEL, cEditRect);//此处获得remark列的item的相对于m_FastDirList的rect，而movewindow使用的是相对于整个client的位置，所以还要进行变换
			m_FastDirList.GetWindowRect(&cListRect);
			ScreenToClient(&cListRect);
            
			cEditRect.left += cListRect.left;
			cEditRect.top += cListRect.top;
			cEditRect.right += cListRect.left;
			cEditRect.bottom += cListRect.top + 3;

			//2. 编辑框最初显示原来这一行这一列的文字
			TCHAR szOrgText[DEFAULT_SIZE] = {0};
			m_FastDirList.GetItemText(nItem, REMARK_COL, szOrgText, DEFAULT_SIZE);
			m_CEditRemark.SetWindowText(szOrgText);

			//3.移动编辑框
			m_CEditRemark.MoveWindow(cEditRect);
            
             //4. 进入编辑状态
			m_CEditRemark.EnableWindow(TRUE); //可用
			m_CEditRemark.ShowWindow(SW_SHOW); //显示出来
			m_CEditRemark.SetFocus();//设置输入焦点到m_CEditRemark
			m_CEditRemark.SetSel(-1);//设置输入光标

		}

	}while(FALSE);

	*pResult = 0;
}
```

PS.如果想退出编辑时能够获得编辑的内容，需要为m_CEditRemark添加失去焦点的事件处理函数

# 重载回车&ESC按键消息

重载消息函数：屏蔽esc和enter键，因为按esc默认调用OnCancel，enter默认调用OnOK(当然也可以重载这两个函数来屏蔽esc和enter)

```c
//如果不屏蔽这两个键，则按这两个键时会退出程序
BOOL CFastDirOpenDlg::PreTranslateMessage(MSG* pMsg)
{
	//if (pMsg->message == WM_KEYDOWN   &&   pMsg->wParam == VK_ESCAPE)     return   TRUE; //为了退出方便，我改主意了，不屏蔽ESC了
	if (pMsg->message == WM_KEYDOWN   &&   pMsg->wParam == VK_RETURN){
        return   TRUE;
	}
	else
		return   CDialog::PreTranslateMessage(pMsg);
}
```

# 拖拽文件到CListCtrl

1. 设置控件属性

   控件属性要 ACCEPT FILES TRUE,父控件也要设置。

2. 添加ON_WM_DROPFILES消息映射

代码示例如下：

```c
//1. 添加消息映射
BEGIN_MESSAGE_MAP(CMyListCtrl, CListCtrl)
	ON_WM_DROPFILES()
END_MESSAGE_MAP()

//2. 添加对应的消息处理程序
/************************************
功能：响应CMyListCtrl窗口的文件拖拽消息
参数：拖拽消息的信息
返回值：无
************************************/
void CMyListCtrl::OnDropFiles(HDROP hDropInfo) //函数原型
{
	if (hDropInfo)//拖拽消息指针
	{
		TCHAR szFilePath[DEFAULT_SIZE] = { 0 };
		int nDrag = DragQueryFile(hDropInfo, -1, NULL, 0);//获取一共拖拽的文件的个数

		for (int i = 0; i < nDrag; i++) {
			RtlZeroMemory(szFilePath, DEFAULT_SIZE);
			DragQueryFile(hDropInfo, i, szFilePath, 1024);//获得第一个文件的路径m_szFilePath，返回路径长度
			//todo
		}
	}

	DragFinish(hDropInfo);
}
```

#  删除选中的item

删除单行只需要通过GetSelectionMark获取item的index，然后DeleteItem(iIndex)就可以了。

删除多行，尤其是不连续的多行，要使用到GetFirstSelectedItemPosition和GetNextSelectedItem，代码如下：

```c
//需要要将控件Single Select属性设置为FALSE
void CMyListCtrl::DelSelectedItem() {
	POSITION sSelPos = NULL;
	int iCount = this->GetItemCount();

	while (sSelPos = this->GetFirstSelectedItemPosition())
	{
		int iIndex = -1;
		iIndex = this->GetNextSelectedItem(sSelPos);

		if (iIndex >= 0 && iIndex < iCount){
			this->DeleteItem(iIndex);
		}
	}
}
```

[示例项目：FastDirOpen](https://github.com/chrishuppor/src/tree/master/MyLittleTools/FastDirOpen)