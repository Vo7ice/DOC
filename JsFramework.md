# Js app的生命周期
- 从`startAbility`后调用到`js`的`app`
1. `AceAbility::OnStart`函数
   ``` C++
   void AceAbility::OnStart(const Want &want)
    {
        HILOG_DEBUG(HILOG_MODULE_ACE, "AceAbility OnStart");
        eventHandler_ = AbilityEventHandler::GetCurrentHandler();
    
        const char * const abilitySrcPath = GetSrcPath();
        const char * const bundleName = GetBundleName();
        HILOG_DEBUG(HILOG_MODULE_ACE, "ace ability src path = %s", abilitySrcPath);
        HILOG_DEBUG(HILOG_MODULE_ACE, "ace ability bundle name = %s", bundleName);
    
        // 重新定位js bundle文件的路径 ex:/system/app/73709738-2d9d-4947-ac63-9858dcae7ccb/src/index.js
        char* jsBundlePath = RelocateJSSourceFilePath(abilitySrcPath, "assets/js/default/");
        HILOG_DEBUG(HILOG_MODULE_ACE, "ace ability js bundle path = %s", jsBundlePath);
        const uint16_t token = 0xff; // js ability's token is hidden by AMS
        jsAbility_.Launch(jsBundlePath, bundleName, (uint16_t)token);
        ACE_FREE(jsBundlePath);
        RootView *rootView = RootView::GetInstance();
        if (rootView == nullptr) {
            HILOG_ERROR(HILOG_MODULE_ACE, "get rootView is nullptr");
            return;
        }
        SetUIContent(rootView);
        Ability::OnStart(want);
        HILOG_DEBUG(HILOG_MODULE_ACE, "AceAbility OnStart Done");
    }
    ```
2. `JSAbility::Launch`函数
   ``` C++
    void JSAbility::Launch(const char * const abilityPath, const char * const bundleName, uint16_t token)
    {
        // 检查impl实现是否为空
        if (jsAbilityImpl_ != nullptr) {
            HILOG_ERROR(HILOG_MODULE_ACE, "Launch only can be triggered once");
            ACE_ERROR_CODE_PRINT(EXCE_ACE_FWK_LAUNCH_FAILED, EXCE_ACE_APP_ALREADY_LAUNCHED);
            return;
        }
    
        // 检查app路劲是否存在
        if ((abilityPath == nullptr) || (strlen(abilityPath) == 0)) {
            HILOG_ERROR(HILOG_MODULE_ACE, "invalid app path");
            ACE_ERROR_CODE_PRINT(EXCE_ACE_FWK_LAUNCH_FAILED, EXCE_ACE_INVALID_APP_PATH);
            return;
        }
    
        // 检查源文件是否存在
        if ((bundleName == nullptr) || (strlen(bundleName) == 0)) {
            HILOG_ERROR(HILOG_MODULE_ACE, "invalid bundle name");
            ACE_ERROR_CODE_PRINT(EXCE_ACE_FWK_LAUNCH_FAILED, EXCE_ACE_INVALID_BUNDLE_NAME);
            return;
        }
    
        // 检查剩余内存
        DumpNativeMemoryUsage();
        // 初始化impl实例
        jsAbilityImpl_ = new JSAbilityImpl();
        // 初始化失败
        if (jsAbilityImpl_ == nullptr) {
            HILOG_ERROR(HILOG_MODULE_ACE, "Create JSAbilityRuntime failed");
            return;
        }
        // 开启trace,性能看护
        START_TRACING(LAUNCH);
        // 强转void* -> JSAbilityImpl*
        JSAbilityImpl *jsAbilityImpl = CastAbilityImpl(jsAbilityImpl_);
        // 注册错误看护
        FatalHandler::GetInstance().RegisterFatalHandler();
        // 开启js环境
        jsAbilityImpl->InitEnvironment(abilityPath, bundleName, token);
        // 下发js的create函数
        jsAbilityImpl->DeliverCreate();
        // 停止trace
        STOP_TRACING();
        // 输出trace
        OUTPUT_TRACE();
    }
   ```
