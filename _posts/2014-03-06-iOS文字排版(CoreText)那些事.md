---
layout: post
title:  iOS 文字排版 (CoreText) 那些事
---


第一次比较深入接触 iOS 文字排版相关内容是在 12 年底，实现某 IM 项目聊天内容的图文混排，照着 nimbus 的 AttributedLabel 和 Raywenderlish 上的这篇文章 [《Core Text Tutorial for iOS: Making a Magazine App》][1] 改出了一个比较适用于聊天内容展现的图文混排 (文字和表情) 控件。选择自己写而不直接使用现有第三方库的原因有三:

* 在这之前也做过一个 iOS 上的 IM 产品，当时这个模块并不是我负责，图文混排的实现非常诡异(通过二分法计算出文字所占区域大小)，效率极低，所以需要重新做一个效率比较高的控件出来。
* 看过一些开源的实现，包括 OHAttribtuedLabel，DTCoreText 和 Nimbus，总觉得他们实现插入图片的接口有点别扭，对于上层调用者来说 CoreText 部分不是完全透明的：调用者需要考虑怎么用自己的图片把原来内容替换掉。(当时的印象，现在具体怎么样已经不清楚了)
* 这是重新造轮子的机会！

直接拿了 Nimbus 的 AttributedLabel 作为基础，然后重新整理图文混排那部分的代码，调整接口，一共也就花了一个晚上的时间：拜一下 Nimbus 的作者。后来也根据项目的需求做了一些小改动，比如 hack iOS7 下计算 size 不准的问题，Label 上支持添加 UIView 的特性等等。最新的代码可以在 github 上找到:[M80AttributedLabel][2]。
不过写这篇文章最重要的原因不是为了放个代码出来，而是在闲暇时整理一下 iOS/OSX 文字排版相关的知识。

## 文字排版的基础概念

* 字体 (Font)：和我们平时说的字体不同，计算机意义上的字体表示的是同一大小，同一样式(Style) 字形的集合。从这个意义上来说，当我们为文字设置粗体，斜体时其实是使用了另外一种字体(下划线不算)。而平时我们所说的字体只是具有相同设计属性的字体集合，即 Font Family 或 typeface。
* 字符 (Character) 和字形 (Glyphs)：排版过程中一个重要的步骤就是从字符到字形的转换，字符表示信息本身，而字形是它的图形表现形式。字符一般指某种编码，如 Unicode 编码，而字形则是这些编码对应的图片。但是他们之间不是一一对应关系，同个字符的不同字体族，不同字体大小，不同字体样式都对应了不同的字形。而由于连写(Ligatures) 的存在，也会出现多个字符对应一个字形的情况。
![此处输入图片的描述][3]

* 字形描述集(Glyphs Metris)：即字形的各个参数。如下面的两张图:
![此处输入图片的描述][4]
![此处输入图片的描述][5]
* 边框(Bounding Box)：一个假想的边框，尽可能地容纳整个字形。
* 基线(Baseline)：一条假想的参照线，以此为基础进行字形的渲染。一般来说是一条横线。
* 基础原点(Origin)：基线上最左侧的点。
* 行间距(Leading)：行与行之间的间距。
* 字间距(Kerning)：字与字之间的距离，为了排版的美观，并不是所有的字形之间的距离都是一致的，但是这个基本步影响到我们的文字排版。
* 上行高度 (Ascent) 和下行高度(Decent)：一个字形最高点和最低点到基线的距离，前者为正数，而后者为负数。当同一行内有不同字体的文字时，就取最大值作为相应的值。如下图，

![此处输入图片的描述][6]
红框高度既为当前行的行高，绿线为 baseline，绿色到红框上部分为当前行的最大 Ascent，绿线到黄线为当前行的最大 Desent，而黄框的高即为行间距。由此可以得出：lineHeight = Ascent + |Decent| + Leading。
更加详细的内容可以参考苹果的这篇文档： [《Cocoa Text Architecture Guide》][7]。当然如果要做到更完善的排版，还需要掌握段落排版 (Paragragh Style) 相关的知识，但是如果只是完成聊天框内的文字排版，以上的基础知识已经够用了。详细的段落样式相关知识可以参考： [《Ruler and Paragraph Style Programming Topics》][8]

## CoreText

iOS/OSX 中用于描述富文本的类是 NSAttributedString，顾名思义，它比 NSString 多了 Attribute 的概念。它可以包含很多属性，粗体，斜体，下划线，颜色，背景色等等，每个属性都有其对应的字符区域。在 OSX 上我们只需解析完毕相应的数据，准备好 NSAttributedString 即可，底层的绘制完全可以交给相应的控件完成。但是在 iOS 上就没有这么方便，想要绘制 Attributed String 就需要用到 CoreText 了。(当然 iOS6 之后已经有 AttributedLabel 了。)
使用 CoreText 进行 NSAttributedString 的绘制，最重要的两个概念就是 CTFrameSetter 和 CTFrame。他们的关系如下：

