///
title=跳过Python源代码生成PythonVM字节码
post_time=0-18-22
weibo_id=1186373612
code=8ef712fb
///


Python虽然被冠以脚本语言之名，但是事实上在运行的时候也是需要编译步骤的，与python源代码同名的.pyc文件既是编译后的结果，类似Java的字节码。

同时与Java相比，Python的字节码要更加“高级”，因此编译速度是非常之快的，几乎很少感觉到编译消耗的时间，所以往往将编译与执行的步骤合并，不需要手动进行编译。在输入*python xxx.py*的时候，解释器在内存中将源代码解析生成字节码，并直接执行。但编译的过程毕竟还是有开销的，因此直接将字节码保存在文件中作为缓存，方便下次使用，保存的文件就是pyc文件。

因为有.pyc文件的存在，就有可能绕开Python前端编译器来自己生成字节码交给解释器执行，甚至将自己的语言编译为字节码，就像JVM上的Scala、Clojure一样。

## .pyc文件结构

首先要搞明白.pyc文件的内容，其实内容相当简单，文件前四个字节是*Python Magic Number*，不同版本的python可能不同，来保证某个版本的解释器不会意外执行了其他版本解释器生成的字节码。

接下来的四个字节为编译时刻的Unix时间戳。每次对比此时间与源文件最后修改时间，即可知道字节码是否过期。这样python的编译过程对于程序员来说就是完全透明的了，每次*python xxx.py*就可保证执行的是最新的代码，有必要时编译结果会被缓存下来。

剩下的内容看上去不容易解析，其实知道了这些都是marshal模块序列化之后的内容就简单了，直接调用marshal反序列化回去就是。python解释器内部也是用marshal模块将编译结果序列化到文件的。反序列化的结果是一个code对象。

## code对象内容

Python虚拟机是一个Stack Machine（*这里特指CPython，估计Pypy一类的也都是，不过谁知道以后呢……*），每个code对象会对应一个stack。而源代码中的module、function、class、method会生成一个新的code对象，其实就是有自己作用域的对象会生成一个code对象，对应一个stack作为自己的作用域。

下面是一个简单的Python module：

<pre>
    a = 1
    b = 2
    print a + b
</pre>   

对应code对象内容：

<pre>
    ('co_argcount', 0),
    ('co_cellvars', ()),
    ('co_code',
      'd\x00\x00Z\x00\x00d\x01\x00Z\x01\x00e\x00\x00e\x01\x00\x17GHd\x02\x00S'),
    ('co_consts', (1, 2, None)),
    ('co_filename', 'aaa.py'),
    ('co_firstlineno', 1),
    ('co_flags', 64),
    ('co_freevars', ()),
    ('co_lnotab', '\x06\x01\x06\x01'),
    ('co_name', '<module>'),
    ('co_names', ('a', 'b')),
    ('co_nlocals', 0),
    ('co_stacksize', 2),
    ('co_varnames', ())
</pre>

这些就是一个stack运行全部需要的内容了。说清楚code对象的内容篇幅有些长，[这篇文章](http://pycoders-weekly-chinese.readthedocs.org/en/latest/issue7/exploring-python-code-objects.html)有比较详细的讲述，《Python源码剖析》相关章节有更加详细的内容。这里只挑用到的几个介绍一下。

*co_consts*为常量表，保存代码中所有的常量。一段作用域内定义的函数/类会生成一个新的code对象，并保存在这个常量表中以供调用。*co_names*为符号表，保存代码中所有的变量名。*co_stacksize*为栈的大小，需要提前计算出栈最大占用的大小。由此可以大约看出Python虚拟机还是略微有些静态，所有需要的信息需要在编译时获得，栈的大小得提前指定。

其中真正的字节码在*co_code*中，可以用*dis*模块查看对应的指令：

<pre><code>
    1           0 LOAD_CONST               0 (1)
                3 STORE_NAME               0 (a)

    2           6 LOAD_CONST               1 (2)
                9 STORE_NAME               1 (b)

    3          12 LOAD_NAME                0 (a)
               15 LOAD_NAME                1 (b)
               18 BINARY_ADD
               19 PRINT_ITEM
               20 PRINT_NEWLINE
               21 LOAD_CONST               2 (None)
               24 RETURN_VALUE
</code></pre>

指令含义可以参考[这里](http://docs.python.org/2/library/dis.html#python-bytecode-instructions)。另外[这个脚本](https://gist.github.com/3951899)可以更清晰的分析.pyc文件，列出code对象所有的属性以及字节码，并遍历其中包括的所有code对象。

# 简单的S-Expression计算器

根据上面的知识，完全可以写一个自己的编译器来生成.pyc文件交给Python解释器来执行。不过这个目标稍微有些大，还是改为写一个计算器吧。需要做的很简单，就是把类似Lisp语言的S-Expression数学表达式编译为Python字节码。源代码类似这样：

<pre>
    (+ 1
      (- 3
        (+ 1 1)))
</pre>

这种语法形式也叫做*波兰表示法*，类似传统的数学的表达式，只不过把操作符放在了操作数前面。而PythonVM是一个Stack Machine，指令集本质上是一串把操作符放在操作数后面的*逆波兰表示法*，因此需要先转换一下，就可以轻松的编译为Python字节码了。

全部源代码如下，因为懒得写Parser了所以需要计算的的表达式直接用Python的元组表示了：

<script src="https://gist.github.com/3968518.js?file=sexpr2pyc.py"></script>

代码很简单，遇到操作数便入栈，遇到操作符则做相应的计算，这会将栈中最后两个操作数出栈并运算，将计算结果入栈。最后的*PRINT_ITEM*会将栈内的内容打印在屏幕上，*PRINT_NEWLINE*顾名思义。但是Python虚拟机规定所有的code对象最后要有返回结果，我们的计算并不需要返回结果，所以最后要用*RETURN_VALUE*返回一个*None*的结果。

这里只实现了加减两个操作符，乘除类似，很简单就可以加上。另外探索的过程中发现Python在编译时会自动将类似*x = 1 + 2*的代码优化为*x = 3*。

执行*python sexpr2pyc.py*即可在本目录下生成*test.pyc*文件，运行“python test.pyc”，可以得出结果为2。
