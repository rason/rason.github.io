title: Markdown语法
categories: Markdown
date: 2015-08-20 15:13:41
tags: Markdown
description: Markdown语法
---

## 常用语法帮助
| 输出后的效果 | Markdown |
| :-----------: | :-------: |
| **Bold** | \*\*text\*\* |
| *Emphasize* | \*text\* |
| ~~Strike-through~~ | \~\~text\~\~ |
| [Link](http://) | \[title\]\(http://\) |
| `Inline Code` | \`code\` |
| Image| \!\[alt\]\(http://\) |
| List | \* item |
| Blockquote | \> quote |
| H1 | \# Heading |
| H2 | \#\# Heading |
| H3 | \#\#\# Heading |            

<!--more-->

## 标题
```
# 一级标题
## 二级标题
### 三级标题
```

## **粗体** *斜体*
```
**粗体**
*斜体*
```

## ~~删除线~~
```
~~删除线~~
```

## 水平线
--------
```
--------
```

##  无序列表
```
符号之后的空格不能少，-+*效果一样，但不能混合使用，因混合是嵌套列表，内容可超长

- 列表一
- 列表二
- 列表三

或者

* 列表二
* 列表二
* 列表三
```

## 有序列表
```
1. 列表一
2. 列表二
3. 列表三
```
1. 列表一
2. 列表二
3. 列表三

## 引用
```
> Markdown 是一种轻量级标记语言，它允许人们使用易读易写的纯文本格式编写文档，然后转换成格式丰富的HTML页面
```
> Markdown 是一种轻量级标记语言，它允许人们使用易读易写的纯文本格式编写文档，然后转换成格式丰富的HTML页面

## 文字链接
```
[点击这里进入百度](URL)
```
[点击这里进入百度](http://www.baidu.com)

## 图片链接
```
相对文字链接多个感叹号，Tooltips可省略
![GitHub-Mark](http://github.global.ssl.fastly.net/images/modules/logos_page/GitHub-Mark.png "GitHub-Mark")
```
![GitHub-Mark](http://github.global.ssl.fastly.net/images/modules/logos_page/GitHub-Mark.png "GitHub-Mark")

## 自动链接
```
<http://rason.github.io>
```
<http://rason.github.io>

## 索引链接
```
[Rason's Blog][1]
![GitHub-Octocat][2]

[1]: http://rason.github.io
[2]: http://github.global.ssl.fastly.net/images/modules/logos_page/Octocat.png
```
[Rason's Blog][1]
![GitHub-Octocat][2]

[1]: http://rason.github.io
[2]: http://github.global.ssl.fastly.net/images/modules/logos_page/Octocat.png


## 行内代码
```
	 `alert('Hello World');`
```
示例： `alert('Hello World');`

## 块级代码
```
每行文字前加4个空格或者1个Tab
public class HelloWorld {
	public static void main(String[] args){
		System.out.println("HelloWorld");
	}
}

```

## 转移字符
```
Markdown中的转义字符为\，转义的有：
\\ 反斜杠
\` 反引号
\* 星号
\_ 下划线
\{\} 大括号
\[\] 中括号
\(\) 小括号
\# 井号
\+ 加号
\- 减号
\. 英文句号
\! 感叹号
```
\\ \` \* \_ \{\} \[\] \(\) \# \+ \- \. \!

原创文章，欢迎转载，无需注明出处。

参考文献
1. [Markdown语法，百度百科](http://baike.baidu.com/link?url=JpeLXH_KHxWXLEfcElZUq9zyB2G-H0QFGOYhGNjTeKyw-1PV0i2EqRWw3SMavbLTbHL8xV2iCSOjFeP-IefTtq)
2. [不如博客](http://ibruce.info/2013/11/26/markdown/)




