# 如何编译runtime-core的js源码

## 源码下载:

> js框架，引擎可以单拉代码仓来编译
>
> 拉取的命令: `https://gitee.com/openharmony/ace_engine_lite.git`

## 源码结构:

```
├── frameworks      # 框架代码目录
│   ├── examples    # 示例代码目录
│   ├── include     # 头文件目录
│   ├── packages    # 框架JS实现存放目录
│   ├── src         # 源代码存放目录
│   ├── targets     # 各目标设备配置文件存放目录
│   └── tools       # 工具代码存放目录
├── interfaces      # 对外接口存放目录
│   └── innerkits   # 对内部子系统暴露的头文件存放目录
│       └── builtin # JS应用框架对外暴露JS三方module API接口存放目录
└── test            # 测试用例目录
```

### `JS`框架结构:

```
+--- core
|   +--- index.js
+--- index.js
+--- observer
|   +--- index.js
|   +--- observer.js
|   +--- subject.js
|   +--- utils.js
+--- profiler
|   +--- index.js
+--- __test__
|   +--- index.test.js
```

## 编译`JS`源码：

- `npm install`
- `npm run build`
- 如果有模块确实，需要用`npm install 模块名字`
- 编译成功后会在`build`文件夹下有生成`framework.min.js`
- 编译jerry的bin
- 生成`framework.min.bc`的二进制
- 更新到`.h`文件中参与系统编译