![此处输入图片的描述][9]
其中 CTFramesetter 是由 CFAttributedString(NSAttributedString)初始化而来，可以认为它是 CTFrame 的一个 Factory，通过传入 CGPath 生成相应的 CTFrame 并使用它进行渲染：直接以 CTFrame 为参数使用 CTFrameDraw 绘制或者从 CTFrame 中获取 CTLine 进行微调后使用 CTLineDraw 进行绘制。
一个 CTFrame 是由一行一行的 CLine 组成，每个 CTLine 又会包含若干个 CTRun(既字形绘制的最小单元)，通过相应的方法可以获取到不同位置的 CTRun 和 CTLine，实现对不同位置 touch 事件的响应。
![此处输入图片的描述][10]

## 图文混排的实现

CoreText 实际上并没有相应 API 直接将一个图片转换为 CTRun 并进行绘制，它所能做的只是为图片预留相应的空白区域，而真正的绘制则是交由 CoreGraphics 完成。(像 OSX 就方便很多，直接将图片打包进 NSTextAttachment 即可，根本无须操心绘制的事情，所以基于这个想法，M80AttributedLabel 的接口和实现也是使用了 attachment 这么个概念，图片或者 UIView 都是被当作文字段中的 attachment。)
在 CoreText 中提供了 CTRunDelegate 这么个 Core Foundation 类，顾名思义它可以对 CTRun 进行拓展：AttributedString 某个段设置 kCTRunDelegateAttributeName 属性之后，CoreText 使用它生成 CTRun 是通过当前 Delegate 的回调来获取自己的 ascent，descent 和 width，而不是根据字体信息。这样就给我们留下了可操作的空间：用一个空白字符作为图片的占位符，设好 Delegate，占好位置，然后用 CoreGraphics 进行图片的绘制。以下就是整个图文混排代码描述的过程：
占位：
{% highlight objc %}
- ()appendAttachment: (M80AttributedLabelAttachment *)attachment
{
    attachment.fontAscent                   = _fontAscent;
    attachment.fontDescent                  = _fontDescent;
    unichar objectReplacementChar           = 0xFFFC;
    NSString *objectReplacementString       = [NSString stringWithCharacters:&objectReplacementChar length:1];
    NSMutableAttributedString *attachText   = [[NSMutableAttributedString alloc]initWithString:objectReplacementString];

    CTRunDelegateCallbacks callbacks;
    callbacks.version       = kCTRunDelegateVersion1;
    callbacks.getAscent     = ascentCallback;
    callbacks.getDescent    = descentCallback;
    callbacks.getWidth      = widthCallback;
    callbacks.dealloc       = deallocCallback;

    CTRunDelegateRef delegate = CTRunDelegateCreate(&callbacks, ( *)attachment);
    NSDictionary *attr = [NSDictionary dictionaryWithObjectsAndKeys:(__bridge id)delegate,kCTRunDelegateAttributeName, nil];
    [attachText setAttributes:attr range:NSMakeRange(0, 1)];
    CFRelease(delegate);

    [_attachments addObject:attachment];
    [self appendAttributedText:attachText];
}
{% endhighlight %}
实现委托回调:
{% highlight objc %}
CGFloat ascentCallback( *)
{
    M80AttributedLabelAttachment *image = (__bridge M80AttributedLabelAttachment *);
    CGFloat ascent = 0;
    CGFloat height = [image boxSize].height;
    switch (image.alignment)
    {
         M80ImageAlignmentTop:
            ascent = image.fontAscent;
            break;
         M80ImageAlignmentCenter:
        {
            CGFloat fontAscent  = image.fontAscent;
            CGFloat fontDescent = image.fontDescent;
            CGFloat baseLine = (fontAscent + fontDescent) / 2 - fontDescent;
            ascent = height / 2 + baseLine;
        }
            break;
         M80ImageAlignmentBottom:
            ascent = height - image.fontDescent;
            break;
        default:
            break;
    }
    return ascent;
}

CGFloat descentCallback( *)
{
    M80AttributedLabelAttachment *image = (__bridge M80AttributedLabelAttachment *);
    CGFloat descent = 0;
    CGFloat height = [image boxSize].height;
    switch (image.alignment)
    {
         M80ImageAlignmentTop:
        {
            descent = height - image.fontAscent;
            break;
        }
         M80ImageAlignmentCenter:
        {
            CGFloat fontAscent  = image.fontAscent;
            CGFloat fontDescent = image.fontDescent;
            CGFloat baseLine = (fontAscent + fontDescent) / 2 - fontDescent;
            descent = height / 2 - baseLine;
        }
            break;
         M80ImageAlignmentBottom:
        {
            descent = image.fontDescent;
            break;
        }
        default:
            break;
    }

    return descent;

}

