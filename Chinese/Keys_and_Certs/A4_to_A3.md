## 背景介绍

给学生打印试卷经常需要用到把 A4 页面的 PDF 按照 A3 页面尺寸来进行打印，尤其 A Level 试卷都是很多页的。但是学校的打印室人员却不懂怎么操作；甚至，教务处所合作的校外专业的打印室也经常出错。只要文档中有矢量图，就会出现漏图、错图等各种错误情况；导致我被学生误解和指责。

所以，最终我还是决定自己来解决这个问题。

研究了一番，搞定！写篇文章记录一下。

简单来说就是，我实现了一个自动化程序，这个程序负责把 A4 排版的 PDF 文件处理成可以进行 A3 格式打印的 PDF 文件，把这个新的文件交给打印室的工作人员直接按照多页方式打印即可，他们不需要进行任何额外设置。

当然，在找到最优方案之前，我调研和尝试了许多种不同的方案，所以这里一并记录一下，可能略显繁琐。不过，也方便感兴趣的朋友窥个全貌。

#### 解决方案

由于我的日常使用电脑都是 MacOS 系统，所以我接下来的解决方案都是基于 MacOS 环境。

##### 方案一：Booklet 小册子打印

**(i) PDF 打印 Booklet**

小册子打印跟 Printer 的功能有关，并不是所有的 Printer 都具有 Booklet 打印能力，因此，当我们 CMD + P 调出打印机设置时，会看到不同的选项，比如：

我们选择 Layout 后，

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/preview_layout1.jpg)

会出现更多的页面选择框

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/layout_dropdown2.jpg)

如果所选择的 Printer 支持自动双页打印（deplex），那么这里的 Two-Sided 会有几个下拉选择项，否则（如果 Printer 不支持 deplex），这里的选择框就是灰色，如下：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/layout_twoside.jpg)

但是，具有 Booklet 打印功能的 Printer（比如 EPSON L4260 Series），在 Layout 选项里就会出现 Booklet 选项，如下：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/layout_portrait_juejin1.jpg)

选择了 Booklet 方式打印后，Pages per Sheet 就是不可选的了，因为是固定每张纸打两页。

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/layout_more_juejin.jpg)

在实际打印前，我们可以虚拟打印一下以便能看到打印后的真实效果，避免打错浪费纸的情况，那么可以这样操作：

点击设置页坐下角的 PDF 下拉按钮，选择 Save as Adobe PDF （注意，这里的前提是已经安装了 Adobe Acrobat Pro 软件 MacOS），

然后按提示步骤生成打印后的 PDF 文件，打开 PDF 文件后就可以看到真实效果。

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/printer_pdf_save_as.jpg)

注意：

- 1）不要直接点击 Print

- 2）我安装了 Air Print/Print to PDF 两个 Virtual Printer App，他们使用的 Driver 是 MacOS 自带的 PostScript，并不支持 Booklet 打印（打印设置页没有 Booklet 选项）。

另外，如果我们没有支持 A3 打印的打印机，我们可以使用 Adobe Acrobat Pro 这款 App 所提供的打印能力来实现 Booklet 打印。

具体方法如下：

- (1) 使用 Adobe Acrobat Pro 打开 PDF 文件

- (2) 按快捷键 CMD + P 调出打印页面，并选择 Booklet 方式打印。

没错，即使你没有能打印 Booklet 的打印机，但 Adobe Acrobat Pro 提供了 Booklet 打印功能。配合 Virtual Printer App，你就可以实现 Booklet 打印。

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/air_printer.jpg)

如果点击左下角的 Printer 按钮去设置系统打印机，那么会出现如下提示。这也证明了， Adobe Acrobat 提供的 Driver 更强大。

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/adobe_print_prompt.jpg)

#### 问题：

**问题一：** Mac Store 中的 Air Printer 或 Print to PDF 两款 Virtual Printer App，免费版本都能打印不超过 5 页的文档，正版则需要 \$39.99。

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/air_printer_price.jpg)

