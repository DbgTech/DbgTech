---
title: Markdown语法模板
date: 2017-06-15 09:33:32
tags:
- Markdown
- 笔记
categories:
- 笔记
---

从最开始看到Markdown开始，到后面准备用Markdown记录笔记，心得；再到后来要鼓捣自己的静态页博客，断断续续学习了不少关于Markdown语法（请允许我这么称呼它）的资料。写东西时，总忘记怎么写，要查一下，这里就整理成博客，记录一下。

文章内容参考了如下几个资料：
1. [作业部落出品的编辑器](https://www.zybuluo.com/mdeditor)
2. [CSDN博客编辑器](http://write.blog.csdn.net/mdeditor)
3. [Markdown语法入门](http://www.appinn.com/markdown/)


### 1. 链接 ###

```
[来源](https://www.zybuluo.com/mdeditor)

This is [an example](http://example.com/ "Title") inline link.
```

显示如下：

[来源](https://www.zybuluo.com/mdeditor)

This is [an example](http://example.com/ "Title") inline link. （鼠标移上链接去有提醒）

### 2. 标题 / ReadMore 标记 ###

```
# 欢迎使用 Cmd Markdown 编辑阅读器 #
## 欢迎使用 Cmd Markdown 编辑阅读器 ##
### 欢迎使用 Cmd Markdown 编辑阅读器 ###
#### 欢迎使用 Cmd Markdown 编辑阅读器 ####
##### 欢迎使用 Cmd Markdown 编辑阅读器 #####
###### 欢迎使用 Cmd Markdown 编辑阅读器 ######

<!-- more -->  // 在文章中不显示，但是在jacman主题中被首页显示为 ReadMore 标签，打开文章
```

貌似Github编辑出来的标题，识别不了那么多，后面再研究一下！！！
<!-- more -->
一级标题

# 欢迎使用 Cmd Markdown 编辑阅读器 #

二级标题

## 欢迎使用 Cmd Markdown 编辑阅读器 ##

三级标题

### 欢迎使用 Cmd Markdown 编辑阅读器 ###

四级标题

#### 欢迎使用 Cmd Markdown 编辑阅读器 ####

五级标题

##### 欢迎使用 Cmd Markdown 编辑阅读器 ####

六级标题

###### 欢迎使用 Cmd Markdown 编辑阅读器 ######

结束




### 3. 分割线 ###

```
***

* * *

------

- - - - - - - - -

---------------------------------------
```

效果不明显，可以仔细看一下！！
第一个

***

第二个

* * *

第三个

------

第四个

- - - - - - - - -

第五个

---------------------------------------

结束

### 4. 引用文字 ###

```
> * 整理知识，学习笔记
> * 发布日记，杂文，所见所想
> * 撰写发布技术文稿（代码支持）
> * 撰写发布学术论文（LaTeX 公式支持）


*   A list item with a blockquote:

    > This is a blockquote
    > inside a list item.
```

如下，第一种是引用中使用列表，第二种是列表中使用引用

> * 整理知识，学习笔记
> * 发布日记，杂文，所见所想
> * 撰写发布技术文稿（代码支持）
> * 撰写发布学术论文（LaTeX 公式支持）


*   A list item with a blockquote:

    > This is a blockquote
    > inside a list item.

### 5. 图片 ###
```
相对目录(相对于站点根目录)：
![cmd-markdown-logo](\img\logo.png)

![cmd-markdown-logo](https://www.zybuluo.com/static/img/logo.png)

<div align="center">

![cmd-markdown-logo](https://www.zybuluo.com/static/img/logo.png)

</div>
```

相对目录(相对于站点根目录)：
![cmd-markdown-logo](/img/logo.png)

![cmd-markdown-logo](https://www.zybuluo.com/static/img/logo.png)

图片居中

<div align="center">

![cmd-markdown-logo](https://www.zybuluo.com/static/img/logo.png)

**  图片说明也居中 **

</div>

### 6. 粗体 和 斜体 ###

```
*single asterisks 斜体*

_single underscores 斜体_

**double asterisks 粗体**

__double underscores 粗体__

~~删除文字~~
```

*single asterisks 斜体*

_single underscores 斜体_

**double asterisks 粗体**

__double underscores 粗体__

~~删除文字~~

### 7. 列表 ###

```
- [ ] 支持以 PDF 格式导出文稿
- [ ] 改进 Cmd 渲染算法，使用局部渲染技术提高渲染效率
- [x] 新增 Todo 列表功能
- [x] 修复 LaTex 公式渲染问题
- [x] 新增 LaTex 公式编号功能

*   Red
*   Green
*   Blue

+   Red
+   Green
+   Blue

-   Red
-   Green
-   Blue

1.  Bird
2.  McHale
3.  Parish


*   Lorem ipsum dolor sit amet, consectetuer adipiscing elit.
	Aliquam hendrerit mi posuere lectus. Vestibulum enim wisi,
    viverra nec, fringilla in, laoreet vitae, risus.
*   Donec sit amet nisl. Aliquam semper ipsum sit amet velit.
	Suspendisse id sem consectetuer libero luctus adipiscing.


*   A list item with a blockquote:

    > This is a blockquote
    > inside a list item.
```

制作一份待办事宜:

- [ ] 支持以 PDF 格式导出文稿
- [ ] 改进 Cmd 渲染算法，使用局部渲染技术提高渲染效率
- [x] 新增 Todo 列表功能
- [x] 修复 LaTex 公式渲染问题
- [x] 新增 LaTex 公式编号功能

普通列表：

*   Red
*   Green
*   Blue

+   Red
+   Green
+   Blue

-   Red
-   Green
-   Blue

1.  Bird
2.  McHale
3.  Parish


*   Lorem ipsum dolor sit amet, consectetuer adipiscing elit.
	Aliquam hendrerit mi posuere lectus. Vestibulum enim wisi,
    viverra nec, fringilla in, laoreet vitae, risus.
*   Donec sit amet nisl. Aliquam semper ipsum sit amet velit.
	Suspendisse id sem consectetuer libero luctus adipiscing.


*   A list item with a blockquote:

    > This is a blockquote
    > inside a list item.

### 8. 特殊文本插入 ###

```
一个 LaTeX 公式：
$$E=mc^2$$

一段代码：
```python
@requires_authorization
class SomeClass:
    pass

if __name__ == '__main__':
    # A comment
    print 'hello world'
```// 实际写时没有该段注释，为了防止 反引号被解析，而加入
```

公式与注脚 在Github生成静态页上貌似没效果：
书写一个质能守恒公式[^LaTeX] 

$$E=mc^2$$

高亮一段代码[^code]

```python
@requires_authorization
class SomeClass:
    pass

if __name__ == '__main__':
    # A comment
    print 'hello world'
```

### 9. 图类编写

```
```flow	//注释
st=>start: Start
op=>operation: Your Operation
cond=>condition: Yes or No?
e=>end

st->op->cond
cond(yes)->e
cond(no)->op
```		//注释

序列图

```sequence  	// 注释
Alice->Bob: Hello Bob, how are you?
Note right of Bob: Bob thinks
Bob-->Alice: I am good thanks!
```    	// 注释

甘特图

```gantt    // 注释
    title 项目开发流程
    section 项目确定
        需求分析       :a1, 2016-06-22, 3d
        可行性报告     :after a1, 5d
        概念验证       : 5d
    section 项目实施
        概要设计      :2016-07-05  , 5d
        详细设计      :2016-07-08, 10d
        编码          :2016-07-15, 10d
        测试          :2016-07-22, 5d
    section 发布验收
        发布: 2d
        验收: 3d
```		 // 注释
```

如下这些图貌似在Github上生成静态页没效果，原因待研究。

[流程图](https://www.zybuluo.com/mdeditor?url=https://www.zybuluo.com/static/editor/md-help.markdown#7-流程图)

```flow
st=>start: Start
op=>operation: Your Operation
cond=>condition: Yes or No?
e=>end

st->op->cond
cond(yes)->e
cond(no)->op
```

[序列图](https://www.zybuluo.com/mdeditor?url=https://www.zybuluo.com/static/editor/md-help.markdown#8-序列图)

```sequence
Alice->Bob: Hello Bob, how are you?
Note right of Bob: Bob thinks
Bob-->Alice: I am good thanks!
```

[甘特图](https://www.zybuluo.com/mdeditor?url=https://www.zybuluo.com/static/editor/md-help.markdown#9-甘特图)

```gantt
    title 项目开发流程
    section 项目确定
        需求分析       :a1, 2016-06-22, 3d
        可行性报告     :after a1, 5d
        概念验证       : 5d
    section 项目实施
        概要设计      :2016-07-05  , 5d
        详细设计      :2016-07-08, 10d
        编码          :2016-07-15, 10d
        测试          :2016-07-22, 5d
    section 发布验收
        发布: 2d
        验收: 3d
```

### 10. 绘制表格 ###

```
| 项目        | 价格   |  数量  |
| --------   | -----: | :----:  |
| 计算机      | $1600  |   5     |
| 手机        |   $12  |   12   |
| 管线        |   $1   |  234  |
```

冒号为该表格的对齐方式，左对齐或右对齐，两边添加就是居中。

| 项目        | 价格   |  数量  |
| --------   | -----:  | :----:  |
| 计算机      | $1600 |   5     |
| 手机        |   $12   |   12   |
| 管线        |    $1    |  234  |


### 11. 添加 图标  ###

```
<i class="icon-share"></i> 发布：将当前的文稿生成固定链接，在网络上发布，分享
<i class="icon-file"></i> 新建：开始撰写一篇新的文稿
<i class="icon-trash"></i> 删除：删除当前的文稿
<i class="icon-cloud"></i> 导出：将当前的文稿转化为 Markdown 文本或者 Html 格式，并导出到本地
<i class="icon-reorder"></i> 列表：所有新增和过往的文稿都可以在这里查看、操作
<i class="icon-pencil"></i> 模式：切换 普通/Vim/Emacs 编辑模式
```

将一些网页中使用的图标，添加进来

<i class="icon-share"></i> 发布：将当前的文稿生成固定链接，在网络上发布，分享
<i class="icon-file"></i> 新建：开始撰写一篇新的文稿
<i class="icon-trash"></i> 删除：删除当前的文稿
<i class="icon-cloud"></i> 导出：将当前的文稿转化为 Markdown 文本或者 Html 格式，并导出到本地
<i class="icon-reorder"></i> 列表：所有新增和过往的文稿都可以在这里查看、操作
<i class="icon-pencil"></i> 模式：切换 普通/Vim/Emacs 编辑模式

### 12. 参考文献 与 脚注 ###

```
生成一个脚注[^footnote].
  [^footnote]: 这里是 **脚注** 的 *内容*.

作者 [@ghosert][3] 2016 年 07月 07日

[^LaTeX]: 支持 **LaTeX** 编辑显示支持，例如：$\sum_{i=1}^n a_i=0$， 访问 [MathJax][4] 参考更多使用方法。

[^code]: 代码高亮功能支持包括 Java, Python, JavaScript 在内的，**四十一**种主流编程语言。

[1]: https://www.zybuluo.com/mdeditor?url=https://www.zybuluo.com/static/editor/md-help.markdown
[2]: https://www.zybuluo.com/mdeditor?url=https://www.zybuluo.com/static/editor/md-help.markdown#cmd-markdown-高阶语法手册
[3]: http://weibo.com/ghosert  "ghosert"
[4]: http://meta.math.stackexchange.com/questions/5020/mathjax-basic-tutorial-and-quick-reference
```

注脚这个貌似在Github上生成的静态页没效果

生成一个脚注[^footnote].
  [^footnote]: 这里是 **脚注** 的 *内容*.

作者 [@ghosert][3] 2016 年 07月 07日

[^LaTeX]: 支持 **LaTeX** 编辑显示支持，例如：$\sum_{i=1}^n a_i=0$， 访问 [MathJax][4] 参考更多使用方法。

[^code]: 代码高亮功能支持包括 Java, Python, JavaScript 在内的，**四十一**种主流编程语言。

[1]: https://www.zybuluo.com/mdeditor?url=https://www.zybuluo.com/static/editor/md-help.markdown
[2]: https://www.zybuluo.com/mdeditor?url=https://www.zybuluo.com/static/editor/md-help.markdown#cmd-markdown-高阶语法手册
[3]: http://weibo.com/ghosert  "ghosert"
[4]: http://meta.math.stackexchange.com/questions/5020/mathjax-basic-tutorial-and-quick-reference

### 13. 生成目录 ###

```
[TOC]
```

如下是生成的一个目录，注意`[TOC]`前后要有空行。（但是这个在Github上貌似没效果）

[TOC]

** 参考文档 **

1. [作业部落出品的编辑器](https://www.zybuluo.com/mdeditor)
2. [CSDN博客编辑器](http://write.blog.csdn.net/mdeditor)
3. [Markdown语法入门](http://www.appinn.com/markdown/)


By Andy @2017-08-07 09:54:23