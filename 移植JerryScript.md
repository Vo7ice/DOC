# `JerryScript`移植
> 介绍`JerryScript`如何移植到281
> 如何能够解释`JS`语言的

## `简介`
1. `Jerryscript`是由三星开发的一款`JavaScript`引擎，是为了让`JavaScript`开发者能够构建物联网应用。
2. 物联网设备在`CPU`性能和内存空间上都有着严重的制约。因此，三星设计了`JerryScript`引擎，它能够运行在小于`64KB`内存上，且全部代码能够存储在不足`200KB`的只读存储（`ROM`）上。

## `代码结构`
```c
.
+--- docs                        文档
+--- jerry-core                  
|   +--- api                     可用的api
|   +--- config.h
|   +--- debugger                开启JERRY_DEBUGGER可调试
|   +--- ecma
|   |   +--- base                基础属性
|   |   +--- builtin-objects     js内建的对象
|   |   |   +--- typedarray      数组类型
|   |   +--- operations          js基础语法
|   +--- include                 头文件
|   +--- jcontext                上下文
|   +--- jmem                    内存
|   +--- jrt                     宏定义
|   +--- lit                     js中关键字
|   +--- parser
|   |   +--- js                  解释
|   |   +--- regexp              JERRY_BUILTIN_REGEXP 控制
|   +--- vm                      虚拟机
+--- jerry-ext
|   +--- arg                     js中的参数
|   +--- common
|   +--- debugger                开启JERRY_DEBUGGER可调试
|   +--- handler-scope           处理js中的scope
|   +--- handler                 处理assert,gc,print,register
|   +--- include
|   +--- module                  处理js中的模块
+--- jerry-libm                  数学公式
+--- jerry-port                  移植文件,与各个系统不同
|   +--- default
|   |   +--- include
```
## `移植步骤`
  1. 将`jerryscript`工程引入到主工程中。
  2. 将`jerry-core`, `jerry-ext`, `jerry-port`参与编译,头文件设置要准确
## `使用示例`
  > 编译通过后使用`docs`里面的`03.API-EXAMPLE.md`包含的例子来使能`jerryscript`
  > 以281为例子, 在`touchgfx_init`函数中调用`start_jerry`
  ``` c++
    void start_jerry()
    {
      /* 从js文件中读取到内存的js源码 */
      const jerry_char_t script[] = "print ('Hello, JS!');";
      /* 初始化引擎环境的上下文,申请内存 */
      jerry_context_t * engineContext_ = jerry_create_context(48 * 1024, context_alloc, NULL);
      /* 设置全局的上下文 */
      jerry_port_default_set_context(engineContext_);
      /* 初始化引擎 */
      jerry_init (JERRY_INIT_EMPTY);
      /* 将print函数与native函数绑定 */
      jerryx_handler_register_global ((const jerry_char_t *) "print",
                                  jerryx_handler_print);
      /* 初始化解释js脚本文件 */
      jerry_value_t parsed_code = jerry_parse (NULL, 0, script, sizeof(script) - 1, JERRY_PARSE_NO_OPTS);
      if (!jerry_value_is_error (parsed_code))
      {
        /* 执行解释后的脚本文件 */
        jerry_value_t ret_value = jerry_run (parsed_code);
        /* 返回值必须释放 */
        jerry_release_value (ret_value);
      }
      /* 解释后的代码必须释放 */
      jerry_release_value (parsed_code);
      /* 引擎内存销毁 */
      jerry_cleanup ();
    }
  ```
  这个例子在运行js脚本的时候执行`print`函数时,会通过`jerry`调用与之绑定的函数.
  在绑定的函数中可以将`js`打印的字符通过`native`的方式打印出来.