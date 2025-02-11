---
title: C#利用GDI+实现橡皮筋效果
author: pokey
pubDatetime: 2024-12-16T08:19:33.211Z
slug: csharp-gdi-learn
featured: true
draft: false
tags:
  - C#
  - GDI+
ogImage: ""
description: C#课一次作业需要在`winform`上实现一个简单的绘图程序，要求添加橡皮筋效果。图像是在`picturebox`控件上绘制的，我一开始始终解决不了的问题是要实现橡皮筋效果，鼠标移动过程中绘制显示的图形就要随时擦除，但是通过GDI+在控件的Graphic对象上绘制图形就不能再擦掉了。在网上搜索了一下，不管是刷新重绘控件，还是通过在内存中开辟位图的办法都失败，后面那种办法是最多的，但我怎么都弄不好，可能是我真地太菜了吧！
canonicalURL: https://example.org/my-article-was-already-posted-here
---
# C#利用GDI+实现橡皮筋效果

因为C#课一次作业需要在 `winform`上实现一个简单的绘图程序，要求添加橡皮筋效果。图像是在 `picturebox`控件上绘制的，我一开始始终解决不了的问题是要实现橡皮筋效果，鼠标移动过程中绘制显示的图形就要随时擦除，但是通过GDI+在控件的Graphic对象上绘制图形就不能再擦掉了。在网上搜索了一下，不管是刷新重绘控件，还是通过在内存中开辟位图的办法都失败，后面那种办法是最多的，但我怎么都弄不好，可能是我真地太菜了吧！

最终找到了办法是通过GDI+自带的双缓冲技术实现的。我们要实现橡皮筋的效果就要先擦除掉原来的图案，然后再绘制新的图案，我们可以先通过GDI+的填充背景色，覆盖原来的图形，再将所有图形绘制到屏幕上，让窗体刷新。但是窗体刷新时会频繁地重绘窗体表面，这样就会导致闪烁，而双缓冲能有效避免这个问题。自己也是在搜素的过程中大致理解了下这个技术。大概是先在内存中创建一个新的与窗体大小一样的缓冲区，我们的绘图操作先在这块缓冲区上完成，然后再将缓冲区上的图形渲染到屏幕上。由于渲染时只是进行位图的拷贝，所以速度是非常快的，能有效避免窗体反复刷新重绘时带来的闪烁。

大概思路是这样的：用 `GeometryBase`基类及其派生类储存图形对象，类中包括相应图形对象相应的绘制函数 `draw(Graphic g)`,用一个list集合 `GeometrySet`存放所有已经绘制完成的图形，定义一个接口 `Tool`，里面包括响应鼠标事件的函数 `onmousedown`,`onmousemove`等，再定义一个工具类及其派生类，继承自接口 `Tool`,实现接口，构造图形对象并添加到 `GeometrySet`集合中。

具体代码如下:

* 定义工具和控件的Graphic对象初始化

  ```C#
  // 绘制工具
        private Tool _actionTool = null;
        private Tool actionTool
        {
            get => _actionTool ?? new NoneTool(op);
            set => _actionTool = value;
        }
        private readonly Graphics pb;
        public Form1()
        {

            InitializeComponent();
            pb = pictureBox1.CreateGraphics();// 获取pictureBox1空间的绘制画布

        }
  ```
* 选择绘制的工具

  ```C#
        /// <summary>
        /// 选择绘图方式
        /// </summary>
        /// <param name="sender"></param>
        /// <param name="e"></param>
        private void comboBox2_SelectedIndexChanged(object sender, EventArgs e)
        {
            // 获取绘图方式
            Drawstyle = this.comboBox2.SelectedIndex;
            // 创建相应的图形绘制工具
            actionTool = CreateToolFactory.getDrawTool(Drawstyle, op);

        }
  ```
* 鼠标移动事件响应函数

  ```C#
  private void pictureBox1_MouseMove(object sender, MouseEventArgs e)
          {
              // 利用双缓冲实现橡皮筋效果和绘制图层容器图形
              BufferedGraphicsContext Mybuffer = BufferedGraphicsManager.Current;   // 创建缓冲图形上下文
              BufferedGraphics buffered = Mybuffer.Allocate(pb, pictureBox1.ClientRectangle); // 创建指定大小的缓冲区
              // 设置背景色为白色
              buffered.Graphics.FillRectangle(Brushes.White, pictureBox1.ClientRectangle); // 填充背景色，不填充的话屏幕最终显示的背景色是黑色
              // 绘制当前的图形
              actionTool.onmousemove(e, buffered.Graphics); // 绘制当前正在绘制中的图形
              // 绘制图层容器中的图形
              LayerService.DrawLayer(buffered.Graphics); // 绘制GeometrySet集合中已经存在的图形
              // 将图形渲染到屏幕上
              buffered.Render(pb); // 将所有图形渲染到屏幕上

              // 释放资源
              buffered.Dispose();
              Mybuffer.Dispose();

          }
  ```
* 鼠标点击事件响应函数

  ```C#
  private void pictureBox1_MouseDown(object sender, MouseEventArgs e)
          {
              // 使用绘制工具构造并添加图形对像
              actionTool.onmousedown(e);
          }
  ```
* 绘制工具类[^以绘制直线为例]

  ```C#
  class DrawLine : AbstractTool
      {
          private Point _startPoint;
          private Point _endPoint;
          private Polyline2D _line;
          public bool _dragging = false;
          public DrawLine(Options p) : base(p) {}
          public override void onmousedown(MouseEventArgs e)
          {
              // 构造图形对象
              base.onmousedown(e);
              if(e.Button == MouseButtons.Left)
              {
                  if (_dragging)
                  {
                      _endPoint = mouseDown;
                      _line.Add_Point(_endPoint);
                      LayerService.Add_Geometry(_line);
                      _dragging = false;

                  }
                  else
                  {
                      _startPoint = mouseDown;
                      _line = new Polyline2D(ops);
                      _line.Add_Point(_startPoint);
                      _dragging = true;
                  }
              }

          }
          // 橡皮筋效果实现
          public override void onmousemove(MouseEventArgs e,Graphics g)
          {
              base.onmousemove(e, g);
              if (_dragging)
              {
                  if (_line.getPtCount() < 2)
                      _line.Add_Point(mouseMove);
                  else
                      _line[1] = mouseMove;
                  _line.draw(g);
              }
          }
      }
  ```