3. `JSAbilityImpl::InitEnvironment`函数
   ``` C++
    void JSAbilityImpl::InitEnvironment(const char * const abilityPath, const char * const bundleName, uint16_t token)
    {
        // 检查入参是否满足标准
        if ((abilityPath == nullptr) || strlen(abilityPath) == 0 || (bundleName == nullptr) || strlen(bundleName) == 0) {
            HILOG_ERROR(HILOG_MODULE_ACE, "invalid input parameters");
            return;
        }
    
        // 检查是否已初始过
        if (isEnvInit_) {
            HILOG_ERROR(HILOG_MODULE_ACE, "already initialized, return");
            return;
        }
        // init engine && js fwk
        JsAppEnvironment *appJsEnv = JsAppEnvironment::GetInstance();
        appContext_ = JsAppContext::GetInstance();
        // check if we should use snapshot mode, do this before everything,
        // but after debugger config is set
        // 检查模式，snapshot为设备默认模式，js解析为模拟器默认模式
        // 由宏 TARGET_SIMULATOR 控制
        appJsEnv->InitRuntimeMode();
        // 设置app的信息
        appContext_->SetCurrentAbilityInfo(abilityPath, bundleName, token);
        // 设置当前运行ability
        appContext_->SetTopJSAbilityImpl(this);
        // 初始化js框架
        appJsEnv->InitJsFramework();
    
        // initialize js object after engine started up
        abilityModel_ = UNDEFINED;
        nativeElement_ = UNDEFINED;
        isEnvInit_ = true;
    
        // relocate app.js fullpath
        // 将app.js或app.bc重定向到完整js源代码路径
        const char * const appJSFileName = (appJsEnv->IsSnapshotMode()) ? "app.bc" : "app.js";
        char *fileFullPath = RelocateJSSourceFilePath(abilityPath, appJSFileName);
        if (fileFullPath == nullptr) {
            HILOG_ERROR(HILOG_MODULE_ACE, "relocate js file failed");
            ACE_ERROR_CODE_PRINT(EXCE_ACE_FWK_LAUNCH_FAILED, EXCE_ACE_APP_ENTRY_MISSING);
            return;
        }
    
        // 开始对js源代码进行eval
        START_TRACING(APP_CODE_EVAL);
        abilityModel_ = appContext_->Eval(fileFullPath, strlen(fileFullPath), true); // generate global.$app js object
        STOP_TRACING();
    
        // 释放资源，初始化路由对象
        ace_free(fileFullPath);
        fileFullPath = nullptr;
        // 初始化页面路由
        router_ = new Router();
        if (router_ == nullptr) {
            HILOG_ERROR(HILOG_MODULE_ACE, "malloc router heap memory failed.");
            return;
        }
    }
    ```
4. `JsAppEnvironment::InitJsFramework`函数
    ``` C++
    void JsAppEnvironment::InitJsFramework() const
    {
        // trace ENGINE_INIT
        START_TRACING(ENGINE_INIT);
        // 以当前时间设置随机数种子
        Srand((unsigned)jerry_port_get_current_time());
        // 设置调试的上下文
        Debugger::GetInstance().SetupJSContext();
        // 初始化jerry环境
        jerry_init(JERRY_INIT_EMPTY);
        // stop trace
        STOP_TRACING();
        // trace fwk_init
        START_TRACING(FWK_INIT);
    #ifdef JSFWK_TEST
        jerry_value_t globalThis = jerry_get_global_object();
        jerry_release_value(jerryx_set_property_str(globalThis, "globalThis", globalThis));
        jerry_release_value(globalThis);
    #endif // JSFWK_TEST
        // 加载内建模块
        LoadAceBuiltInModules();
        // 加载framework js脚本 framework_min_js.h或者framework_min_bc.h
        LoadFramework();
        // 加载app的模块
        LocalModule::Load();
        STOP_TRACING();
    }
    ```
