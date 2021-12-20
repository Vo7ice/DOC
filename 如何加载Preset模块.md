# 如何加载`Preset`模块

## 以`ConsoleModule`为例

1. `JsAppEnvironment::LoadAceBuiltInModules`函数

   ```c++
   void JsAppEnvironment::LoadAceBuiltInModules() const
   {
       ConsoleModule::Load();
       RenderModule::Load();
       RequireModule::Load();
       FeaAbilityModule::Load();
       JsTestModule::Load();
       TimersModule::Load();
       PerformaceProfilerModule::Load();
       AceVersionModule::Load();
       IntlControlModule::Load();
   }
   ```

   

2. `ConsoleModule::Load()`函数

   - 主要实现了`confole.log/info/debug/warn/error`函数的绑定

   ```c++
   static void Load()
   {
       ConsoleModule consoleModule;
       consoleModule.Init();
   }
   void ConsoleModule::Init()
   {
       const char * const debug = "debug";
       const char * const info = "info";
       const char * const warn = "warn";
       const char * const log = "log";
       const char * const error = "error";
       CreateNamedFunction(debug, LogDebug);
       CreateNamedFunction(info, LogInfo);
       CreateNamedFunction(warn, LogWarn);
       CreateNamedFunction(log, Log);
       CreateNamedFunction(error, LogError);
   }
   // PresetModule为preset模块的基类,提供jerry函数绑定入口,其中module_为预置的模块名字,funcName为JS的函数名，handler为C/C++函数实现
   // 对应console模块为module_:console,funcName:log,js函数全程为console.log, handler:Log
   void PresetModule::CreateNamedFunction(const char * const funcName, jerry_external_handler_t handler)
   {
       if (funcName == nullptr) {
           return;
       }
       JerrySetFuncProperty(module_, funcName, handler);
   }
   // js_fwk_common为jerry绑定实现的函数实现层，提供C++需要的jerry绑定的接口实现
   void JerrySetFuncProperty(jerry_value_t object, const char * const name, jerry_external_handler_t handler)
   {
       if (name == nullptr || !strlen(name)) {
           HILOG_ERROR(HILOG_MODULE_ACE, "Failed to set function property cause by empty name.");
           return;
       }
   
       if (handler == nullptr) {
           HILOG_ERROR(HILOG_MODULE_ACE, "Failed to set function property cause by empty handler.");
           return;
       }
   
       jerry_value_t func = jerry_create_external_function(handler);
       jerryx_set_property_str(object, name, func);
       jerry_release_value(func);
   }
   ```

