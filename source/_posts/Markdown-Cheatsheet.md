---
title: Markdown Cheatsheet
date: 2017-05-31 11:38:38
tags:
    - Markdown
    - 备忘
    - Cheat Sheet
---


开始写博客后，Markdown成为一种常用的文档格式，Markdown 的语法全由一些符号所组成，这些符号经过精挑细选，其作用一目了然，里面的语法记下来，为了以后查询方便。

## 文本样式

```
*斜体*或_斜体_
**粗体**
***加粗斜体***
~~删除线~~
```

- *斜体* 或 _斜体_
- **粗体**
- ***加粗斜体***
- ~~删除线~~

- *This text will be italic*
- _This will also be italic_
- **This text will be bold**
- __This will also be bold__
- *You **can** combine them*

<!-- more -->

## 引用

使用大于号 `>` 表示引用内容：

As Grace Hopper said:
> I’ve always been more interested
> in the future than in the past.

## 分级标题

第一种写法

```
这是一个一级标题
============================
这是一个二级标题
--------------------------------------------------
```
第二种写法

```
# 一级标题
## 二级标题
### 三级标题
#### 四级标题
##### 五级标题
###### 六级标题
```

## 超链接

Markdown 支持两种形式的链接语法： 行内式和参考式两种形式，行内式一般使用较多。

### 行内式

语法说明：

[]里写链接文字，()里写链接地址, ()中的”“中可以为链接指定title属性，title属性可加可不加。title属性的效果是鼠标悬停在链接上会出现指定的 title文字。[链接文字](链接地址 “链接标题”)’这样的形式。链接地址与链接标题前有一个空格。

```
欢迎来到[我的博客](https://lvraikkonen.github.io/)
欢迎来到[我的博客](https://lvraikkonen.github.io/ "博客名")
```

- 欢迎来到[我的博客](https://lvraikkonen.github.io/)
- 欢迎来到[我的博客](https://lvraikkonen.github.io/ "博客名")

### 参考式

参考式超链接一般用在学术论文上面，或者另一种情况，如果某一个链接在文章中多处使用，那么使用引用 的方式创建链接将非常好，它可以让你对链接进行统一的管理。

语法说明：
参考式链接分为两部分，文中的写法 [链接文字][链接标记]，在文本的任意位置添加[链接标记]:链接地址 “链接标题”，链接地址与链接标题前有一个空格。

如果链接文字本身可以做为链接标记，你也可以写成[链接文字][]
[链接文字]：链接地址的形式，见代码的最后一行。

```
我经常去的几个网站[Google][1]、[Leanote][2]以及[自己的博客][3]
[Leanote 笔记][2]是一个不错的[网站][]。
[1]:http://www.google.com "Google"
[2]:http://www.leanote.com "Leanote"
[3]:https://lvraikkonen.github.io "lvraikkonen"
[网站]:https://lvraikkonen.github.io
```

我经常去的几个网站[Google][1]、[Leanote][2]

以及[自己的博客][3]，[Leanote 笔记][2]是一个不错的[网站][]。

[1]:http://www.google.com "Google"
[2]:http://www.leanote.com "Leanote"
[3]:https://lvraikkonen.github.io "lvraikkonen"
[网站]:https://lvraikkonen.github.io

### 自动链接

语法说明：
Markdown 支持以比较简短的自动链接形式来处理网址和电子邮件信箱，只要是用<>包起来， Markdown 就会自动把它转成链接。一般网址的链接文字就和链接地址一样，例如：

```
<http://example.com/>
<address@example.com>
```

显示效果：

- <http://example.com/>
- <address@example.com>

## 列表

无序列表使用星号 `*`、加号 `+`或是减号 `-` 作为列表标记

*   Red
*   Green
*   Blue

有序列表则使用数字接着一个英文句点：

1.  Bird
2.  McHale
3.  Parish


列表项目可以包含多个段落，每个项目下的段落都必须缩进 4 个空格或是 1 个制表符：

1.  This is a list item with two paragraphs. Lorem ipsum dolor
    sit amet, consectetuer adipiscing elit. Aliquam hendrerit
    mi posuere lectus.

    Vestibulum enim wisi, viverra nec, fringilla in, laoreet
    vitae, risus. Donec sit amet nisl. Aliquam semper ipsum
    sit amet velit.

2.  Suspendisse id sem consectetuer libero luctus adipiscing.

如果要在列表项目内放进引用，那 `>` 就需要缩进：

*   A list item with a blockquote:

    > This is a blockquote
    > inside a list item.

## 图像

和链接很相似的语法来标记图片，同样也允许两种样式： 行内式和参考式

### 行内式

- 一个惊叹号 !
- 接着一个方括号，里面放上图片的替代文字
- 接着一个普通括号，里面放上图片的网址，最后还可以用引号包住并加上 选择性的 'title' 文字

```
![Alt text](/path/to/img.jpg)

![Alt text](/path/to/img.jpg "Optional title")
```

Inline-style:
![alt text](https://github.com/adam-p/markdown-here/raw/master/src/common/images/icon48.png "Logo Title Text 1")


### 参考式

id是图片参考的名称，图片参考的定义方式则和链接参考一样：
```
![Alt text][id]
[id]: url/to/image  "Optional title attribute"
```

Reference-style:
![alt text][logo]

[logo]: https://github.com/adam-p/markdown-here/raw/master/src/common/images/icon48.png "Logo Title Text 2"

## 代码

如果要标记一小段行内代码，你可以用反引号把它包起来 `code`

也可以使用三个反引号包裹一段代码，并指定一种语言

```javascript
var s = "JavaScript syntax highlighting";
alert(s);
```

```python
s = "Python syntax highlighting"
print s
```

```
No language indicated, so no syntax highlighting.
But let's throw in a <b>tag</b>.
```


## 表格

| Tables        | Are           | Cool  |
| ------------- |:-------------:| -----:|
| col 3 is      | right-aligned | $1600 |
| col 2 is      | centered      |   $12 |
| zebra stripes | are neat      |    $1 |


以下是Github支持的markdown语法：

## To-Do list

TASK LISTS
- [x] this is a complete item
- [ ] this is an incomplete item
- [x] @mentions, #refs, [links](), **formatting**, and <del>tags</del> supported
- [x] list syntax required (any unordered or ordered list supported)


## EMOJI

GitHub supports emoji!
:+1: :sparkles: :camel: :tada:
:rocket: :metal: :octocat:
