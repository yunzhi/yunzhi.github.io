---
layout: post
title: 正则表达式
description: 记录下正则表达式的用法
category: blog
---

某日，在通州图书管极少的计算机图书中，发现了这一本《学习正则表达式》，拿来看之余，记录下一些内容。以后有时间整理下

## 正则表达式 regular expressions

1. \d可以像[0-9]一样匹配任何数字。这种正则表达式式叫做字符组简写式。也叫转义字符。\D匹配任一非数字字符
2. .（点）用来匹配任一字符（某些情况下不能匹配行起始符）
3. 捕获分组，向后引用。要创建捕获分组，现将一个\d放在一对圆括号中，后面可以用\1或者$1来引用。如：

		(\d)\d\1		\* 匹配 123 等*\
4. 使用量词 {}
5. 扩选文字符 ( | )
6. 匹配单词- \w：匹配字母，数字和下划线。\W匹配非单词字符
7. 匹配空白符\s. 会匹配空格，制表(\t)，换行(\n)，回车(\r)
8. 断言也成零宽度断言。单词边界\b，非单词边界\B。零宽度断言不会匹配两边的字符，但是它会识别文字两边是否是单词边界。有些应用程序中，指定单词边界的是另一种用法，\<来指定单词的开头，\>来指定单词的结尾（vim上就是这样的语法）
9. grep -E表示使用扩展的正则表达式（ERE），而不是grep默认使用的基本正则表达式（BRE）
10. 选项：（?i）让匹配的模式不再区分大小写。（？x）忽略空格和注释。
11. 命名分组：(?<name>...) 命名分组

		(?<z>0{3})\k<z> 匹配 000000
12. 非捕获分组(?:) 如果分组不需要后向引用，例如，匹配大小写均可的the

		（？i:the）
13. POSIX 字符组

		[[:alnum:]]	 匹配字母及数字
		[[:alpha:]]	匹配字母
		[[:ascii:]]		匹配ascii字符（128个）
		[[:blank:]]	匹配空白字符
		[[:ctrl:]]	匹配控制字符
		[[:digit:]]	匹配数字
		[[:graph:]]	匹配图形字符
		[[:lower:]]	匹配小写字母
		[[:print:]]	匹配可打印字符
		[[:punct:]]	匹配标点符号
		[[:space:]]	匹配空格
		[[:upper:]]	匹配大写字符
		[[:word:]]		匹配单词字符
		[[:xdigit:]]	匹配十六进制数字
14. 量词。

		？ 	零个或一个
		+ 	一个或多个
		* 	零个或多个
		{n} 	匹配n次
		{n，} 	至少n次
		{m,n} 	匹配m到n次
		{0，1} 	与？相同
		{1，}	与+相同
		{0，} 	与*相同

15. 环视。

正前瞻：(?)exp(?=)

例如，一个文本文件foo.txt内容为 `this is a test`

如下的perl语句将会打印出该语句：

		perl 'print if /(?i)this (?=is)/' foo.txt ;#i表示忽略大小写

反前瞻：这个就比较有用：`/(?i)this (?!is)/`
 
 正后顾：与正前瞻方向相反，会查看左边的内容

		perl 'print if /(?i)(?<=this) is/' foo.txt
反后顾：`perl print 'if /(?i)(?<!this) is/' foo.txt`		

16.一个perl脚本转化一段文字为HTML格式

        #!/usr/bin/perl -p
        
        if ($. == 1){
          chomp($title = $_);
        }
        
        print "<!DOCTYPE html>\
        <html xmlns=\"http://www.w3.org/1999/xhtml\">\
          <head>\
            <title>$title</title>\
          </head>
        <body>\
          <h1>$title</h1>\n" if $. == 1;
        
        s/^(ARGUMENT|I{0,3}V?I{0,2})\.$/<h2>$1<\/h2>/;
        if ($. == 5) {
          s/^([A-Z].*)$/<p>$1<\/p>/;
        }
        
        if ($. == 9) {
          s/^[  ]*(.*)/ <p>$1<br\/>/;
        }
        
        if (10...832) {
          s/^[  ]{5,7}.*/$1<br\/>/;
        }
        
        if (9...eof) {
          s/^(.*)$/$1<\/p>\n  <\/body>\n<\/html>\n/;
        }

## 参考书目
1. 《学习正则表达式》人民邮电出版社 2013 Micbael Fitzgerald等，王热宇译
2.  FCRE 

["Yunzhi made"](http://yunzhi.github.io) &copy;
