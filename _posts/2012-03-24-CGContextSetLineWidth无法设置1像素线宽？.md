---
layout: post
title:  CGContextSetLineWidth无法设置1像素线宽？
---

前段时间美术在验收界面时提了问题：为啥要求1像素宽的一个矩形框似乎却变成了2，3个像素宽。仔细检查过代码后发现，的确设置了LineWidth为1，但绘制效果却并不如人愿。似乎在ios上绘制最低要2个像素的线宽。

查看文档后发现造成这个问题的原因是Quartz的抗锯齿机制。一种粗暴的解决方案是不采用抗锯齿，即：CGContextSetShouldAntialias(context, NO)。但是显而易见的问题是取消抗锯齿会导致绘制效果变差。而另外一种方案则比较取巧：将绘制调整到半像素坐标系上：
比如 
> CGContextMoveToPoint(context, 100.0, 100.0); CGContextAddLineToPoint(context, 100.0, 200.0);

改为
> CGContextMoveToPoint(context, 100.5, 100.5); CGContextAddLineToPoint(context, 100.5, 200.5);


这是因为：所谓的线宽指的是给定路径的中心到两边的粗细，换句话是在路径的两边各绘制一半。如图 
![此处输入图片的描述][1]


在绘制线宽为1的直线(3,1)到(3,5)时，实际上是占据了左右两个像素各半个像素，而真正绘制时当然是以一个像素为标准单位，所以浅蓝色区域就会以相近的方式进行渲染。这也是宽为1.0的线绘制并不准确的原因。而当将绘制中心调整到半个像素上就不会有这个问题，见右图:(3.5,1)到(3.5,5)。详细可以参考[mozilla canvas绘制的文档][2]。


  [1]: /images/ios_anti.jpg
  [2]: https://developer.mozilla.org/En/Canvas_tutorial/Applying_styles_and_colors#A_lineWidth_example