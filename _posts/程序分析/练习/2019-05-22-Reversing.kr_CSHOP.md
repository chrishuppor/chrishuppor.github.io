---
layout: post
title: "Reversing.kr_CSHOP"
pubtime: 2019-05-22
updatetime: 2019-05-22
categories: Reverse
tags: WriteUp
---

Reversing.kr的第十四题，是一个简单但巧妙的题，利用了“白纸白字”的思想。

# Reversing.kr_CSHOP

## 解题过程

1. 解压程序后，没有readme。直接运行程序，发现除了一个框什么也没有。联想到之前的ImageProc，试着在框中画几笔，没有成功，果然不是一个思路。*（PS.目前还没有发现这个网站有思路重复的题目）*

2. 拖入PEiD，发现是C#程序，没有壳。

3. 拖入dnSpy，发现这个程序进行了混淆。关闭dnSpy，使用de4dot进行去混淆。

4. 将去混淆的程序拖入dnSpy。程序十分简单，一共也才不到二百行代码。

   1. 有一个派生子Form的FrmMain类，该类有四个方法

      1. InitializeCompenont：初始化函数，生成一个按钮和十个静态Label，使用Form1_Load将按钮和label的文字全部初始化为""，按钮事件由btnStart_Click处理。

         其中，为每个控件指派了TabIndex，说明可以使用Tab键操作控件。

         ```c#
         private void InitializeComponent()
         {
         	ComponentResourceManager componentResourceManager = new ComponentResourceManager(typeof(FrmMain));
         	this.btnStart = new Button();
         	this.lblGu = new Label();
         	this.lblNu = new Label();
         	this.lblSu = new Label();
         	this.lblTu = new Label();
         	this.lblKu = new Label();
         	this.ppppp = new Label();
         	this.lblMu = new Label();
         	this.lblXu = new Label();
         	this.lblZu = new Label();
         	this.lblQu = new Label();
         	base.SuspendLayout();
         	this.btnStart.Location = new Point(165, 62);
         	this.btnStart.Name = "btnStart";
         	this.btnStart.Size = new Size(0, 0);
         	this.btnStart.TabIndex = 0; //
         	this.btnStart.UseVisualStyleBackColor = true;
         	this.btnStart.Click += this.btnStart_Click; //按钮事件
         	this.lblGu.Location = new Point(43, 123);
         	this.lblGu.Name = "lblGu";
         	this.lblGu.Size = new Size(53, 23);
         	this.lblGu.TabIndex = 1;
         	this.lblGu.Text = "label1";
         	this.lblNu.Location = new Point(90, 123);
         	this.lblNu.Name = "lblNu";
         	this.lblNu.Size = new Size(53, 23);
         	this.lblNu.TabIndex = 2;
         	this.lblNu.Text = "label2";
         	this.lblSu.Location = new Point(135, 123);
         	this.lblSu.Name = "lblSu";
         	this.lblSu.Size = new Size(53, 23);
         	this.lblSu.TabIndex = 3;
         	this.lblSu.Text = "label3";
         	this.lblTu.Location = new Point(182, 123);
         	this.lblTu.Name = "lblTu";
         	this.lblTu.Size = new Size(53, 23);
         	this.lblTu.TabIndex = 4;
         	this.lblTu.Text = "label4";
         	this.lblKu.Location = new Point(228, 123);
         	this.lblKu.Name = "lblKu";
         	this.lblKu.Size = new Size(53, 23);
         	this.lblKu.TabIndex = 5;
         	this.lblKu.Text = "label4";
         	this.ppppp.Location = new Point(278, 123);
         	this.ppppp.Name = "ppppp";
         	this.ppppp.Size = new Size(53, 23);
         	this.ppppp.TabIndex = 6;
         	this.ppppp.Text = "label4";
         	this.lblMu.Location = new Point(324, 123);
         	this.lblMu.Name = "lblMu";
         	this.lblMu.Size = new Size(53, 23);
         	this.lblMu.TabIndex = 7;
         	this.lblMu.Text = "label4";
         	this.lblXu.Location = new Point(369, 123);
         	this.lblXu.Name = "lblXu";
         	this.lblXu.Size = new Size(53, 23);
         	this.lblXu.TabIndex = 8;
         	this.lblXu.Text = "label4";
         	this.lblZu.Location = new Point(413, 123);
         	this.lblZu.Name = "lblZu";
         	this.lblZu.Size = new Size(53, 23);
         	this.lblZu.TabIndex = 9;
         	this.lblZu.Text = "label4";
         	this.lblQu.Location = new Point(457, 123);
         	this.lblQu.Name = "lblQu";
         	this.lblQu.Size = new Size(53, 23);
         	this.lblQu.TabIndex = 10;
         	this.lblQu.Text = "label4";
         	base.AutoScaleDimensions = new SizeF(7f, 12f);
         	base.AutoScaleMode = AutoScaleMode.Font;
         	base.ClientSize = new Size(626, 316);
         	base.Controls.Add(this.lblQu);
         	base.Controls.Add(this.lblZu);
         	base.Controls.Add(this.lblXu);
         	base.Controls.Add(this.lblMu);
         	base.Controls.Add(this.ppppp);
         	base.Controls.Add(this.lblKu);
         	base.Controls.Add(this.lblTu);
         	base.Controls.Add(this.lblSu);
         	base.Controls.Add(this.lblNu);
         	base.Controls.Add(this.lblGu);
         	base.Controls.Add(this.btnStart);
         	base.FormBorderStyle = FormBorderStyle.FixedSingle;
         	base.Icon = (Icon)componentResourceManager.GetObject("$this.Icon");
         	base.MaximizeBox = false;
         	base.Name = "FrmMain";
         	base.StartPosition = FormStartPosition.CenterScreen;
         	this.Text = "CSHOP";
         	base.Load += this.Form1_Load;
         	base.ResumeLayout(false);
         }
         ```

      2. Form1_Load：控件文字初始化

      3. btnStart_Click：当点击按钮时对Label的文字进行修改

         这串字符疑似Flag，而且也没有别的字符了，所以将其输入Auth，居然是Wrong。看来这几个字符有顺序问题。

         ```c#
         private void btnStart_Click(object sender, EventArgs e)
         {
         	this.lblSu.Text = "W";
         	this.lblGu.Text = "5";
         	this.lblNu.Text = "4";
         	this.lblKu.Text = "R";
         	this.lblZu.Text = "E";
         	this.lblMu.Text = "6";
         	this.lblTu.Text = "M";
         	this.ppppp.Text = "I";
         	this.lblGu.Text = "P";
         	this.lblQu.Text = "S";
         	this.ppppp.Text = "P";
         	this.lblTu.Text = "6";
         	this.lblXu.Text = "S";
         }
         ```

      4. 查看InitializeComponent中各标签的位置，发现都在同一行，从左到右分别是lblGu、lblNu、lblSu、lblTu、lblKu、ppppp、lblMu、lblXu、lblZu、lblQu，所以btnStart_Click中字符的顺序从左到右为P4W6RP6SES。需要注意的是，在btnStart_Click中出现的一个label有多次赋值的，要以最后一次赋值为准。

         PS.还有其他方法获得flag：

         * 因为按钮可以通过Tab来设置焦点，所以可以按Tab，然后按enter，就算是点击了按钮，即触发btnStart_Click，显示出flag

         * 使用spylite将按钮放到到铺满整个exe窗口，点击之后再缩小，flag就可以显示出来了。

           1. 打开cshop.exe，使用spylite获取其窗口。

              ![图1 spylite获取cshop.exe窗口](https://chrishuppor.github.io/image/Snipaste_2019-05-23_08-44-23.PNG)

           2. 在“窗口”中查看“子窗口”，很容易就能找到Button对应的子窗口。

              ![图2 子窗口列表](https://chrishuppor.github.io/image/Snipaste_2019-05-23_08-46-21.PNG)

           3. 选择button子窗口，进入“消息”选项，在窗口状态中选择”最大化“，此时会将button铺满整个客户区域。

              ![图3 窗口状态](https://chrishuppor.github.io/image/Snipaste_2019-05-23_08-47-36.PNG)

           4. 在cshop.exe区域中点击，然后将最大化去掉就可以看到flag了。

   ## 小结

   * 一个很简单的c#逆向题，dnSpy可以将其源码清晰的展现出来
     * dnSpy反编译的强大使我误以为c#程序的逻辑无处可藏，但通过咨询大佬得知没这么简单——c#程序可以附带自解压功能，将核心逻辑隐藏在资源节，在程序运行时自行解压该段代码并运行。然而dnSpy无法识别资源节的代码，此时需要用WinDbg进行动态调试。
     * OD无法调试c#程序，因为c#属于托管代码，由公共语言运行库环境运行，OD只能调试c等由os直接运行的非托管代码
   * 使用了“白纸白字”的方法隐藏了所有的label，使用0size的方法隐藏了按钮。
   * 其实理论上这个题也可以通过修改代码中btn的属性来使btn可见，从而点击按钮，但是我并没有实践成功。