当...时发生了什么？
==================

这个仓库试图回答一个古老的面试问题：当你在浏览器中输入google.com并且按下回车之后发生了什么？

不过我们不再局限于平常的回答，而是想办法回答地尽可能具体，不遗漏任何细节。

这将是一个协作的过程，所以深入挖掘吧，并且帮助我们一起完善它。仍然有大量的细节等待着你来添加，欢迎向我们发送Pull Requset！

这些内容使用 `Creative Commons Zero`_ 协议发布。

回车键按下
----------

为了从头开始，我们选择键盘上的回车键被按到最低处作为起点。在这个时刻，一个专用于回车键的电流回路被直接或者通过电容器闭合了，使得少量的电流进入了键盘的逻辑电路系统。这个系统会扫描每个键的状态，对于按键开关的电位弹跳变化进行噪音消除(debounce)，并将其转化为键盘码值。在这里，回车的码值是13。键盘控制器在得到码值之后，将其编码，用于之后的传输。现在这个传输过程几乎都是通过通用串行总线(USB)或者蓝牙(Bluetooth)来进行的，以前是通过PS/2或者ADB连接进行。

以USB为例子，键盘的USB元件与计算机上的USB接口(也就是USB控制器)连接，USB接口中的第一号针为它提供了5V的电压。17.78mA的电流将会通过D+针或者D-针(中间的两个针)从键盘返回计算机。携带电流的那个针被用来建立一个高速数据流(速度取决于是USB 1，2还是3)，
回车键的键码信息按照序列进行编码，从键盘传输到计算机。这个序列信号在计算机的USB控制器处被解码，然后被人机交互设备(HID)通用键盘驱动进行进一步解释。之后按键的码值被传输到操作系统的硬件抽象层。

产生中断
--------

键盘在它的中断请求线(IRQ)上发送信号，信号会被中断控制器映射到一个 ``中断向量`` (一个整数) 。CPU使用``中断描述符表(IDT)`` 把中断向量映射到对应函数(``中断处理器``)，这些函数由操作系统内核提供。当一个中断到达时，CPU按照IDT和中断向量索引到对应的中端处理器，然后内核登场了。

(在Windows上)一个 ``WM_KEYDOWN`` 消息被发往应用程序
---------------------------------------------------

HID把键盘按下的事件传送给 ``KBDHID.sys`` 驱动，把HID的信号转换成一个扫描码(Scancode)，这里回车的扫描码是 ``VK_RETURN`` (``0x0d``)。 ``KBDHID.sys`` 驱动和 ``KBDCLASS.sys`` (键盘类驱动,keyboard class driver)进行交互，这个驱动负责安全地处理所有键盘和小键盘的输入事件。之后它又去调用 ``Win32K.sys`` (在这之前有可能把消息传递给安装的第三方键盘过滤器)。这些都是发生在内核模式。

``Win32K.sys`` 通过 ``GetForegroundWindow()`` API函数找到当前哪个窗口是活跃的。这个API函数提供了当前浏览器的地址栏的句柄。Windows系统的"message pump"机制调用 ``SendMessage(hWnd, WM_KEYDOWN, VK_RETURN, lParam)`` 函数， ``lParam`` 是一个用来指示这个按键的更多信息的掩码，这些信息包括按键重复次数（这里是0）,实际扫描码（可能依赖于OEM厂商，不过通常不会是 ``VK_RETURN`` ），功能键（alt, shift, ctrl）是否被按下（在这里没有），以及一些其他状态。

Windows的 ``SendMessage`` API直接将消息添加到特定窗口句柄(``hWnd``)的消息队列中，之后赋给 ``hWnd`` 的主要消息处理函数（叫做 ``WindosProc``）将会被调用，用于处理队列中的消息。

当前活跃的句柄(``hWnd``)实际上是一个edit control控件，这种情况下，``WindowProc`` 有一个用于处理 ``WM_KEYDOWN`` 消息的处理器，这段代码会查看 ``SendMessage`` 传入的第三个参数(``wParam``)，因为这个参数是 ``VK_RETURN`` ，于是它知道用户按下了回车键。


(在Mac OS X上)一个 ``KeyDown`` NSEvent被发往应用程序
----------------------------------------------------

中断信号引发了I/O Kit Kext键盘驱动的中断处理事件，驱动把信号翻译成键码值，然后传给OS X的 ``WindowServer`` 进程。然后， ``WindowServer`` 将这个事件通过Mach端口分发给合适的（活跃的，或者正在监听的）应用程序，这个信号会被放到应用程序的消息队列里。队列中的消息可以被拥有足够高权限的线程使用``mach_ipc_dispatch``函数读取到。这个过程通常是由 ``NSApplication`` 主事件循环产生并且处理的，通过 ``NSEventType`` 为 ``KeyDown`` 的 ``NSEvent`` 。

输入的是URL还是搜索的关键字？
-----------------------------

解析URL
--------

检查HSTS列表
------------

转换非ASCII的Unicode字符
------------------------

* 浏览器检查输入是否含有不是 ``a-z``， ``A-Z``，``0-9``， ``-``或者``.``的字符
* 这里主机名是``google.com``，所以没有非ASCII的字符，如果有的话，浏览器会对主机名部分使用 `Punnycode`_ 编码

DNS查询
-------

* 浏览器检查域名是否在缓存当中
* 如果缓存中没有，就去调用 ``gethostbynme`` 库函数（操作系统不同函数也不同）进行查询
* ``gethostbyname``函数在试图进行DNS解析之前首先检查域名是否在 ``/etc/hosts`` 里
* 如果 ``gethostbyname`` 没有这个域名的缓存记录，也没有在 ``hosts`` 里找到，它将会向DNS 服务器发送一条DNS查询请求。DNS服务器是有网络通信栈提供的，通常是本地路由器或者ISP的缓存DNS服务器。

* 本地DNS服务器接收到查询请求


.. _`Creative Commons Zero`: https://creativecommons.org/publicdomain/zero/1.0/
.. _`"CSS lexical and syntax grammar"`: http://www.w3.org/TR/CSS2/grammar.html
.. _`Punycode`: https://en.wikipedia.org/wiki/Punycode
.. _`Ethernet`: http://en.wikipedia.org/wiki/IEEE_802.3
.. _`WiFi`: https://en.wikipedia.org/wiki/IEEE_802.11
.. _`Cellular data network`: https://en.wikipedia.org/wiki/Cellular_data_communication_protocol
.. _`analog-to-digital converter`: https://en.wikipedia.org/wiki/Analog-to-digital_converter
.. _`network node`: https://en.wikipedia.org/wiki/Computer_network#Network_nodes