5. `JsAppEnvironment::LoadFramework`函数
    ``` C++
    void JsAppEnvironment::LoadFramework() const
    {
        size_t len = 0;
        // 加载编译好的js脚本解析
        // load framework js/snapshot file to buffer
        const char * const jsFrameworkScript = GetFrameworkRawBuffer(snapshotMode_, len);
        const jerry_char_t *jScript = reinterpret_cast<const jerry_char_t *>(jsFrameworkScript);
        // eval framework to expose
        START_TRACING(FWK_CODE_EVAL);
    
        jerry_value_t retValue = UNDEFINED;
        if (snapshotMode_) {
            retValue = jerry_exec_snapshot(reinterpret_cast<const uint32_t *>(jScript), len, 0, 1);
        } else {
            retValue = jerry_eval(jScript, len, JERRY_PARSE_NO_OPTS);
        }
        STOP_TRACING();
        // 解析成功，处理返回值
        bool hasError = jerry_value_is_error(retValue);
        if (hasError) {
            HILOG_ERROR(HILOG_MODULE_ACE, "Failed to load JavaScript framework.");
            ACE_ERROR_CODE_PRINT(EXCE_ACE_FWK_LAUNCH_FAILED, EXCE_ACE_INIT_FWK_FAILED);
            PrintErrorMessage(retValue);
        } else {
            HILOG_INFO(HILOG_MODULE_ACE, "Success to load JavaScript framework.");
        }
        jerry_release_value(retValue);
        Debugger::GetInstance().StartDebugger();
    }
    ```
6. `LocalModule::Load`函数
    ``` C++
    void LocalizationModule::Init()
    {
        // 获取全局的上下文
        jerry_value_t globalContext = jerry_get_global_object();
        const char * const name = "ViewModel";
        jerry_value_t propertyName = jerry_create_string(reinterpret_cast<const jerry_char_t *>(name));
        if (JerryHasProperty(globalContext, propertyName)) {
            // get the prototype of AbilitySlice
            jerry_value_t currentApp = jerry_get_property(globalContext, propertyName);
            jerry_value_t protoType = jerryx_get_property_str(currentApp, "prototype");
            // register $t to the prototype of abilitySlice
            jerry_value_t functionHandle = jerry_create_external_function(Translate);
            const char * const propName = "$t";
            JerrySetNamedProperty(protoType, propName, functionHandle);
            // register $tc to the prototype of abilitySlice
    #ifdef LOCALIZATION_PLURAL
            jerry_value_t pluralHandle = jerry_create_external_function(TranslatePlural);
            const char * const pluralFuncName = "$tc";
            JerrySetNamedProperty(protoType, pluralFuncName, pluralHandle);
            jerry_release_value(pluralHandle);
    #endif // LOCALIZATION_PLURAL
            ReleaseJerryValue(functionHandle, protoType, currentApp, VA_ARG_END_FLAG);
        } else {
            HILOG_ERROR(HILOG_MODULE_ACE, "app is not create.");
        }
        ReleaseJerryValue(propertyName, globalContext, VA_ARG_END_FLAG);
    }
    ```
