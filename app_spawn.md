# APP SPAWN 应用孵化

## AppSpawnInit
  - 初始化`g_appSpawnService`
  - 调用`invoke`
  ### invoke函数
    1. SplitMessage 通过进程间通信函数IpcIoPopDataBuff从 ID_CALL_CREATE_SERVICE 获取MessageSt
``` C
  typedef struct {
      char* bundleName;
      char* sharedLibPaths;
      unsigned long long identityID;
      int uID;
      int gID;
  } MessageSt;
```
    2. 调用`CreateProcess`函数启动
    

## appspawn_process::CreateProcess
  1. GetEnvStrs 拿环境变量
  2. 判断/bin/abilityMain是否存在
  3. fork一个进程
  4. 通过bundle执行abilityMain来运行一个app