CGFloat widthCallback(* )
{
    M80AttributedLabelAttachment *image  = (__bridge M80AttributedLabelAttachment *);
    return [image boxSize].width;
}
{% endhighlight %}
真正的绘制：
{% highlight objc %}
- ()drawAttachments
{
     ([_attachments count] == 0)
    {
        return;
    }
    CGContextRef ctx = UIGraphicsGetCurrentContext();
     (ctx == nil)
    {
        return;
    }

    CFArrayRef lines = CTFrameGetLines(_textFrame);
    CFIndex lineCount = CFArrayGetCount(lines);
    CGPoint lineOrigins[lineCount];
    CTFrameGetLineOrigins(_textFrame, CFRangeMake(0, 0), lineOrigins);
    NSInteger numberOfLines = [self numberOfDisplayedLines];
     (CFIndex i = 0; i < numberOfLines; i++)
    {
        CTLineRef line = CFArrayGetValueAtIndex(lines, i);
        CFArrayRef runs = CTLineGetGlyphRuns(line);
        CFIndex runCount = CFArrayGetCount(runs);
        CGPoint lineOrigin = lineOrigins[i];
        CGFloat lineAscent;
        CGFloat lineDescent;
        CTLineGetTypographicBounds(line, &lineAscent, &lineDescent, NULL);
        CGFloat lineHeight = lineAscent + lineDescent;
        CGFloat lineBottomY = lineOrigin.y - lineDescent;

        // Iterate through each of the "runs" (i.e. a chunk of text) and find the runs that
        // intersect with the range.
         (CFIndex k = 0; k < runCount; k++)
        {
            CTRunRef run = CFArrayGetValueAtIndex(runs, k);
            NSDictionary *runAttributes = (NSDictionary *)CTRunGetAttributes(run);
            CTRunDelegateRef delegate = (__bridge CTRunDelegateRef)[runAttributes valueForKey:(id)kCTRunDelegateAttributeName];
             (nil == delegate)
            {
                continue;
            }
            M80AttributedLabelAttachment* attributedImage = (M80AttributedLabelAttachment *)CTRunDelegateGetRefCon(delegate);

            CGFloat ascent = 0.0f;
            CGFloat descent = 0.0f;
            CGFloat width = (CGFloat)CTRunGetTypographicBounds(run,
                                                               CFRangeMake(0, 0),
                                                               &ascent,
                                                               &descent,
                                                               NULL);

            CGFloat imageBoxHeight = [attributedImage boxSize].height;
            CGFloat xOffset = CTLineGetOffsetForStringIndex(line, CTRunGetStringRange(run).location, nil);

            CGFloat imageBoxOriginY = 0.0f;
            switch (attributedImage.alignment)
            {
                 M80ImageAlignmentTop:
                    imageBoxOriginY = lineBottomY + (lineHeight - imageBoxHeight);
                    break;
                 M80ImageAlignmentCenter:
                    imageBoxOriginY = lineBottomY + (lineHeight - imageBoxHeight) / 2.0;
                    break;
                 M80ImageAlignmentBottom:
                    imageBoxOriginY = lineBottomY;
                    break;
            }

            CGRect rect = CGRectMake(lineOrigin.x + xOffset, imageBoxOriginY, width, imageBoxHeight);
            UIEdgeInsets flippedMargins = attributedImage.margin;
            CGFloat top = flippedMargins.top;
            flippedMargins.top = flippedMargins.bottom;
            flippedMargins.bottom = top;

            CGRect attatchmentRect = UIEdgeInsetsInsetRect(rect, flippedMargins);

            id content = attributedImage.content;
             ([content isKindOfClass:[UIImage class]])
            {
                CGContextDrawImage(ctx, attatchmentRect, ((UIImage *)content).CGImage);
            }
              ([content isKindOfClass:[UIView class]])
            {
                UIView *view = (UIView *)content;
                 (view.superview == nil)
                {
                    [self addSubview:view];
                }
                CGRect viewFrame = CGRectMake(attatchmentRect.origin.x,
                                              self.bounds.size.height - attatchmentRect.origin.y - attatchmentRect.size.height,
                                              attatchmentRect.size.width,
                                              attatchmentRect.size.height);
                [view setFrame:viewFrame];
            }
            
            {
                NSLog(@"Attachment Content Not Supported %@",content);
            }

        }
    }
}
{% endhighlight %}

详细的代码可以直接在 github 上查看:[https://github.com/xiangwangfeng/M80AttributedLabel/][11]。


  [1]: http://www.raywenderlich.com/4147/core-text-tutorial-for-ios-making-a-magazine-app
  [2]: https://github.com/xiangwangfeng/M80AttributedLabel
  [3]: /images/ct1.jpg
  [4]: /images/ct2.jpg
  [5]: /images/ct3.jpg
  [6]: /images/ct4.jpg
  [7]: https://developer.apple.com/library/mac/documentation/TextFonts/Conceptual/CocoaTextArchitecture/Introduction/Introduction.html#//apple_ref/doc/uid/TP40009459-CH1-SW1
  [8]: https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/Rulers/Rulers.html#//apple_ref/doc/uid/10000089i
  [9]: /images/ct5.jpg
  [10]: /images/ct6.jpg
  [11]: https://github.com/xiangwangfeng/M80AttributedLabel/