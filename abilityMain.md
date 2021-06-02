# abilityMain
- 用来启动应用的入口

## 编译所需组件
    # hilog
    "//base/hiviewdfx/frameworks/ilog_litefeatured:hilog_shared",
    # cjson
    "//third_party/cJSON:cjson_shared",
    # ablity组件
    "//foundation/aafwk/frameworksability_lite:aafwk_abilitykit_lite",
    # bundle组件
    "//foundation/appexecfwk/frameworks/bundle_lite:bundle",
    # 跨进程组件
    "//foundation/communication/frameworksipc_lite:liteipc_adapter",
    # 系统服务组件
    "//foundation/distributedschedule/services/samgr_litesamgr:samgr",
    # 安全函数组件
    "//third_party/bounds_checking_function:libsec_shared",
    # kv组件
    "//utils/native/lite/kv_store:kv_store",

## 入口
- 启动`AbilityThread`
    1. OHOS::AbilityThread::ThreadMain(token);
    2. abilityThread.AttachBundle(token);
    3. abilityThread.Run();
- `AttachBundle`函数
    1. 往服务端发送`ATTACH_BUNDLE`命令
    ``` C++
    foundation/aafwk/frameworks/ability_lite/src/ability_thread.cpp
    // 构造AbilityEventHandler,AbilityScheduler两个实例
    eventHandler_ = new AbilityEventHandler();
    abilityScheduler_ = new AbilityScheduler(*eventHandler_, *this);
    // 检查ams是否初始化
    if (!AbilityMsClient::GetInstance().Initialize()) {
        HILOG_ERROR(HILOG_MODULE_APP, "ams feature is null");
        return;
    }
    // 检查identity是否正常
    identity_ = static_cast<SvcIdentity *>(AdapterMalloc(sizeof(SvcIdentity)));
    if (identity_ == nullptr) {
        HILOG_ERROR(HILOG_MODULE_APP, "ams identity is null");
        return;
    }
    // 注册跨进程回调的callback，接受服务端的命令
    int32_t ret = RegisteIpcCallback(AbilityScheduler::AmsCallback, 0, IPC_WAIT_FOREVER, identity_, abilityScheduler_);
    if (ret != 0) {
        HILOG_ERROR(HILOG_MODULE_APP, "RegisterIpcCallback failed");
        AdapterFree(identity_);
        return;
    }
    // 用ATTACH_BUNDLE进行进程间通信
    AbilityMsClient::GetInstance().ScheduleAms(nullptr, token, identity_, ATTACH_BUNDLE);
    ```
    2. 处理`ATTACH_BUNDLE`
    ``` C++
    foundation/aafwk/services/abilitymgr_lite/src/ability_mgr_handler.cpp 
    void AbilityMgrHandler::AttachBundle(AbilityThreadClient *client)
    {
        // 空指针检查
        PRINTD("AbilityMgrHandler", "start");
        CHECK_NULLPTR_RETURN(client, "AbilityMgrHandler", "invalid augument");
        AbilityMsStatus status = abilityWorker_.AttachBundle(*client);
        // 回收内存
        delete client;
        CHECK_RESULT_LOG(status);
    }
    AbilityMsStatus AbilityWorker::AttachBundle(const AbilityThreadClient &client)
    {
        PRINTD("AbilityWorker", "app token(%{private}" PRIu64 ")", client.GetToken());
        // 启动一个task
        AbilityAttachTask attachTask(&client);
        return attachTask.Execute();
    }
    ```
    3. `AbilityAttachTask::Execute`函数
    ``` C++
    AbilityMsStatus AbilityAttachTask::Execute()
    {
        PRINTD("AbilityAttachTask", "start");
        // 检查空指针
        if (client_ == nullptr) {
            return AbilityMsStatus::TaskStatus("Attach", "invalid argument");
        }
        // step1: Get app record by token.
        auto appRecord = const_cast<AppRecord *>    (AppManager::GetInstance().GetAppRecordByToken(
            client_->GetToken(), client_->GetPid()));
        if (appRecord == nullptr) {
            return AbilityMsStatus::TaskStatus("Attach", "appRecord not found");
        }
        // step2: Save ability thread client.
        AbilityMsStatus status = appRecord->SetAbilityThreadClient (*client_);
        CHECK_RESULT(status);
    
        // step3: Load permission
        status = appRecord->LoadPermission();
        if (!status.IsOk()) {
            AppManager::GetInstance().RemoveAppRecord(*appRecord);
            return status;
        }
    
        // step4: Init app
        status = appRecord->AppInitTransaction();
        CHECK_RESULT(status);
    
        // step5: Launch pending ability.
        return appRecord->LaunchPendingAbility();
    }
    ```
    4. `AppInitTransaction` 函数,下发`SCHEDULER_APP_INIT`命令
    ``` C++
    AbilityMsStatus AppRecord::AppInitTransaction() const
    {
        if (abilityThreadClient_ != nullptr) {
            return abilityThreadClient_->AppInitTransaction(bundleInfo_);
        }
        return AbilityMsStatus::AppTransanctStatus("app init ability thread client not exsit");
    }
    AbilityMsStatus AbilityThreadClient::AppInitTransaction(const     BundleInfo &bundleInfo)
    {
        PRINTD("AbilityThreadClient", "start");
        if (bundleInfo.bundleName == nullptr || bundleInfo.codePath == nullptr ||
            bundleInfo.numOfModule > MAX_MODULE_SIZE) {
            return AbilityMsStatus::AppTransanctStatus("app init invalid argument");
        }
        // 构造跨进程对象
        IpcIo req;
        char data[IPC_IO_DATA_MAX];
        IpcIoInit(&req, data, IPC_IO_DATA_MAX, 0);
        IpcIoPushString(&req, bundleInfo.bundleName);
        IpcIoPushString(&req, bundleInfo.codePath);
        IpcIoPushString(&req, bundleInfo.dataPath);
        IpcIoPushBool(&req, bundleInfo.isNativeApp);
        // transact moduleName
        IpcIoPushInt32(&req, bundleInfo.numOfModule);
        for (int i = 0; i < bundleInfo.numOfModule; i++) {
            if (bundleInfo.moduleInfos[i].moduleName != nullptr) {
                IpcIoPushString(&req, bundleInfo.moduleInfos[i].moduleName);
            }
        }
        uintptr_t ptr;
        // 下发SCHEDULER_APP_INIT命令
        if (Transact(nullptr, svcIdentity_, SCHEDULER_APP_INIT, &req,
            nullptr, LITEIPC_FLAG_DEFAULT, &ptr) != LITEIPC_OK) {
            return  AbilityMsStatus::AppTransanctStatus("app init ipc error");
        }
        // 释放内存
        FreeBuffer(nullptr, reinterpret_cast<void *>(ptr));
        return AbilityMsStatus::Ok();
    }
    ```
    5. 处理`SCHEDULER_APP_INIT`
    ``` C++
    case SCHEDULER_APP_INIT: {
        // 从跨进程组件中拿到需要的app信息
        AppInfo appInfo;
        char *bundleName = reinterpret_cast<char *>(IpcIoPopString(data, nullptr));
        char *srcPath = reinterpret_cast<char *>(IpcIoPopString(data, nullptr));
        char *dataPath = reinterpret_cast<char *>(IpcIoPopString(data, nullptr));
        if ((bundleName == nullptr) || (srcPath == nullptr) || (dataPath == nullptr)) {
            HILOG_ERROR(HILOG_MODULE_APP, "ams call back error, bundleName, srcPath or dataPath is null");
            ClearIpcMsg(ipcMsg);
            return PARAM_NULL_ERROR;
        }
        appInfo.bundleName = bundleName;
        appInfo.srcPath = srcPath;
        appInfo.dataPath = dataPath;
        appInfo.isNativeApp = IpcIoPopBool(data);
        int moduleSize = IpcIoPopInt32(data);
        if (moduleSize > MAX_MODULE_SIZE) {
            HILOG_ERROR(HILOG_MODULE_APP, "moduleSize is too big");
            ClearIpcMsg(ipcMsg);
            return COMMAND_ERROR;
        }
        for (int i = 0; i < moduleSize; i++) {
            char *moduleName = reinterpret_cast<char *>(IpcIoPopString(data, nullptr));
            if ((moduleName != nullptr) && (strlen(moduleName) > 0)) {
                appInfo.moduleNames.emplace_front(moduleName);
            }
        }
        // 调用
        scheduler->PerformAppInit(appInfo);
        break;
        }
    void AbilityScheduler::PerformAppInit(const AppInfo &appInfo)
    {
        // 定义lambda函数表达式
        auto task = [this, appInfo] {
            // 执行AbilityThread::PerformAppInit
            scheduler_.PerformAppInit(appInfo);
        };
        // 将函数表达式推入栈,执行函数
        eventHandler_.PostTask(task);
    }
    ```
    6. `AbilityThread::PerformAppInit`函数
    ``` C++
    void AbilityThread::PerformAppInit(const AppInfo &appInfo)
    {
        HILOG_INFO(HILOG_MODULE_APP, "Start app init");
        // 检查入参
        if ((appInfo.bundleName.empty()) || (appInfo.srcPath.empty())) {
            HILOG_ERROR(HILOG_MODULE_APP, "appInfo is null");
            return;
        }
        if (!appInfo.isNativeApp && (appInfo.moduleNames.size() != 1)) {
            HILOG_ERROR(HILOG_MODULE_APP, "only native app support multi hap");
            return;
        }
        AbilityEnvImpl::GetInstance().SetAppInfo(appInfo);
        AbilityThread::isNativeApp_ = appInfo.isNativeApp;
    
        for (const auto &module : appInfo.moduleNames) {
            std::string modulePath;
            // 判断是否是nativeAPP的函数，nativeAPP为用C++写的app
            if (appInfo.isNativeApp) {
                modulePath = appInfo.srcPath + PATH_SEPARATOR + module + LIB_PREFIX + module + LIB_SUFFIX;
                if (modulePath.size() > PATH_MAX) {
                    continue;
                }
                char realPath[PATH_MAX + 1] = { 0 };
                if (realpath(modulePath.c_str(), realPath) == nullptr) {
                    continue;
                }
                modulePath = realPath;
            } else {
                modulePath = ACE_LIB_PATH;
            }
            void *handle = dlopen(modulePath.c_str(), RTLD_NOW | RTLD_GLOBAL);
            if (handle == nullptr) {
                HILOG_ERROR(HILOG_MODULE_APP, "Fail to dlopen %{public}s, [%{public}s]", modulePath.c_str(), dlerror());
                exit(-1);
            }
            // 在list头部添加元素
            handle_.emplace_front(handle);
        }
    
        int ret = UtilsSetEnv(GetDataPath());
        HILOG_INFO(HILOG_MODULE_APP, "Set env ret: %{public}d, App init end", ret);
    }
    ```
    7. `LaunchPendingAbility`函数,下发`SCHEDULER_ABILITY_LIFECYCLE`命令
    ``` C++
    AbilityMsStatus AppRecord::LaunchPendingAbility()
    {
        if (pendingAbilityRecord_ != nullptr) {
            AbilityMsStatus status;
            // 判断是否需要gui
            if (pendingAbilityRecord_->GetAbilityInfo().abilityType == AbilityType::SERVICE) {
                status = pendingAbilityRecord_->InactiveAbility();
            } else {
                status = pendingAbilityRecord_->ActiveAbility();
            }
            // MissionRecord release AbilityRecord
            pendingAbilityRecord_ = nullptr;
            return status;
        }
        return AbilityMsStatus::LifeCycleStatus("pending ability not exsit");
    }
    AbilityMsStatus PageAbilityRecord::ActiveAbility()
    {
        // 判断是否状态已经是STATE_ACTIVE
        if (currentState_ == STATE_ACTIVE) {
            return AbilityMsStatus::LifeCycleStatus("current state is already active when active");
        }
        // 判断appRecord_是否为空指针
        if (appRecord_ == nullptr) {
            return AbilityMsStatus::AppTransanctStatus("app record not exsit");
        }
        // 生命周期变更
        TransactionState state = {token_, STATE_ACTIVE};
        auto status = appRecord_->AbilityTransaction(state, want_, abilityInfo_.abilityType);
        AdapterFree(want_.sid);
        return status;
    }
    // 生命周期切换
    AbilityMsStatus AppRecord::AbilityTransaction(const TransactionState &state,
    const Want &want, AbilityType abilityType) const
    {
        // 进行跨进程通信
        if (abilityThreadClient_ != nullptr) {
            return abilityThreadClient_->AbilityTransaction(state, want, abilityType);
        }
        return AbilityMsStatus::AppTransanctStatus("life cycle ability thread client not exsit");
    }
    AbilityMsStatus AbilityThreadClient::AbilityTransaction(const TransactionState &state,
    const Want &want, AbilityType abilityType) const
    {
        PRINTD("AbilityThreadClient", "start");
        // 构建跨进程对象信息
        IpcIo req;
        char data[IPC_IO_DATA_MAX];
        IpcIoInit(&req, data, IPC_IO_DATA_MAX, MAX_OBJECTS);
        IpcIoPushInt32(&req, state.state);
        IpcIoPushUint64(&req, state.token);
        IpcIoPushInt32(&req, abilityType);
        if (!SerializeWant(&req, &want)) {
            return AbilityMsStatus::AppTransanctStatus("SerializeWant failed");
        }
        // 下发SCHEDULER_ABILITY_LIFECYCLE
        int32_t ret = Transact(nullptr, svcIdentity_, SCHEDULER_ABILITY_LIFECYCLE, &req,
            nullptr, LITEIPC_FLAG_ONEWAY, nullptr);
        if (ret != LITEIPC_OK) {
            return AbilityMsStatus::AppTransanctStatus("lifecycle ipc error");
        }
        return AbilityMsStatus::Ok();
    }
    ```
    8. 处理`SCHEDULER_ABILITY_LIFECYCLE`
    ``` C++
    case SCHEDULER_ABILITY_LIFECYCLE: {
        // 从跨进程对象中拿到需要的数据
        int state = IpcIoPopInt32(data);
        uint64_t token = IpcIoPopUint64(data);
        int abilityType = IpcIoPopInt32(data);
        Want want = { nullptr, nullptr, nullptr, 0 };
        // 反序列化want
        if (!DeserializeWant(&want, data)) {
            result = SERIALIZE_ERROR;
            break;
        }
        scheduler->PerformTransactAbilityState(want, state, token, abilityType);
        break;
    }
    void AbilityScheduler::PerformTransactAbilityState(const Want &want, int state, uint64_t token, int abilityType)
    {
        // lambda构建函数，并清除want
        auto task = [this, want, state, token, abilityType] {
            scheduler_.PerformTransactAbilityState(want, state, token, abilityType);
            ClearWant(const_cast<Want *>(&want));
        };
        eventHandler_.PostTask(task);
    }
    void AbilityThread::PerformTransactAbilityState(const Want &want, int state, uint64_t token, int abilityType)
    {
        HILOG_INFO(HILOG_MODULE_APP, "perform transact ability state to [%{public}d]", state);
        Ability *ability = nullptr;
        auto iter = abilities_.find(token);
        // 没找到实例
        if ((iter == abilities_.end()) || (iter->second == nullptr)) {
            // want中没有元素
            if (want.element == nullptr) {
                HILOG_ERROR(HILOG_MODULE_APP, "element name is null, fail to load ability");
                AbilityMsClient::GetInstance().SchedulerLifecycleDone(token, STATE_INITIAL);
                return;
            }
            // 如果是native app，abilityName从解包中取得，js应用名字固定
            auto abilityName = isNativeApp_ ? want.element->abilityName : ACE_ABILITY_NAME;
            // 通过name来获取实例
            ability = AbilityLoader::GetInstance().GetAbilityByName(abilityName);
            if (ability == nullptr) {
                HILOG_ERROR(HILOG_MODULE_APP, "fail to load ability: %{public}s", abilityName);
                AbilityMsClient::GetInstance().SchedulerLifecycleDone(token, STATE_INITIAL);
                return;
            }
            HILOG_INFO(HILOG_MODULE_APP, "Create ability success [%{public}s]", want.element->abilityName);

            // Only page ability need to init display
    #ifdef ABILITY_WINDOW_SUPPORT
            // 如果type为page时，初始化uiTask
            if (abilityType == PAGE) {
                InitUITaskEnv();
            }
    #endif
            // 初始化app的相关信息
            ability->Init(token, abilityType, AbilityThread::isNativeApp_);
            // map中记录新建的app信息
            abilities_[token] = ability;
        } else {
            // 直接找到实例
            ability = iter->second;
        }
    
        // 生命周期函数切换
        if (ability->GetState() != state) {
            HandleLifecycleTransaction(*ability, want, state);
        }
    
        HILOG_INFO(HILOG_MODULE_APP, "perform transact ability state done [%{public}d]", ability->GetState());
        // 通知服务端生命周期切换结束，下发ABILITY_TRANSACTION_DONE
        AbilityMsClient::GetInstance().SchedulerLifecycleDone(token, ability->GetState());
        // page启动成功
        if (ability->GetState() == STATE_ACTIVE) {
            StartAbilityCallback(want);
        }
    
        // app停止服务，擦除数据，释放对象
        if (ability->GetState() == STATE_INITIAL) {
            abilities_.erase(token);
            delete ability;
        }
    }
    ```
## Js应用启动
- JS应用时`PerformTransactAbilityState`中的`ability`的类型为`AceAbility`
