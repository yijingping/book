# 基本环境
## 虚拟机
Python 是一种半编译半解释型运行环境。首先，它会在模块“载入” 时将源码编译成字节码(Byte Code)。
而后，这些字节码会被虚拟机在一个“巨大” 的核心函数里解释执行。这是导致Python性能较低的重要原因，好在现在有了内置Just-in-time二次编译器的[PyPy](http://pypy.org)可供选择。

当虚拟机开始运行时，它通过初始化函数完成整个运行环境设置：

* 创建解释器和主线程状态对象，这是整个进程的根对象。
* 初始化内置类型。数字、列表等类型都有专门的缓存策略需要处理。
* 创建 `__builtins__` 模块，该模块持有所有内置类型和函数。
* 创建 `sys` 模块，其中包含了 `sys.path`、`modules` 等重要的运行行期信息。
* 初始化 `import` 机制。
* 初始化内置 `Exception`。
* 创建 `__main__` 模块，准备运行行所需的名字空间。
* 通过 `site.py` 将 `site-packages` 中的第三方方扩展库添加到搜索路径列表。
* 执行入口 py 文件。执行前会将 `__main__.__dict__` 作为名字空间传递进去。
* 程序执行结束。
* 执行行清理操作，包括调用退出函数，GC 清理现场，释放所有模块等。
* 终止进程。

## 名字空间
可以看出,名字空间就是一个字典(dict)，C 变量名是内存地址别名。

    >>> x = 123
    >>> globals()  # 获取 module 名字空间。
    {'x': 123, ......}

使用用名字空间管理上下文文对象，带来无无与伦比比的灵活性，但也牺牲了执行行性能。
毕竟从字典中查找对象远比比指针低效很多，各有得失。

locals() 和 globals()

## 类型和对象
Python 中的一切都是对象,包括类型在内的每个对象都包含一个标准头，通过头部信息就可以明确知道其具体类型。

    define PyObject_HEAD \
        Py_ssize_t ob_refcnt; \         // 引用计数 
        struct _typeobject *ob_type;    // 类型

    typedef struct _object {            // object对象
        PyObject_HEAD
    } PyObject;

    typedef struct {                    // int对象
        PyObject_HEAD   // 在 64 位平台上,头长度为 16 字节。
        long ob_ival;   // long 是 8 字节。
    } PyIntObject;



## 内存管理
为了提升性能，python从系统申请大块内存(arena)，

## 引用传递
对象总是按引用传递，简单点说就是通过复制指针来实现多个名字指向同一对象。
因为 arena 也是在堆上分配的,所以无无论何种类型何种大大小小的对象，都存储在堆上。
Python 没有值类型和引用用类型一说，就算是最简单的整数也是拥有标准头的完整对象。

如果不想改变对象，使用：

* 不可变对象：int，long，str，tuple，frozenset
* copy.deepcopy
* 序列化对象，如pickle，cpickle，marshal

## 引用计数
Python 默认采用用引用计数来管理对象的内存回收。当引用用计数为 0 时,将立即回收该对象内存（并且调用`__del__`）。

某些内置类型，比如小小整数，因为缓存的缘故，计数永远不会为 0，直到进程结束才由虚拟机清理函数释放。

弱引用不增加引用用计数。但不是所有类型都支持弱引用，比如 list、dict，弱引用会引发异常。

## 垃圾回收

### 回收机制
Python同时使用2套垃圾回收机制：

* 引用计数
* 处理循环引用的GC

能引发循环引用用问题的，都是那种容器类对象，比如 list、set、object 等。
对于这类对象,虚拟机在为其分配内存时，会额外添加用用于追踪的 PyGC_Head。
这些对象被添加到特殊链表里，以便 GC 进行行管理。

### 回收等级
同 .NET、JAVA 一样，Python GC 将要回收的对象分成 3 级代龄。

### 处理循环引用
当一个对象显式定义了__del__方法，而且里面有循环引用，Python不会自动回收这个对象。 如果这种情况没有正确处理，会造成内存泄漏。
解决的办法是在__del__中手动解除循环引用，或者干脆避免这种有循环引用的写法。

## 编译
Python 实现了 Stack-Based VM 架构，通过字节码实现跨平台。

字节码文件: pyc 或 pyo

编译发生在模块载入那一刻。分为两种情况。

有pyc文件的流程：

* 核对文件 Magic 标记。
* 检查时间戳和源码文文件修改时间是否相同,以确定是否需要重新编译。
* 载入模块。

没有pyc但有py：

* 对源码进行 AST 分析。
* 将分析结果编译成 PyCodeObject。
* 将 Magic、源码文件修改时间、PyCodeObject 保存到 pyc 文件中。
* 载入模块。

Magic 是一个特殊的数字，由 Python 版本号计算得来， 作为 pyc 文件和 Python 版本检查标记。

## 执行
相比 .NET、JAVA 的 CodeDOM 和 Emit，Python 天生拥有无与伦比的动态执行优势。

使用eval/exec/execfile/runpy

# 内置类型
Python内置数据类型：

* 空值：None
* 数字：bool，int，float，float，complex
* 序列：str, unicode, list，tuple
* 字典：dict
* 集合：set，frezonset

## 数字
__bool__

    # 注意 
    bool([]) == False
    bool([[]]) == True 

    # bool可以直接当数字使用
    int(True) == 1
    int(False) == 0 
    range(10)[True] == 1

__int__
int在32位平台是32位，64位平台是64位（`sys.maxint`）。

整数是虚拟机特殊照顾对象：

* 从堆上按需申请名为 PyIntBlock 的缓存区域存储整数对象。

* 使用用固定数组缓存 [-5, 257) 之间的小小数字,只需计算下标就能获得指针。

* PyIntBlock 内存不会返还给操作系统,直至至进程结束。（防止内存泄漏）

__long__

当超出int限制，会自动转换成long。

    >>> sys.maxint + 1
    2147483648L

long是变长对象，只要内存足够，可以存储天文数字

    >>> 1 << 3000
    12302319221611....890612250135171889174899079911291512399773872178519018229989376L

__float__
双精度浮点数。（编程语言普遍存在这样的问题）不能用“==” 判断相等，“四舍五入”结果也不确定。

    >>> 3 / 2  # 除法默认返回整数，在 Python 3 中返回浮点数。
    1
    >>> float(3) / 2
    1.5

    >>> 3 * 0.1 == 0.3  # 这个容易导致莫名其妙的错误。
    False

如果需要精确计算，可用 Decimal 代替，它能精确控制运算精度、有效数位和 round 的结果。

## 字符串
与字符串相关的问题总是很多，比比如池化(intern)、编码 (encode) 等。
字符串是不可变类型，保存字符序列或二二进制数据。

* 短字符串存储在 arena 区域，str、unicode 单字符会被永久缓存。
* str 没有缓存机制，unicode 则保留 1024 个宽字符长度小于 9 的复用内存块。
* 内部包含 hash 值，str 另有标记用来判断是否被池化。

### 表示
建议用单引号（'）表示字符，用双引号（"）表示字符串，用三双引号（"""）或者多个双引号（"" ""）表示长字符串。

    >>> r"abc\x"    # r 前缀定义非非转义的 raw-string。
    'abc\\x'
    >>> "a" "b" "c" # 自自动合并多个相邻字符串。
    'abc'

### 编码
在shell中，默认是系统编码

    >>> "中国人人"  # UTF-8 字符串 (*nux 系统默认)。
    '\xe4\xb8\xad\xe5\x9b\xbd\xe4\xba\xba'
    >>> type(s), len(s)
    <type 'str'>, 9
    >>> u"中国人人"  # 使用用 u 前缀定义 UNICODE 字符串。
    u'\u4e2d\u56fd\u4eba'
    >>> type(u), len(u)
    <type 'unicode'>, 3

在文件中，Python 2.x 默认使用ascii编码，如果要使用其他编码，需要指定

    #!/usr/bin/python           # 如果是脚本，将这一行放在开头 
    # -*- coding: utf-8 -*-     # 必须在第1或2行。
                                # 这是文件编码，如果没有这一行，而文件中含有Non-Ascii字符，将不能被编译
    # from __future__ import unicode_literals     # 如果在开头加上这一行，
                                                  # 文件中的字符串变量被解析成unicode，默认是utf-8

    import sys                     
    print sys.getdefaultencoding() 
    s = "中国人"                   
    print type(s), s, len(s)       
                                   
    s = s.encode("utf-8")          
    print type(s), s, len(s)       
                                   
    s = "中国人"                   
    s = s.encode("gb2312")         
    print type(s), s, len(s)       

执行上面的脚本：

    ascii
    <type 'unicode'> 中国人 3
    <type 'str'> 中国人 9
    <type 'str'> �й��� 6

unicode只是抽象编码，需要转化成其他编码才能存储，utf-8是3字节/汉字，gb2312是2字节/汉字

str、unicode 都提供了 encode 和 decode 编码转换方方法。

* encode：将默认编码转换为其他编码。
* decode：将默认或者指定编码字符串转换为 unicode。

### 格式化
Python 提供了3种字符串格式化方法。

* C 样式

    标记：- 左对齐，+ 数字符号，# 进制前缀，或者用空格、0 填充。

        %[(key)][flags][width][.precision]typecode

* 更强大的 format
* string.Template 模板

### 池化
在 Python 进程中，无数的对象拥有一堆类似 `__name__`、`__doc__` 这样的名字，池化有助于减少对象数量和内存消耗， 提升性能。

用 intern() 函数可以把运行行期动态生生成的字符串池化。当池化的字符串不再有引用时，将被回收。

## 列表
从功能上看，列表 (list) 类似 Vector，而非非数组或链表。

* 列表对象和存储元素指针的数组是分开的两块内存，后者在堆上分配。
* 虚拟机会保留 80 个列表复用对象，但其元素指针数组会被释放。
* 列表会动态调整指针数组大小，预分配内存多于实际元素数量。

使用sort对list排序，如果每次插入都要排序，使用bisect效率更高

__性能__

列表用用 realloc() 调整指针数组内存大小，可能需要复制数据。插入和删除操作，还会循环移动后续元素。这些都是潜在的性能隐患。对于频繁增删元素的大型列表，应该考虑用链表等数据结构代替。

某些时候,可以考虑用数组(array.array)代替列表。

## 元组
元组 (tuple) 看上去像列表的只读版本，但在底层实现上有很多不同之处。

* 只读对象，元组和元素指针数组内存是一次性连续分配的。
* 虚拟机缓存 n 个元素数量小于 20 的元组复用对象。

在编码中，应该尽可能用元组代替列表。除内存复用更高效外，其只读特征更利于并行开发。

__namedtuple__

使用namedtuple定义常量

    from collections import namedtuple                              
    def _enum(**enums):                                             
        configtuple = namedtuple('Configure',map(str,enums.keys())) 
        return configtuple(*enums.values())                         
                                                                        
    SCORE = _enum(                                                  
            MAX_SCORE = 100,
            MIN_SCORE = 0
     )

    s = SCORE.MAX_SCORE

## 字典
字典 (dict) 采用用开放地址法的哈希表实现。

* 自带元素容量为 8 的 smalltable，只有 "超出" 时才到堆上额外分配元素表内存。
* 虚拟机缓存 80 个字典复用用对象，但在堆上分配的元素表内存会被释放。
* 按需动态调整容量。扩容或收缩操作都将重新分配内存，重新哈希。
* 删除元素操作不会立即收缩内存。

对于大字典，调用keys()、values()、items() 会构造同样巨大的列表。建议用迭代器替代，以减少内存开销。

### 视图
使用 `v = d.viewitems()`获取试图，操作类似与set，视图会与dict同步改变。

### defautdict

    >>> from collections import defaultdict
    >>> d = defaultdict(list) # key "a" 不存在,直接用用 list() 函数创建一一个空列表作为 value。
    >>> d["a"].append(1)
    >>> d["a"].append(2)
    >>> d["a"]
    [1, 2]


### OrderdDict
有序字典

## 集合
除了hash不等，值也不能相同。

    判重公式:(a is b) or (hash(a) == hash(b) and eq(a, b))

在内部实现上，set和dict非常相似，除了 Entry 没有 value 字段。

set支持集合运算

    == 是否相等
    != 是否不等 
    >= 超集判断(issuperset)
    <= 子集判断(issubset)
    |  并集(union)
    &  交集(intersection)
    -  差集(difference)
    ^  对称差集(symmetric_difference)
    isdisjoint  判断是否没有交集

集合和字典主键都必须是可哈希类型对象，但常用的 list、dict、set、defaultdict、OrderedDict都是不可哈希的，仅有 tuple、frozenset 可用用。

如果想把自定义类型放入集合，需要保证 hash 和 equal 的结果都相同才能去重。

# 表达式
## 类型转换

    >>> str(123), int('123')                             # int 
    >>> bin(17), int('0b10001', 2)
    >>> oct(20), int('024', 8)
    >>> hex(22), int('0x16', 16)
    >>> str(0.9), float("0.9")                           # float
    >>> ord('a'), chr(97), unichr(97)                    # char
    >>> str([0, 1, 2]), eval("[0, 1, 2]")                # list
    >>> str((0, 1, 2)), eval("(0, 1, 2)")                # tuple
    >>> str({"a":1, "b":2}), eval("{'a': 1, 'b': 2}")    # dict
    >>> str({1, 2, 3}), eval("{1, 2, 3}")                # set


# 函数
当编译器遇到 def，会生成创建函数对象指令。也就是说 def 是执行行指令，而而不仅仅是个语法关键字。可以在任何地方动态创建函数对象。
一个完整的函数对象由函数和代码两部分组成。其中，PyCodeObject 包含了字节码等执行数据，而 PyFunctionObject 则为其提供了状态信息。

函数声明:

    def name([arg,... arg = value,... *arg, **kwarg]):
        suite

结构定义:

    typedef struct {
        PyObject_HEAD
        PyObject *func_code;        // PyCodeObject
        PyObject *func_globals;     // 所在模块的全局名字空间
        PyObject *func_defaults;    // 参数默认值列表
        PyObject *func_closure;     // 闭包列表
        PyObject *func_doc;         // __doc__
        PyObject *func_name;        // __name__
        PyObject *func_dict;        // __dict__
        PyObject *func_weakreflist; // 弱引用用链表
        PyObject *func_module;      // 所在 Module
    } PyFunctionObject;

lambda 可以创建匿名函数

## 参数
\* 展开序列或者字典键值，\** 展开字典，所以可以使用序列和字典构造变参

    >>> d = {'a': 1, 'b': 2}
    >>> def f(*args, **kwargs):
    ...     print args
    ...     print kwargs
    ...     

    >>> f(d)
    ({'a': 1, 'b': 2},)
    {}

    >>> f(*d)
    ('a', 'b')
    {}

    >>> f(**d)
    ()
    {'a': 1, 'b': 2}


__小心默认形参__

默认值对象在创建函数时生成，所有调用都使用同一对象。如果该默认值是可变类型，那么就如同 C 静态局部变量。

    >>> def test(x, ints = []):
    ... ints.append(x)
    ... return ints
    >>> test(1)
    [1]
    >>> test(2)     # 保持了上次调用用状态。
    [1, 2]
    >>> test(1, []) # 显式提供实参，不使用默认值。
    [1]
    >>> test(3)     # 再次使用用默认值。
    [1, 2, 3]

## 作用域
函数形参和内部变量都存储在 locals 名字空间中。

    >>> def test(a, *args, **kwargs):
    ... s = "Hello, World!"
    ... print locals()
    >>> test(1, "a", "b", x = 10, y = "hi")
    {
        'a': 1,
        'args': ('a', 'b'),
        'kwargs': {'y': 'hi', 'x': 10}
        's': 'Hello, World!',
    }

除非使用 global、nonlocal 特别声明，否则在函数内部使用赋值语句，总是在 locals 名字空间中新建一个对象关联。注意:"赋值" 是指名字指向新的对象，而非通过名字改变对象状态。

如果仅仅是引用用外部变量，那么按 LEGB 顺序在不同作用用域查找该名字。

    名字查找顺序: locals -> enclosing function -> globals -> __builtins__

* locals: 函数内部名字空间，包括局部变量和形参。
* enclosing function: 外部嵌套函数的名字空间。
* globals: 函数定义所在模块的名字空间。
* \__builtins\__: 所有模块载入入时都持有该模块，其中包含了内置类型和函数。

想想看，如果将对象引入` __builtins__` 名字空间，那么就可以在任何模块中直接访问，如同内置函数那样。不过鉴于 `__builtins__` 的特殊性，这似乎不是个好主意。

    >>> __builtins__.b = "builtins"
    >>> g = "globals"
    >>> def enclose():
    ...     e = "enclosing"
    ...     def test():
    ...         l = "locals"
    ...         print l
    ...         print e
    ...         print g
    ...         print b
    ...    
    ...     return test
    >>> t = enclose()
    >>> t()
    locals
    enclosing
    globals
    builtins

现在，获取外部空间的名字没问题了，但如果想将外部名字关联到一个新对象，就需要使用用 global 关键字，指明要修改的是globals 名字空间。Python 3 还提供了 nonlocal 关键字，用来修改外部嵌套函数名字空间，可惜 2.7 没有。

    >>> x = 100
    >>> hex(id(x))
    0x7f9a9264a028
    >>> def test():
    ...     global x, y         # 声明 x, y 是 globals 名字空间中的。
    ...     x = 1000            # globals()["x"] = 1000
    ...     y = "Hello, World!" # globals()["y"] = "..."。 新建名字。
    ...     print hex(id(x)) 
    >>> test()
    0x7f9a9264a028              # 可以看到 test.x 引用用的是外部变量 x。
    >>> x, y                    # globals 名字空间中出现了 y。
    (1000, 'Hello, World!')
 
## 闭包
闭包是指:当函数离开创建环境后，依然持有其上下文文状态。注意闭包有"延迟获取"现象。

## 堆栈帧
Python 堆栈帧基本上就是对 x86 的模拟，用指针对应 BP、SP、IP 寄存器。堆栈帧成员包括函数执行所需的名字空间、调用堆栈链表、异常状态等。

    typedef struct _frame {
        PyObject_VAR_HEAD
        struct _frame *f_back;   // 调用用堆栈 (Call Stack) 链表
        PyCodeObject *f_code;    // PyCodeObject
        PyObject *f_builtins;    // builtins 名字空间
        PyObject *f_globals;     // globals 名字空间
        PyObject *f_locals;      // locals 名字空间
        PyObject **f_valuestack; // 和 f_stacktop 共同维护运行行帧空间，相当于 BP 寄存器。
        PyObject **f_stacktop;   // 运行行栈顶，相当于 SP 寄存器的作用用。
        PyObject *f_trace;       // Trace function
        PyObject *f_exc_type, *f_exc_value, *f_exc_traceback; // 记录当前栈帧的异常信息
        PyThreadState *f_tstate; // 所在线程状态
        int f_lasti;             // 上一一条字节码指令在 f_code 中的偏移量，类似 IP 寄存器。
        int f_lineno;           // 与当前字节码指令对应的源码行行号
        ... ...
        PyObject *f_localsplus[1]; ! // 动态申请的一一段内存，用用来模拟 x86 堆栈帧所在内存段。
    } PyFrameObject;

可使用用 `sys._getframe(0)` 或 `inspect.currentframe()` 获取当前堆栈帧。其中 _getframe() 深度参数为 0 表示示当前函数，1 表示示调用用堆栈的上个函数。除用于调试外，还可利用堆栈帧做些有意思的事情。

`sys._current_frames` 返回所有线程的当前堆栈帧对象。虚拟机会缓存 200 个堆栈帧复用对象，以获得更好的执行性能。整个程序跑下来，天知道要创建多少个这类对象。

## 包装
用functools.partial() 可以将函数包装成更简洁的版本。

## 常用函数
__print__

Python 2.7 可使用用 print 表达式，Python 3 就只能用用函数了。

用标准库中的 pprint.pprint() 代替 print，能看到更漂亮的输出结果。

__input__

input 会将输入入的字符串进行行 eval 处理，raw_input 直接返回用用户输入入的原始字符串。

Python 3 已经将 raw_input 重命名为 input。

用标准库 getpass 输入入密码。

__exit__

exit([status]) 调用用所有退出函数后终止止进程，并返回 ExitCode。

* 忽略或 status = None，表示示正常退出， ExitCode = 0。
* status = <number>，表示示 ExiCode = <number>。
* 返回非非数字对象表示示失败，参数会被显示示， ExitCode = 1。

sys.exit() 和 exit() 完全相同。os_exit() 直接终止进程，不调用退出函数，且退出码必须是数字。

__vars__

获取 locals 或指定对象的名字空间。

    >>> vars() is locals()
    True
    >>> import sys
    >>> vars(sys) is sys.__dict__
    True

__dir__

获取 locals 名字空间中的所有名字，或指定对象所有可访问成员 (包括基类)。

    >>> set(locals().keys()) == set(dir())
    True

# 迭代器
在 Python 文文档中，实现接口通常被称为遵守协议。因为 "弱类型" 和 "Duck Type" 的缘故，很多静态语言中繁复的模式被悄悄抹平。
## iter
迭代器对象要求支持迭代器协议，所谓支持迭代器协议就是对象包含__iter__()和next()方法。其中__iter__()方法返回迭代器对象自己；next()方法返回下一个前进到下一个结果，在结尾时引发StopIteration异常。

列表不是迭代器对象，但是列表通过__iter__()可以得到一个迭代器对象来遍历整个列表的内容，像列表这样的序列对象都属于这种情况；与序列不同，文件对象本身就是一种迭代器对象。

    class Reverse:
        """Iterator for looping over a sequence backwards."""
        def __init__(self, data):
            self.data = data
            self.index = len(data)
        def __iter__(self):
            return self
        def next(self):
            if self.index == 0:
                raise StopIteration
            self.index = self.index - 1
            return self.data[self.index]


## generator 
生成器使python可以很容易的支持迭代协议。生成器通过生成器函数产生，生成器函数可以通过常规的def语句来定义，但是不用return返回，而是用yeild一次返回一个结果，在每个结果之间挂起和继续它们的状态，来自动实现迭代协议。

    def reverse(data):
        for index in range(len(data)-1, -1, -1):
            yield data[index]

__TIPS__:

像列表这种序列类型的对象，我们可以通过iter()来产生多个迭代器，在迭代的过程中各个迭代器相互对立；但是迭代器对象没法通过iter()方法来产生多个不同的迭代器，它们都指向了自身，所以没法独立使用。

## 协程
当今一些具备协程语义的语言，比较重量级的如C#、erlang、golang，以及轻量级的python、lua、javascript、ruby，还有函数式的scala、scheme等。相比之下，作为原生态语言的 C 反而处于尴尬的地位，原因在于 C 依赖于一种叫做栈帧的例程调用，例程内部的状态量和返回值都保留在堆栈上，这意味着生产者和消费者相互之间无法实现平级调用。
如果将每个协程的上下文（比如程序计数器）保存在其它地方而不是堆栈上，协程之间相互调用时，被调用的协程只要从堆栈以外的地方恢复上次出让点之前的上下文即可，这有点类似于 CPU 的上下文切换，遗憾的是似乎只有更底层的汇编语言才能做到这一点。


## 标准库itertools
ifilter/imap/izip等等

# 模块
Python 模块是运行期对象，模块名对应源码文件。
## 重要属性

* `__name__`：模块名<package>.<module>，在sys.modules中以此为主键。
* `__file__`：文件名，不带路径。
* `__dict__`：模块globals名字空间。

使用sys.reload()重新导入模块。

## 搜索路径
虚拟机按以下顺序搜索模块（包）：

* 当前进程根目录。
* PTTHONPATH环境变量指定的路径列表。
* Python标准库目录列表。
* 路径文件（.pth）保存的目录（通常放在site-packages目录下）。

进程启动后，所有这些路径都被组织到sys.path列表中（顺序可能会被修改）。任何import操作都会按照sys.path列表查找目标模块。可以在运行时往sys.path添加自定义路径。

虚拟机按以下顺序匹配目标模块：

* py源码文件
* pyc字节码文件
* egg包文件或目录
* so、dll、pyd等扩展文件。
* 内置模块。
* 其他。

## 导入模块
进程中的模块对象是唯一的。在首次导入后，被添加到sys.modules，后面导入的会检查是否已经存在，存在则不重复导入。可用`sys.modules[__name__]`获取当前模块对象。

模块内使用import，导入到globals，函数内使用则导入到函数的locals。

`__all__`
为防止`import *`导入不必要的成员，使用`__all__`指定可导出的成员。

    __all__ = ["add", "x"]

私有成员和`__all__`都不会显示到出。但是Python还是可以调用到（约定大于配置）。

`__import__`

`imp.load_souce` 和 `reload()`

## 构建包
多个模块文件放到一个目录，并提供__init__.py，就形成了包。

导入包/包中的模块/导入成员 都会执行`__init__.py`，且只执行一次。可用来初始化包环境，初始化包环境，存储帮助、版本等信息。

### \__all__

建议不要使用`import *`，换种方式，将要公开的模块和模块成员显示导入到`__init__.py`中，调用者使用：

    import <package>
    value = <package>.<member>

### \__path__

包内文件太多，需要分到多个目录中，但有不想拆分成新的包或者子包。只要在 `__init__.py` 中用 `__path__`指定所有子目录全路径即可。
子目录中不需要`__init__.py`文件。
    
    from os.path import abspath, join
    subdirs = lambda *dirs: [abspath(join(__path__[0], sub)) for sub in dirs]
    __path__ = subdirs("a", "b")

### pkgutil

使用pkgutil，而不是os.listdir()获取所有模块类表。

iter_modules()遍历指定目录，walk_packages()遍历制定目录及其子目录。

pkguitl.get_data()可读取包内任何文件内容

### egg

将包压缩成单个文件，以便于分发和安装。类似Java JAR。

    ## 创建目录和setup.py文件

    from setuptools import setup, find_packages
    setup (
            name = "test",
            version = "0.0.9",
            keywords = ("test", ),
            description = "test package",
            url = "http://github.com/qyuhen",
            author = 'Q.yuhen',
            author_email = "qyuhen@hotmail.com",
            packages = find_packages(),
          )

    ## 创建egg压缩文件，test-0.0.9-py2.7.egg 文件保存在dist目录中
    $ python setup.py bdist_egg

    ## 将test-0.0.9-py2.7.egg全路径添加到.pth或PYTHONPATH环境变量就可使用。更常见的做法是将其安装到site_packages目录
    $ sudo easy_install dist/test-0.0.9-py2.7.egg

    ## 安装后的搜索路径被自动添加到site-packages/easy-install.pth中
    $ cat site-packages/easy-install.pth
    import sys; sys.__plen = len(sys.path)
    ./setuptools-0.6c11-py2.7.egg
    ./pip-1.2.1-py2.7.egg
    ./beanstalkc-0.3.0-py2.7.egg 
    ./PyYAML-3.10-py2.7-linux-x86_64.egg
    ./redis-2.8.0-py2.7.egg
    import sys; new=sys.path[sys.__plen:]; del sys.path[sys.__plen:]; p=getattr(sys,'__egginsert',0); sys.path[p:p]=new; sys.__egginsert = p+len(new)

### pip 与 easy_install
TODO
### egg 与egg-info
TODO

# 类
Python 2.x 存在两种类：Classic Class 和 New-Style Class。
Python 3中仅保留了后一种，和type是一样的。所以即使在2.x中，也应显示的从object继承。

    >>> class User: pass
    >>> type(User)  # 2.x 
    Classic Class

    >>> class User(object): pass
    >>> type(User)  # 2.x 
    type 

## 名字空间
如同def，class关键字起着创建类型对象的作用。
类型对象很特殊，在整个进程中是单例的，不被回收。

    typedef struct
    {
        PyObject_HEAD
        PyObject *cl_bases;    /* A tuple of class objects */
        PyObject *cl_dict;    /* A dictionary */
        PyObject *cl_name;    /* A string */
        PyObject *cl_getattr;
        PyObject *cl_setattr;
        PyObject *cl_delattr;
    } PyClassObject;

类型（class）存储了所有的静态成员和方法，实例（instance）仅存储实例字段（所有父类的实例字段）

    typedef struct
    {
        PyObject_HEAD
        PyClassObject *in_class;    /* The class object */
        PyObject  *in_dict;         /* A dictionary */
        PyObject  *in_weakreflist;  /* List of weak references */
    } PyInstanceObject;

类型和实例各自拥有各自的名字空间。

    >>> class S(object):
    ...     a = 1
    ...     def f():pass
    >>> S.__dict__
    <dictproxy {'__dict__': <attribute '__dict__' of 'S' objects>,
    '__doc__': None,
    '__module__': '__main__',
    '__weakref__': <attribute '__weakref__' of 'S' objects>,
    'a': 1,
    'f': <function __main__.f>}>

    >>> a = S()
    >>> a.__dict__
    {}

访问对象成员不在locals和globals中查找，而是名字空间。查找顺序如下：

    instance.__dict__ -> class.__dict__ -> baseclass.__dict__

## 字段
字段（field）和属性（property）是不同的。

* instance的字段在`instance.__dict__`中
* class的字段在`class.__dict__`中
* 双下划线的方法和成员，视为私有，会被重命名：`_<class>__<name>`

## 属性
属性（property）是由getter、setter、deleter几个方法构成的逻辑。
与field不同的是：

* property可以返回动态运算值。
* 在`__dict__`中显示的是`<property at 0xa83c5a4>}>`

在查找成员的时候，（即使是父类的）属性比字段优先。 我们因该尽量使用属性，以免内部字段曝光。

## 方法
所有的方法都存储在`class.__dict__`中，所以不支持方法重载（Overload）。

方法与函数最大的区别就是，方法有self这个参数。

    >>> class User(object):
    ...  def print_id(self):
    ...     print hex(id(self))

    >>> u = User()
    >>> u.print_id
    <bound method User.print_id of <__main__.User object at 0x10cf58b50>>

    >>> u.print_id()
    0x10cf58b50

    >>> User.print_id
    <unbound method User.print_id>

    >>> User.print_id(u)
    0x10cf58b50

从上面可以看出实例方法的特殊性。当instance调用的时候，是bound method，动态绑定到调用的当前instance；当class调用的时候，是unbound method，必须显示的传递instance。

### 静态方法
staticmethod和classmethod

    >>> class User(object):
    ...  def a(): pass
    ...
    ... @staticmethod
    ... def b(): pass
    ...
    ... @classmethod
    ... def c(cls): pass

    >>> User.a
    <unbound method User.a>
    >>> User.b
    <function b at 0x10c8ef320>
    >>> User.c
    <bound method type.c of <class '__main__.User'>>

    >>> User.a()
    TypeError: unbound method a() must be called with User instance as first argument (got nothing instead)

### 特殊方法

* `__new__`：创建instance
* `__init__`：初始化instance
* `__del__`：instace回收前被调用

`__new__`可以返回任意类型，一般返回self。

### 继承（inherit）
所有基类的实例字段都存储在`instance.__dict__`中，其他则各归各家。

直接基类存储在`__bases__`中，直接派生类存储在`__subclasses__`中

__多重继承__：

多重继承的成员和方法搜索顺序：mro（method resolution order）。也就是：

    从下到上，从左到右。

mro中没有instance

    >>> C.__mro__
    (<class '__main__.C'>, <class '__main__.A'>, <class '__main__.B'>, <type 'object'>)

__super()__：

super() 依照mro顺序搜索基类成员。

但多重继承的时候，super按照mro顺序查找的，因此会出问题，建议改用“组合模板”实现类似功能。

不建议用`self.__class__`代替当前的类名，因为在继承的时候会引发混乱。

## 抽象类（Abstract Class）

抽象类无法实例化

## 开放类（Open Class）
Open Class几乎是所有动态语言的标配，也是精华所在。
即使是运行期，也可以随意改动对象，增加或删除属性和方法。

* 普通方法，应该绑定到class，而不是instance。否则只是普通方法，而不是bound method。
* 静态方法，应该使用staticmethod、classmethod包装一下，否则只会成为上面的普通方法。
* 使用setattr、getattr、delattr，直接操作instance和class的名字空间。

`__slots__`：

`__slots__`会阻止虚拟机创建`instance.__dict__`，仅为`__slots__`名单中的成员分配内存空间，运行期间也不以增加新的成员。
这有助于减少内存占用，提高运行效率，尤其是大量使用该对象的时候。

    >>> class User(object):
    ...     __slots__ = ("_name", "_age")
    ...
    ...     def __init__(self, name, age):
    ...         self._name = name
    ...         self._age = age
    >>> u = User("Tom", 34)
    >>> hasattr(u, "__dict__")
    False
    >>> u.title = "CXO" # 动态增加字段失败
    AttributeError: 'User' object has no attribute 'title'
    >>> del u._age          # 已有的可以删除
    >>> u._age = 18          # 坑是可以补回来的
    >>> u._age
    18
    >>> del u._age
    >>> u._title = "CXO"
    AttributeError: 'User' object has no attribute '_title'
    >>> vars(u)     # 因为没有__dict__，vars失败
    TypeError: vars() argument must have __dict__ attribute

虽然没有`__dict__`，但依然可以用dir()和inspect.getmember()获取实例成员信息。

派生类必须用`__slots__`为新增的字段分配存储空间（即使`__slots__ = []`），否则会更慢。

## 操作符重载
### \__setitem__ 
索引器，像序列和字典一样操作对象。

    >>> class A(object):
    ...     def __init__(self, **kwargs):
    ...         self._data = kwargs
    ...
    ...     def __getitem__(self, key):
    ...         return self._data.get(key)
    ...
    ...     def __setitem__(self, key, value):
    ...         self._data[key] = value
    ...
    ...     def __delitem__(self, key):
    ...         self._data.pop(key, None)
    ...
    ...     def __contains__(self, key):
    ...         return key in self._data.keys()
    >>> a = A(x = 1, y = 2)
    >>> a["x"]
    1
    >>> a["z"] = 3
    >>> "z" in a
    True
    >>> del a["y"]
    >>> a._data
    {'x': 1, 'z': 3}

### \__call__ 
把instance当function使，也就是传说中的callable。

    >>> class A(object):
    ...     def __call__(self, *args, **kwargs):
    ...         print hex(id(self)), args, kwargs
    >>> a = A()
    >>> a(1, 2, s = "hi")    # 完全把instance，伪装成函数接口
    0x10c8957d0 (1, 2) {'s': 'hi'}

### \__dir__
返回所有内部成员或方法（包括私有的）

### \__getattr__ 
先看看这几个方法的触发时机：

* `__getattr__`：访问不存在的成员。
* `__setattr__`：对任何成员的赋值操作。
* `__delattr__`：删除成员操作。
* `__getattribute__`：访问任何存在或不存在的成员，包括`__dict__`。

不要在这几个方法里直接访问对象成员，也不要使用hasattr/getattr/setattr/delattr函数，因为它们会被再次拦截，形成无限循环。正确的做法是直接操作dict。

    >>> class A(object):
    ...   def __init__(self, x):
    ...       self.x = x          # 会被__setattr__捕获
    ...
    ...   def __getattr__(self, name):
    ...       print "get:", name
    ...       return self.__dict__.get(name)
    ...
    ...  def __setattr__(self, name, value):
    ...       print "set:", name, value
    ...       self.__dict__[name] = value
    ...
    ...   def __delattr__(self, name):
    ...       print "del:", name
    ...       self.__dict__.pop(name, None)
    ...
    ...   def __getattribute__(self, name):
    ...       print "attribute:", name
    ...       return object.__getattribute__(self, name)

    
    >>> a = A(10)           # __init__ 里面的self.x = x 被 __setattr__ 捕获
    set: x 10
    attribute: __dict__
    
    >>> a.x                 # 访问已存在字段，仅被 __getattribute__ 捕获
    attribute: x
    10
    
    >>> a.y = 20            # 创建新的字段，被 __setattr__ 捕获
    set: y 20
    attribute: __dict__
    
    >>> a.z                 # 访问不存在字段，被 __getattr__ 和 __getattribute__ 捕获
    attribute: z
    get: z
    attribute: __dict__
    
    >>> del a.y             # 删除字段，被 __delattr__ 捕获
    del: y
    attribute: __dict__

### \__cmp__
`__cmp__`通过返回数字来判断大小，而`__eq__`仅用于相等判断。

### 其他

`__repr__`、`unicode__`

`__lt__`、`__le__`、`__gt__`、`__ge__`、`__ne__`

`__add__`、`__radd__`、`__mul__`、`__imul__`

`__getslice__`、`__setslice__`

要了解更多的方法，最好去看Python源码。

# 异常
异常不仅仅是错误，也是正常的跳转逻辑。

制造异常最快的方式是`1/0`，也可以用这种方法来调试，以查看堆栈信息。

容易犯错的写法：

    try:
        ...
    except AException,BException:
        ...

这里会执行BException=AException，根据Python的动态执行特性，你会发现非常奇怪的现象，原始的BException不存在了，它现在只是AException的引用而已。 正确的写法：

    try:
        ...
    except (AException,BException) as e:
        ...

总之，记着任意时候都加上`as e`。

使用raise向上传递异常
    
    try:
        ...
    except:
        raise

如果需要，也可以用`sys.exc_info`获取调用堆栈上最后的异常信息。

    >>> def test():
    ...     try:
    ...         raise KeyError("key error!")
    ...     except:
    ...         exc_type, exc_value, traceback = sys.exc_info()
    ...         sys.excepthook(exc_type, exc_value, traceback)      # 打印异常信息


    >>> test()
    Traceback (most recent call last):
        raise KeyError("key error!")
    KeyError: 'key error!'

## 警告（warnings模块）
仅显示警告信息

## 断言（assert）
assert 受只读参数`__debug__`控制，可以在启动时加`-O`使其失效

    $ python -O main.py

### 上下文（context）
为代码块提供安全的初始化和清理操作。

* `__enter__`：初始化环境，返回上下文对象。
* `__exit__`：执行清理操作。返回True时，将阻止异常向外传递。

如果在with语句中使用多个上下文对象，顺序是FILO。

__contextlib__

标准库contextlib提供了contextmanager装饰器，用来简化上下文的开发。

上下文管理协议用途很广，比如：

* Synchronized：为代码提供lock/unlock线程同步
* DBContext：提供共享的数据库连接，并关闭数据库连接
* 等等

# 装饰器（Decorator）
两种构造方式

* return function
* 使用`__call__`

__Class__

为Class提供Decorator很简单，只需要将Class当作参数看待即可。

    >>> def singleton(cls):
    ...     def wrap(*args, **kwargs):
    ...         o = getattr(cls, "__instance__", None)
    ...         if not o:
    ...             o = cls(*args, **kwargs)
    ...             cls.__instance__ = o
    ...
    ...         return o
    ...
    ...     return wrap
    
    >>> @singleton
    ... class A(object):
    ...     def __init__(self, x):
    ...         self.x = x
    >>> A
    <function wrap at 0x108afff50>
    >>> a, b = A(1), A(2)
    >>> a is b
    True

也可以对cls的`__new__`方法做包装。返回一个包装后的子类，但要注意的是，会影响继承链。

    >>> def singleton(cls):
    ...     class wrap(cls):
    ...         def __new__(cls, *args, **kwargs):
    ...             o = getattr(cls, "__instance__", None)
    ...             if not o:
    ...                 o = object.__new__(cls)
    ...                 cls.__instance__ = o
    ...
    ...             return o
    ...
    ...     return wrap
    
    >>> @singleton
    ... class A(object):
    ...  def test(self): print hex(id(self))
    
    >>> a, b = A(), A()
    >>> a is b
    True
    >>> a.test()
    0x1091e9990

__元编程（MetaProgramming）__

    >>> def action(cls):
    ... cls.mvc = staticmethod(lambda: "Action")
    ... return cls
    >>> @action
    ... class Login(object): pass
    >>> Login.mvc()
    'Action'

__functools.wraps__

`functools.wraps`是装饰器的装饰器，作用是将原函数对象的指定属性复制给包装函数的对象。
默认有`__module__`、`__name__`、`__doc__`，或者通过参数选择。

# 描述符（Descriptor）
描述符协议：

    __get__(self, instance, owner) --> return value
    __set__(self, instance, value)
    __delete__(self, instance)

# 元类（metaclass）
元类创建类型，类型传将对象。

    class = metaclass(...)  # 元类创建类型
    instance = class(...)   # 类型创建实例

    instance.__class__ is class     # 实例的类型
    class.__class__ is metaclass    # 类型的类型

关键字class会被编译成创建类型对象指令，class 和 def 一样是指令，可以任何地方创建类型对象。

New-Style Class 的默认元类是type。

    >>> class Data(object): pass

    >>> Data.__class__
    <type 'type'>

    >>> type.__class__      # 最终的类型就是type，包括type自己
    <type 'type'>

## \__metaclass__
默认元类是type，可以使用`__metaclass__`自定义元类，自定义的元类一般继承自type。

实现的功能是：对类型对象创建过程进行干预。

解释器创建类型对象的时候，按照以下过程查找`__metaclass__`属性。

    class.__metaclass__ -> bases.__metaclass__ -> module.__metaclass__ -> type

# python进程与线程

## 什么时候使用多线程和多进程
__Python多线程只有用在非计算密集型应用（如I/O密集的抓取程序）上才有意义__

因为Python解释器使用了GIL（Global Interpreter Lock，全局解释器锁定），在任意时刻只允许单个Python线程执行。

- 如果程序是费I/O的，可以使用多线程；
- 如果程序是费CPU的，多线程反而会降低速度；（这种情况应该使用c模块扩展实现）

__但Python多进程是可以利用多个CPU的__

所以上面的问题也可以用python多进程来解决。
但凡是多进程的程序都应优先考虑下面的解决方案：

- 将要共享的数据放在redis或者数据库中，分布式执行程序，遇到性能问题，只要加机器就行了。
- 或者使用分布式队列系统，如ZeroMQ、RabbitMQ、Beanstalk、Thrift（这个不是队列）等， 开多个worker。

## 多进程（multiprocessing）

### 创建多进程
两种创建方式：传参和继承

传参 

    import multiprocessing
    import time
    
    def clock(interval):
        while True:
            print("The time is %s" % time.ctime())
            time.sleep(interval)
    
    if  __name__ == '__main__':
        p = multiprocessing.Process(target=clock, args=(3,))
        p.start()

继承 

    import multiprocessing
    import time
    
    class ClockProcess(multiprocessing.Process):
        def __init__(self, interval):
            multiprocessing.Process.__init__(self)
            self.interval = interval
        def run(self):
            while True:
                print("The time is %s" % time.ctime())
                time.sleep(self.interval)
    
    if  __name__ == '__main__':
        p = ClockProcess(3)
        p.start()

__Tips：__ 

如果设置了`p.daemon=True`，应在`main`结束前执行`p.join()`等待`p`进程结束，否则main thread退出，children thread也会退出

### 进程间通信（IPC）
`multiprocessing`模块支持进程间通信的2种主要形式：队列（Queue/JoinableQueue）和管道（Pipe），其他方式（非主流）：共享内存和信号量。

* Queue：
主要方法： `get()`和`put(item [, block [, timeout]])`

* JoinableQueue：
比Queue多2个主要方法：`task_done()`和`join()`

* Pipe：
在进程之间创建一条管道，并返回元组(conn1,conn2)，默认情况下，是双向的。1发的在2收，2发的在1收。
主要方法： `send(obj)` 和 `recv()`， `send_bytes(buffer [, offset [, size]])`和`recv_bytes([maxlength])`

在创建一个进程的时候，元组是不同的，所以在不同的进程执行close只会关闭当前的conn。

* 共享内存和信号量
比队列和管道效率都更高，限制也更多。应为前面两种执行了序列化和反序列化操作。

如何生产者（producer） 想要通知消费者（consumer），它不再生产任何东西，消费者应该关闭。需要编写代码设置标志（sentinel）。如：None。

    import multiprocessing
    
    def consumer(input_q):
        while True:
            item = input_q.get()
            if item is None:
                break
            print(item)
        # 关闭
        print("Consumer done")
    
    def producer(sequence, output_q):
        for item in sequence:
            output_q.put(item)
    
    if __name__ == '__main__':
        q = multiprocessing.Queue()
        cons_p = multiprocessing.Process(target=consumer, args=(q,))
        cons_p.start()
    
        sequence = [1,2,3,4]
        producer(sequence, q)
        # 设置关闭标志
        q.put(None)
    
        # 等待消费者进程关闭
        cons_p.join()

### 进程池（procss pool）

### 多进程处理的一般建议
* 避免使用共享数据，尽量使用消息传递和队列。这样可以更好的扩展。
* 避免混合多进程和多线程

## 多线程（multithreading）

### 创建多线程
与多进程一样，两种创建方式：传参和继承。（Java中也是如此）
只须将`multiprocessing.Process`换成`multiprocessing.Thread`即可

### 使用Lock
涉及使用Lock的时候，一定要用下面的形式

    # 第1种
    try:
        lock.acquire()
        # critical section
        statements
        ...
    finally:
        lock.release()

    # 第2种
    with lock:
        # critical section
        statements
        ...

### 多线程通信
提供三种队列：Queue(FIFO)/LifoQueue/PriorityQueue。

与进程的队列不同，他们都有task_done()和join()方法。
