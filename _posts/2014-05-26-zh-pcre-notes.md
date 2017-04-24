---
layout: post
title: 非主流的 PCRE 学习笔记
---

记了也是白记，很容易查，只是：“好记性不如烂笔头”。  

非捕获分组和零宽断言
----
会用 `[]()|+*{}` 之后，正则给人感觉还是蛮简单的，但实际匹配比较复杂，特别是从字符串里去 MATCH。
	(?: ) # 非捕获分组
	(?= )  (?! ) # 零宽先行断言（前瞻、向后环视）
	(?<= ) (?<! ) # 零宽后发断言（后瞻、向前环视）
非捕获分组都还理解，但是非捕获分组会消耗（吞）掉非捕获的部分，如果只是想协助判断，但是不吞掉，就应该使用零宽断言。
目前大部分正则都支持零宽先行断言，但个别语言不支持零宽后发断言（可能是出于实现性能考虑，因为相当于要逆序检测），因此还是需要谨慎使用零宽后发断言。

分隔符：Delimiters
----
特妹的除了下面的字符，都可以作为分隔符。
	字母数字（alphanumeric）：[a-zA-Z0-9]
	反斜线（backslash）:\
	空白字符（whitespace）:[\s\t\r\n]
我一直以为只能是 / 呢

[!code=php]
转义字符 \R
----
### 实例
	echo preg_match_all('/\R/',"\r\r\n")."\r\n";
	echo preg_match_all('/\R/',"\n\n\r")."\r\n";
	echo preg_match_all('/\R/',"\n\r\n")."\r\n";
	echo preg_match_all('/[\R]/',"R");
### 输出
	2
	3
	2
	1
### 结论
优先匹配 \r\n，落单的 \r 和 \n 也能够被匹配。不能作为单字符参与匹配会被等效于 R 字符本身。
\R 本身是没有出现在官方文档里的，他在零宽断言里的行为好像也有点不正常。可以使用 `(?:\r|\n|\r\n)` 代替。

函数 preg_replace_callback
----
替换修饰符 e 的函数，非常好用。但一定要注意合理使用断言，什么地方消耗，什么地方应该使用零宽，要掌握好。

大型匹配
----
match 的结果是个数组，成员 0 为符合条件的整个段，剩下依次为每个指定的捕获。
如果正则这么写
	'/(exp1)|(exp2)|(exp3)/'
那么如果匹配到 exp1，match 的长度就是 2；匹配到 exp2，match 的长度就是3。这个特性可以用了识别被哪个 exp 匹配到。
由于正则消耗，一段文字不会被多次匹配，用于字符串提取和分割是不错的手段。