破解版（免费下载）：

<https://macoshome.com/app/productivity/17303.html#Down>

**问题二：** Adobe Acrobat Pro 默认打印出来的文件，PDF 中的图片会缺失或错乱。比如像下面这样：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/printer_scambled.jpg)

原因：

If fonts and images not supported by acrobat are embedded in a PDF from another source this issue may occur。

###### 解决方法：

点击 Advanced 按钮，

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/printer_adobe_setting.jpg)

然后，在新弹出的设置页面中，勾选 Print As Image

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/printer_adobe_setting1.jpg)

另：Air Printer 在 Windows/MacOS 中都有，我们可以下载破解版。其他类似的软件还有 Print Conductor。

<https://air-printer.softonic.cn/>

<https://airprint.softonic.cn/>

**问题三：** Air Printer 不支持打印 A3

**解决方案：** 使用支持 A3 打印的 Printer（比如 EPSON L4260 Series），然后选择 Save As Adobe PDF。

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/air_printer_no_a3.jpg)

不过，经过测试，打印出来的 A3 文件会有大量的 margin，也就是缩放了，暂时不知道如何解决。

因此，最简单的方式，还是直接使用 Booklet 打印功能。

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/printer_vertical.jpg)

**(ii) Word 打印 Booklet**

1、打开需要编辑的Word文档，点击打开页面布局中的“分栏”，选择“两栏”选项。

2、再选择纸张大小 A3，纸张方向横向。

3、打印预览，在上标尺处可以调整布局，打印方式选择单双面。

##### 方案二：手动折页打印

在方案一中，Printer 提供的 Booklet Driver 帮我们实现了自动折页打印的效果。我们也可以在本地把 PDF 文件按照 Booklet 折页的顺序排列好，然后手动进行一张多页的打印方式进行打印。

注：所谓的一张多页，也就是 MacOS 中的打印方式：Layout + Soft-Edge Binding）

Windows 中，一张多页打印的设置页面如下：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/printer_layout_order.jpg)

**现在问题的核心就是：**

**我们如何手动解决 A4 页面排序成 Booklet 所要求的页面顺序？**

一张 A3 纸，正反面打印的话，是包含了 4 张 A4 大小的页面的，因此，A4 尺寸的文件的页数需要是 4 的倍数，如果不是 4 的倍数，那么我们必须手动添加空白页，使得它变成 4 的倍数。（比如文件有 10 页，那么应该添加两个空白页变成 12 页），否则的话就无法实现正确的 Booklet 效果。

手动添加页面的方式：

用 Preview App 打开 pdf 文件，然后选择 Edit > Insert > Blank Page

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/preview_layout_juejin.jpg)

另外，为了跟原文件保持风格一致，添加的空白页还应该添加页码。Preview 没法做到这个，只能使用 PDF 编辑软件了。

接下来，我们研究一下 Booklet 效果所需的页面顺序：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/booklet_order.jpg)

如上，总页数是 8 页。那么第一张 A3 纸（正反面）应该按照 8 1 2 7 的顺序打印，第二张 A3 纸应该按照 6 3 4 5 的顺序打印。

同理，如果一个文件是 12 页，那么顺序应该如下：

> 12 1 2 11
> 
> 10 3 4 9
> 
> 8 5 6 7

如果一个文件是 16 页，那么顺序如下：

> 16 1 2 15
> 
> 14 3 4 13
> 
> 12 5 6 11
> 
> 10 7 8 9

参考：<https://helpx.adobe.com/acrobat/kb/print-booklets-acrobat-reader.html>

现在，虽然解决了排序问题，但我们不可能每次在 PDF 编辑软件里面手动来排序页面吧，页面少还好，页面多的话实在太繁琐，浪费时间，

作为程序员的敏感，自然是要写个程序来自动化这个事情。


#### 代码实现