3. C/C++函数具体实现

   ```c++
   jerry_value_t ConsoleModule::Log(const jerry_value_t func,
                                    const jerry_value_t context,
                                    const jerry_value_t* args,
                                    const jerry_length_t length)
   {
   #if DISABLED(CONSOLE_LOG_OUTPUT)
       return UNDEFINED;
   #else
       return LogNative(LOG_LEVEL_NONE, func, context, args, length);
   #endif // DISABLED(CONSOLE_LOG_OUTPUT)
   }
   
   jerry_value_t LogNative(const LogLevel logLevel,
                           const jerry_value_t func,
                           const jerry_value_t context,
                           const jerry_value_t *args,
                           const jerry_length_t argc)
   {
       (void)func;    /* unused */
       (void)context; /* unused */
   
       // print out log level if needed
       LogOutLevel(logLevel);
   
       const char * const nullStr = "\\u0000";
       jerry_value_t retVal = jerry_create_undefined();
   
       // 先转换标识符，然后拼接成完整的字符串输出到buffer中
       for (jerry_length_t argIndex = 0; argIndex < argc; argIndex++) {
           jerry_value_t strVal;
   
           if (jerry_value_is_symbol(args[argIndex])) {
               strVal = jerry_get_symbol_descriptive_string(args[argIndex]);
           } else {
               strVal = jerry_value_to_string(args[argIndex]);
           }
   
           if (jerry_value_is_error(strVal)) {
               /* There is no need to free the undefined value. */
               retVal = strVal;
               break;
           }
   
           jerry_length_t length = jerry_get_utf8_string_length(strVal);
           jerry_length_t substrPos = 0;
           const uint16_t bufLength = LOG_BUFFER_SIZE;
           jerry_char_t substrBuf[bufLength] = {0};
   
           do {
               jerry_size_t substrSize =
                   jerry_substring_to_utf8_char_buffer(strVal, substrPos, length, substrBuf, bufLength - 1);
   
               jerry_char_t *bufEndPos = substrBuf + substrSize;
   
               /* Update start position by the number of utf-8 characters. */
               for (jerry_char_t *bufPos = substrBuf; bufPos < bufEndPos; bufPos++) {
                   /* Skip intermediate utf-8 octets. */
                   if ((*bufPos & 0xc0) != 0x80) {
                       substrPos++;
                   }
               }
   
               for (jerry_char_t *bufPos = substrBuf; bufPos < bufEndPos; bufPos++) {
                   char chr = static_cast<char>(*bufPos);
   
                   if (chr != '\0') {
                       LogChar(chr, logLevel);
                       continue;
                   }
   
                   for (jerry_size_t null_index = 0; nullStr[null_index] != '\0'; null_index++) {
                       LogChar(nullStr[null_index], logLevel);
                   }
               }
           } while (substrPos < length);
   
           jerry_release_value(strVal);
       }
       // output end
       LogChar('\n', logLevel, true);
       FlushOutput();
       return retVal;
   }
   // 输出到buffer里面
   void LogChar(char c, const LogLevel logLevel, bool endFlag)
   {
       logBuffer[logBufferIndex++] = c;
       if ((logBufferIndex == (LOG_BUFFER_SIZE - 1)) || (c == '\n')) {
           if ((c == '\n') && (logBufferIndex > 0)) {
               logBufferIndex--; // will trace out line separator after print the content out
           }
           logBuffer[logBufferIndex] = '\0';
           Output(logLevel, logBuffer, logBufferIndex);
           logBufferIndex = 0;
           if (c == '\n' || !endFlag) {
               // this is the newline during the console log, need to append the loglevel prefix,
               // example: console.log("aa\nbb");
   #if !defined(FEATURE_ACELITE_HI_LOG_PRINTF) && !defined(FEATURE_USER_MC_LOG_PRINTF)
               Output(logLevel, "\n", 1); // hilog will trace our the line separator directly
   #endif
               if (!endFlag) {
                   LogOutLevel(logLevel);
               }
           }
       }
   }
   // buffer输出到具体的日志框架中，鸿蒙为hilog
   void Output(const LogLevel logLevel, const char * const str, const uint8_t length)
   {
       if (str == nullptr) {
           return;
       }
       (void)length;
       Debugger::GetInstance().Output(str);
   #if defined(FEATURE_ACELITE_HI_LOG_PRINTF) || defined(FEATURE_USER_MC_LOG_PRINTF)
       OutputToHiLog(logLevel, str);
   #endif
   #ifdef TDD_ASSERTIONS
       // output to extra handler if it was set by test cases
       if (g_logOutputExtraHandler != nullptr) {
           g_logOutputExtraHandler(logLevel, str, length);
       }
   #endif // TDD_ASSERTIONS
   }
   ```

4. 以此类推可以按照此逻辑查看其他预置模块的绑定逻辑。

5. 打包后的`bundle.js`例子

   ```javascript
   (function () {
     return new ViewModel({
       render: function (vm) {
         var _vm = vm || this;
           return _c('swiper', {
             attrs : {index : 0,duration : 75},
             staticClass : ['container'], 
             on : {'change' : _vm.swiperChange}} , [
               _c('stack', {
                 staticClass : ['container']} , [
                   _c('text', {
                     attrs : {value : function() {return _vm.textValue}},
                     staticClass : ['pm25-name'],
                     staticStyle : {color : 16711680},
                     on : {'click' : _vm.click1}
                    })
               ])
         ]);
       },
       styleSheet: {
         classSelectors: {
           'pm25-value': {
             textAlign: 'center',
             fontSize: 38,
             color: 15794175,
             height: 454,
             width: 454,
             top: 235
           },
           'pm25-name': {
             textAlign: 'center',
             color: 10667170,
             width: 454,
             height: 50,
             top: 285
           }
         }
       },
      data: {textValue: 'Hello World'},
      onInit: function onInit() {},
      onShow: function onShow() {},
      openDetail: function openDetail() {},
      click3: function click3() {
        var sum = num + 1;
        this.textValue = 'Hello Ace';
      },
      click2: function click2() {
        this.click3();
      },
      click1: function click1() {
        this.click2();
      }
    });
   })();
   ```