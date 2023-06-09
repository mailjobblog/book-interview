# PHP面试题

### 常见的数组函数

array_chunk($arr,$number)：把数组分割为有特定($number)个元素的数组块。

array_column($arr,$column)：返回数组中某一个单列的值。

array_combine($arr1,$arr2)：合并两个数组为一个新数组，并把$arr1的值作为键，$arr2的值作为值。

array_count_values(Array('a','b','c','a','b'))：返回数组中所有值出现的次数，函数执行结果：Array ( [a] => 2 [b] => 2 [c] => 1 )。

array_diff($arr1,$arr2)：返回两个数组的差集(只比较键值)

array_diff_key($arr1,$arr2)：返回两个数组的差集(只比较键名)，该数组返回在$arr1中，但不在 $arr2中的元素。

array_intersect($arr1,$arr2)：比较数组，返回两个数组的交集。

array_key_exists($key,$arr)：查询数组中是否存在指定的键名。

array_keys($arr)：返回数组中所有的键名，并且组成一个新数组。

array_map('myFunction',$arr)：将用户自定义函数作为回调函数作用在数组的每个元素上，返回一个新数组。

array_merge($arr1,$arr2,$arr3.....)：合并一个或多个元素为一个新数组，如果两个或更多元素有相同的键名，后者会覆盖前者。

### Composer自动加载原理

- spl_autoload_register() 调用 __autoload 注册多个堆栈列表
- 当php执行找不到某一个类的时候，会调用这个堆栈列表，实现自动加载

### PHP-FPM生命周期

- PHP-FPM初始化：启动zend引擎，加载注册的扩展模块。php-fpm的master进程启动多个worker进程，通过socket监听各个worker进程，各个worker进程accept请求，准备解析fastCGI程序。
- 请求到达nginx后，nginx会公共fastCGI协议通知php-fpm。
- php-fpm读取脚本文件，进行语法分析，词法解析，生成语法树。
- zend引擎编译语法树，生成opcode。
- zend引擎执行opcode，返回脚本执行结果。

### PHP-FPM优化方法

- static（静态）：代表fpm启动时，master进程fork出max_children个worker进程

- dynamic（动态）：代表fpm启动时候，master进程先fork出start_servers个worker进程，随着并发量的增大，最大启动max_children个worker个进程。

总结：静态相比动态，缺点是：会在并发量小的时候，占用更多的系统资源。优点是：可以拥有更好的性能。

max_children：理论上该志配置的越大越好，因为可以拥有更高的并发性能。但是他也是服务器配置决定的，一般来说每一个php-cgi在峰值的时候占用内存是20M，所以配置10的话，意味着最大占用内存100*20M=2000M。反过来说配置的越小，php-fpm的并发性能越差。如果一个请求长时间没有得到处理，就会出现 504 Gateway time-out 的错误。如果处理脚本的worker出现问题，那么就会出现 502 bad-Gateway 的错误。

start_servers：代表php-fpm启动时，动态模式下最小启动进程数量，此配置的默认值是2。一般而言，配置10～20就足够了。

max_request：代表每一个php-cgi在处理多少请求数就自动重启该进程。主要解决的是：如果php-fpm如果不定期重启php-cgi那么就会造成内存的上升或者内存的泄漏。但是存在的问题是，php-cgi间歇性的不可用，会造成502的错误，所以这也是在高并发的时候出现502错误的原因。一般来说根据压测，第一个php-cgi的重启和最后一个php-cgi的重启在1分钟左右的时候，是一个良好的配置。

### PHP垃圾回收机制

5.3 之前采用引用计数：如果一个变量被定义或者引用，那么计数器 refcount 就会 +1 ，如果变量被销毁那么就会 -1 ，如果判断 refcount = 0，那么就会垃圾回收。但是当遇到环转环状引用后，销毁后计数器并不会消减为0，所以也就不会被垃圾回收，造成内存泄露。

5.3 之后引入了根缓冲区：当一个变量被定义的时候，会在zend内部维护一个 zend value 容器，zend value 容器中除了计数器refcount，还引入了 is_ref 的布尔形变量，用来标记是否存在存在环状引用，并且放到根缓冲区。如果当根缓冲区的数量达到配置文件的数量后，就会开启垃圾回收。如果引用计数减小到0，所有容器都会被清除，则不属于垃圾。如果减小后还大于0，那么会进入垃圾回收周期，通过判断是否 -1 ，并且引用次数是否为0，来发现垃圾。

### Laravel生命周期

- Composer 自动加载项目依赖
- 创建应用实例
  - 创建容器
  - 绑定内核（HTTP 内核类 Console 内核 绑定异常处理）
- 接收请求并响应
- 解析内核
  - 处理 HTTP 请求
- 发送响应
- 终止应用程序

### 如何保证接口安全性

- 数据加密，防止报文明文攻击
- 数据加密验签
- token授权机制
- 时间戳timestamp超时机制
- timestamp+nonce方案防止重放攻击
- 限流机制
- 黑名单机制
- 白名单机制
- 数据脱敏掩码
- 数据参数合法性校验

### PHP如何读取大文件

- 配置较大的 memory_limit 参数，就可以用 file_get_content 全部读取到内存
- 使用php的fseek读取文件，不需要将文件内容全部读取到内存，而使用指针操作。使用 \n 判断换行数据，使用 EOF 判断文件结束，可以一段一段的读取。
- 使用Linux命令读取文件，例如 tail 命令来显示最后几行