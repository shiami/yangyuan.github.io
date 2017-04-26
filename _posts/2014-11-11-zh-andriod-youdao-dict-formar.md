---
layout: post
title: 有道词典 Andriod 版本数据格式分析
---

其实很简单无聊

基于版本 5.3 分析。
其实也简单分析了有道词典iOS版本，必应词典的各个版本，以及金山词典的各个版本，还有那个一直逍遥法外的林格斯词典。
由于在各个平台上的限制，同一词典的不同版本大多都采用了不用的实现方式。

一般 PC 版和 iOS 版本都有一定程度的加密，而 Andriod 版本则比较单纯。可能是 Andriod 硬件千差万别，不敢做额外消耗 CPU 的处理。

基本索引和词典分开
----
这是大多数词典都干了的事情，包括 PC 本地词典。基本索引就是在输入的时候给与下拉提示的部分，一般会给几个备选的单词以及非常精简的释意。
而真正查询某个词的时候，则单独调用其他本地词典，已经网络 API。性能的考虑，很好理解。
这个“基本索引”，在不同词典上实现不一。必应词典使用 sqlite，有道词典使用分割的文本文件。Sqlite 很好理解，有道词典选择本地文件，可能是为了省内存。

iOS 版本上，大家对基本索引处理比较自在。
如有道词典 iOS 版本，这个基本索引是放在两个巨大的数据文件里，加起来有40MB，可想而知为什么有道词典的 iOS 版本为什么比较慢，可能测试机的性能较好，他们无所谓。


有道词典的索引文件
----
有道词典选择本地文件存索引数据，可能是为了省内存。嗯，毕竟现在一些无聊软件，专门给你列内存占用列表。
这方案的确可行，因为现在手机使用的存储并不慢，把数据分割，用哪块取哪块，也不慢，而且内存占用极小。
整个数据大概40MB，有道把他们分成256块，中到英128块，英到中128块。
每块呢有两个文件，一个idx相当于单词列表，一个def，相当于词典。这也很好理解，在idx里找到了词，那么在def里，对于的位置就能找到具体数据。此外还有一个块的索引，列出了每个文件的第一个单词。

从需求的角度，当你输入一个东西，需要给与提示。需要使用 1 到 2 个块（因为输入的内容可能刚好在块的边缘），那么从数学角度，占用内存大概是 400K，这比40MB 还是差很远的。而实际上，英中词典并不大，一般仅有 60000-100000，这样的数据量就算线性处理都是非常快的，如果加上简单的索引，完全没有必要调用第三方数据库。

IDX 和 DEF 格式
----
IDX 格式和 DEF 格式也是非常简单。是一种在流处理里很常见的方式：{长度+字符串} 的数组。大端16位数字代表长度，后面跟着的是与这个长度相符的字符串。
IDX 文件里，这个字符串就是单词，而在 DEF 里，这是个 json 格式。
以 DEF 为例，可以使用PHP这样读取

[!code=php]
	$count = 0;
	for ($i=0; $i<128; $i++) {
		$file = fopen('e2c_'.$i.'.def', 'r');
		while($head = fread($file, 2)) {
			// 之前做过预判，没有异常数据
			$size = unpack('n', $head)[1];
			$string = fread($file, $size);
			$dict = json_decode($string, TRUE);
			
			// 这个 dict 就是单个词典的数据
			$word = $dict['word'][0]['return-phrase']['l']['i'];
			// 单词
			echo $word."\r\n";
			foreach ($dict['word'][0]['trs'] as $tr) {
				$tr = $tr['tr'][0]['l']['i'][0];
				// 解释（可能有多个）
				echo $tr."\r\n";
			}
		}
		fclose($file);
	}

额外地，这些 JSON 数据冗余严重，简单的处理完全可以减少到 30MB。