前面已经分析的很详细了，我们要做的事情就是让程序去按照 Booklet 的方式去重排文档，重拍之后的文档再按照双页打印的方式打印到 A3 的纸张上就可以了。

Automator 是 MacOS 上一个创建自动化流程的工具，我们可以把一系列操作集成为一个 workflow，从而可以简化许多自动化的工作。
我相信平常使用 MacOS 的人应该都至少有听说过这个工具，Windows 以及 iOS 和安卓系统上其实也都有类似的工具，这类工具的就是来帮助我们流程化一些日常操作的。

我们需要的就是一丁点的编程能力。

下面，就是我在 MacOS 上编写的 Automator 脚本，它实现了我们上述的要求。

如果你需要类似的功能，我免费分享到了这个网盘地址：

下载链接：[https://pan.baidu.com/s/1cchQtI7c0KFwBPIazhFHbg?pwd=pi76](https://pan.baidu.com/s/1cchQtI7c0KFwBPIazhFHbg?pwd=pi76)
提取码: pi76

下载之后，解压，双击安装，会出现如下对话框：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/automator_installer_juejin1.png)

点击 Install 即可。或者，你也可以直接把解压后的文件放入 `~/Library/Services` 目录。
Script 安装完成之后，在 pdf 文件点击右键，点击 Make booklet PDF on Desktop，
如下：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/print_image_quickactions.jpg)

执行完成之后，这个 Script 会把 PDF 文件页面按照 Booklet 所要求的顺序重新排好，然后我们只需要在打印机中选择一张多页的方式打印即可。
卸载：
进入 `~/Library/Services` 目录，删除即可。

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/macos_quickactions_juejin1.jpg)

最后，再啰嗦几句虚拟打印的知识点：

所谓的虚拟打印就是指在电脑上按照指定格式打印成 PDF，（比如打印出 A3 排版的 PDF）。好处是可以在实际打印前先预览效果，避免会浪费纸张。

像之前介绍的一样，有两种方式，CMP + P 调出打印设置页面，

- （1）选择 Air Printer （虚拟打印机）直接打印（会生成一个 pdf）

- （2）点击左下角 PDF 下拉选项，选择 Save as Adobe PDF 

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/virt_print_save_as.jpg)

总结：
1. PDF + Script + 一行多页 (A3 打印）
2. PDF + Printer (A3 ability) + Booklet (Save as Adobe PDF)
3. PDF + Adobe Acrobat Pro Printer (A3 ability) + Virtual Printer (no A3 ability)

第三种缺点是：有些文件必须勾选 Advanced > Print As Image，否则图像会花。
如果勾选了 Print As Image，那么输出的 PDF 文件 size 更大。否则会更小。

##### 实际打印：

如果使用 Virtual Printer 按照打印格式（即 Booklet 方式）输出了 PDF 文件，在实际打印的时候，可以选择打印 A4 或者 A3 都可以。

记住勾选下面几点设置：

- 普通打印（不要勾选一张多页或者 Booklet，因为虚拟打印的时候已经输出了 Booklet 方式）
- 双面打印
- 长边翻转


长边翻转 vs 短边翻转

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/langside_flip_jj.jpg)


长边翻转，是指以纸张【长边】一边为准，不动，进行翻页。分为向左翻页的【左装订】和向右翻页的【右装订】，还有向上翻
页的【上装订】。一般双面打印长边翻转默认【左装订】。具体装订方式，还需要看纸张方向，是横向还是纵向
短边翻转，是指以纸张【短边】为准不动，进行翻页。一般有以短边为准的向左翻页的【左装订】，和向上翻页的【上装订】。具
体装订方式，还需要看纸张方向，是横向还是纵向，

注：纸张是纵向的，就沿纵向装订；纸张是横向的，就沿横向装订；

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/hengxiang_zhongxiang.png)

注：在双面打印中，系统默认为翻转长边页面，这种方式也是最常見的。

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/print_longside_flip.jpg)



全文完！

如果你喜欢我的文章，欢迎关注我的公众号 codeandroad