7. `jerry_value_t JsAppContext::Eval`函数
    ``` C++
    jerry_value_t JsAppContext::Eval(const char * const jsFileFullPath, size_t fileNameLength, bool isAppEval) const
    {
        // 检查入参
        if ((jsFileFullPath == nullptr) || (fileNameLength == 0)) {
            HILOG_ERROR(HILOG_MODULE_ACE, "Failed to eval js code cause by empty JavaScript script.");
            ACE_ERROR_CODE_PRINT(EXCE_ACE_ROUTER_REPLACE_FAILED, EXCE_ACE_PAGE_INDEX_MISSING);
            return UNDEFINED;
        }
    
        uint32_t contentLength = 0;
        // trace 页面初始化
        START_TRACING(PAGE_CODE_LOAD);
        // 读取环境标记
        bool snapshotMode = JsAppEnvironment::GetInstance()->IsSnapshotMode();
        // 读取源文件
        char *jsCode = ReadFile(jsFileFullPath, &contentLength, snapshotMode);
        STOP_TRACING();
        // 检查源文件
        if (jsCode == nullptr) {
            HILOG_ERROR(HILOG_MODULE_ACE, "empty js file, eval user code failed");
            ACE_ERROR_CODE_PRINT(EXCE_ACE_ROUTER_REPLACE_FAILED, EXCE_ACE_PAGE_FILE_READ_FAILED);
            return UNDEFINED;
        }
    
        START_TRACING(PAGE_CODE_EVAL);
        jerry_value_t viewModel = UNDEFINED;
        if (snapshotMode) {
            viewModel = jerry_exec_snapshot(reinterpret_cast<const uint32_t *>(jsCode), contentLength, 0, 1);
        } else {
            // 源码字符强转成jerry的char*
            const jerry_char_t *jsScript = reinterpret_cast<const jerry_char_t *>(jsCode);
            // 调用jerry_parse函数解析源码
            jerry_value_t retValue = jerry_parse(reinterpret_cast<const jerry_char_t *>(jsFileFullPath), fileNameLength, jsScript, contentLength, JERRY_PARSE_NO_OPTS);
            if (jerry_value_is_error(retValue)) {
                ACE_ERROR_CODE_PRINT(EXCE_ACE_ROUTER_REPLACE_FAILED, EXCE_ACE_PAGE_JS_EVAL_FAILED);
                PrintErrorMessage(retValue);
                // free js code buffer
                ace_free(jsCode);
                jsCode = nullptr;
                jerry_release_value(retValue);
                STOP_TRACING();
                return UNDEFINED;
            }
            // 调用jerry_run执行解析后的jerry object
            viewModel = jerry_run(retValue);
            jerry_release_value(retValue);
        }
    
        STOP_TRACING();
        // free js code buffer
        ace_free(jsCode);
        jsCode = nullptr;
    
        if (jerry_value_is_error(viewModel)) {
            ACE_ERROR_CODE_PRINT(EXCE_ACE_ROUTER_REPLACE_FAILED, EXCE_ACE_PAGE_JS_EVAL_FAILED);
            PrintErrorMessage(viewModel);
            jerry_release_value(viewModel);
            return UNDEFINED;
        }
    
        SetGlobalNamedProperty(isAppEval, viewModel);
        return viewModel;
    }
    ```
8. `JSAbilityImpl::DeliverCreate`函数
    ``` C++
    void JSAbilityImpl::DeliverCreate()
    {
        // trace oncreate
        START_TRACING(APP_ON_CREATE);
        // call InvokeOnCreate
        InvokeOnCreate();
        STOP_TRACING();
        // if we have done the render or not initialized yet, don't call render
        if (rendered_ || (appContext_ == nullptr)) {
            ACE_ERROR_CODE_PRINT(EXCE_ACE_FWK_LAUNCH_FAILED, EXCE_ACE_APP_RENDER_FAILED);
            return;
        }
        // call render to setup user interface
        jerry_value_t object = jerry_create_object();
        JerrySetStringProperty(object, ROUTER_PAGE_URI, PATH_DEFAULT);
        if (router_) {
            jerry_release_value(router_->Replace(object, false));
            rendered_ = true;
        }
        jerry_release_value(object);
    }
    ```
9. `JSAbilityImpl::InvokeOnCreate`函数
    ``` C++
    void JSAbilityImpl::InvokeOnCreate() const
    {
        if (IS_UNDEFINED(abilityModel_)) {
            HILOG_ERROR(HILOG_MODULE_ACE, "view model is undefined when call user's init");
            return;
        }
        // 从解析的jerryobject对象中调用oncreate函数
        jerry_value_t onCreateFunction = jerryx_get_property_str(abilityModel_, ABILITY_LIFECYCLE_CALLBACK_ON_CREATE);
        if (IS_UNDEFINED(onCreateFunction)) {
            // user does not set onInit method
            return;
        }
        CallJSFunctionAutoRelease(onCreateFunction, abilityModel_, nullptr, 0);
        jerry_release_value(onCreateFunction);
    }
    ```
10. 至此一个`js`的`ability`可以调用正常生命周期函数了
    - 源码位置: `OpenHarmony\foundation\ace\frameworks\lite`
    - `JS`执行环境：`OpenHarmony\foundation\ace\frameworks\lite\packages\runtime-core`
    - `JS`解析库: `OpenHarmony\third_party\jerryscript`
    - `cJSON`: `OpenHarmony\third_party\cJSON`