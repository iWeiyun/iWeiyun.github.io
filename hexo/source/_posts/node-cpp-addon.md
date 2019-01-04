---
title: Node.js C++ Addon应用实践
date: 2019-01-04 11:24:43
author: farren
tags: node,nan,n-api
---



近期项目中尝试使用Electron来实现跨平台桌面客户端。由于Node.js支持c++实现native addon，将c++接口封装供js调用，我们考虑将已经被多平台使用的SDK作为native扩展库引入Electron工程。

Node.js有两套API可以供c++侧选择来实现v8接口的封装，NAN和N-API。NAN是受到广泛认可（包括Node官方）的三方接口层，内部兼容各种Node版本v8接口的变动，对外提供稳定的API，免去长期以来c++开发者升级node版本需要频繁兼容c++接口的苦恼；N-API则是Node官方自行推出的新接口层，并非取代原有接口，而是旨在提供更稳定更贴近c风格的API，同样是为了避免c++开发者频繁改动，更利于生态圈的发展。由于我们在启动Electron版本的计划时，Electron的稳定版本使用的8.9.x版本的Node.js，该版本的官方文档中，N-API还处于实验阶段，权衡之后，我们在几个重要的SDK中使用成熟稳定的NAN来为我们electron桌面端提供native封装模块；后续一些新增的小模块，则尝试使用N-API来实现。

​	

----



### c++ addon的本质

在编写自己的addon模块之前，可以随便拿一个.node文件来探查一下它的本质究竟是什么。直接查看一个.node文件的二进制数据，结果如下

```
CFFAEDFE 07000001 03000000 08000000 0D000000 30070000 85800100 00000000 19000000 C8020000 5F5F5445 58540000 00000000 00000000 00000000 00000000 00D01D00 00000000 00000000 00000000 00D01D00 00000000 07000000 05000000 08000000 ...
```

可以看到文件头标识有0xCFFAEDFE，根据 https://opensource.apple.com/source/xnu/xnu-792/EXTERNAL_HEADERS/mach-o/loader.h 头文件中的定义

```cpp
/* Constant for the magic field of the mach_header_64 (64-bit architectures) */
#define MH_MAGIC_64	0xfeedfacf	/* the 64-bit mach magic number */
#define MH_CIGAM_64	NXSwapInt(MH_MAGIC_64)   //0xcffaedfe （little-endian）
```

这串标识是苹果Mach-O文件在64位机器上的魔数，表明.node文件在mac平台上是一个Mach-O文件，我们可以通过lipo/otool/nm等工具对其结构进行查看。

```sh
$ otool -hv DemoSDK.node
Mach header
      magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
MH_MAGIC_64  X86_64        ALL  0x00      BUNDLE    23       4088   NOUNDEFS DYLDLINK TWOLEVEL WEAK_DEFINES BINDS_TO_WEAK
```

.node文件mach header中的fileType是MH_BUNDLE，也即.node文件是一个Bundle类型的动态库。一般此类动态库用于在运行时加载，允许被作为插件动态扩展程序的行为。Bundle类型动态库的后缀名可以自行指定，生成的库若不携带资源，也可以选择不生成类似framework那样的目录结构，而只输出一个二进制文件（相当于只能显式加载的.dylib）。这就是.node模块的本质，同样在windows上，.node模块实质就是一个dll。至此，mac开发者似乎都嗅到了一股熟悉的味道。

```js
// Loads a module at the given file path. Returns that module's
// `exports` property.
Module.prototype.require = function(path) {
  assert(path, 'missing path');
  assert(typeof path === 'string', 'path must be a string');
  return Module._load(path, this, /* isMain */ false);
};

// Given a file name, pass it to the proper extension handler.
Module.prototype.load = function(filename) {
  ...
  assert(!this.loaded);
  this.filename = filename;
  this.paths = Module._nodeModulePaths(path.dirname(filename));

  var extension = path.extname(filename) || '.js';
  if (!Module._extensions[extension]) extension = '.js';
  Module._extensions[extension](this, filename);
  this.loaded = true;
	...
};

//Native extension for .node
Module._extensions['.node'] = function(module, filename) {
  return process.dlopen(module, path._makeLong(filename));
};
```

在node源码中找到require()方法的实现，require()调用了\_load()方法，\_load()方法检查该模块是否已存在缓存，如果需要执行载入，则调用真正的加载方法load()。这表明一个c++模块在整个node进程中的引入是全局唯一的，node底层会对每一个导入的c++模块进行缓存，这是高效且合理的。Module.load()方法里面主要对加载路径进行了处理。对于c++ 模块来说，会执行Module.\_extensions['.node'] 的闭包，即调用process.dlopen来动态加载模块。

```c
// DLOpen is process.dlopen(module, filename).
// Used to load 'module.node' dynamically shared objects.
static void DLOpen(const FunctionCallbackInfo<Value>& args) {
  Environment* env = Environment::GetCurrent(args);
  uv_lib_t lib;
 	...
  Local<Object> module = args[0]->ToObject(env->isolate());  // Cast
  node::Utf8Value filename(env->isolate(), args[1]);  // Cast
  const bool is_dlopen_error = uv_dlopen(*filename, &lib);
	...
}
```

在node.cc文件中可以看到，process.dlopen对应的c++函数为DLOpen()，这个是Node程序一启动就注册好的。DLOpen()函数实际上是调用了libuv库的uv_dlopen()方法。libuv库是个跨平台库，其unix版本代码调用了系统函数dlopen() 。顺带一提，在windows上，uv_dlopen也是调用了系统函数LoadLibraryExW()来实现c++模块的导入。

​	

---



### 用node-gyp构建c++ addon

构建一个c++ addon模块，一般是通过node-gyp命令行工具来执行。其实早期是用另外一款工具的，不过过去的就让他过去吧...构建c++addon时，node-gyp会根据binding.gyp配置文件调用各平台上的编译工具集来进行编译，mac上是Xcode Command Line Tools，windows上则是Visual Studio。
​	
不论是window还是mac平台上，在安装了Node环境后，都可以方便地通过npm来全局安装node-gyp

```sh
$ npm install -g node-gyp
```

node-gyp 的常用命令如下：

```sh
$ cd node-addon-root-directory
$ node-gyp clean configure build --debug --silly  #node
$ node-gyp clean configure build --target=2.0.2 --arch=x64 --dist-url=https://atom.io/download/atom-shell  #electron
```

clean: 用于清空build缓存

configure: 根据binding.gyp文件进行编译链接相关的配置

build: 则启动构建过程

rebuild：包含了clean configure build 三个命令的集合操作

比较常见的选项：

—debug: 表明目标将构建为debug版本， 默认为release版本

—silly: 输出全部构建日志详情，包括所有编译链接选项和进度等

—target —arch —diet-url: 用于指定一个具体的版本的node进行编译，若要指定electron对应的node版本可以加上这些参数

—msvs_version 用于windows上指定vs版本

node-gyp的其他命令及选项可以参考 https://github.com/nodejs/node-gyp

binding.gyp文件的结构很简单，其实就是json格式的配置文件。一般实际工程中涉及的配置大体是下面这个样子：

```json
{
    "targets": [
        {
            "target_name": "DemoSDK",
            "sources": [
                "DemoSDK4Node.cpp", 
				...
                ],
            "include_dirs": [
                "<!(node -e \"require('nan')\")",
                "DemoSDK/src/",
                ...
            ],
            "libraries": [
                "../Libraries/mac/lib.a",
                "-F../Libraries/mac -framework DemoLib",
                ...
                ], 
            "defines": [
                ...
            ],
            'conditions': [
                ['OS=="mac"', {
                    'configurations': {
                        'Debug': {
                            "xcode_settings": {
                                "GCC_OPTIMIZATION_LEVEL": "0",
                                "DEPLOYMENT_POSTPROCESSING": "NO",
                                "GCC_GENERATE_DEBUGGING_SYMBOLS": "YES",
                				"GCC_SYMBOLS_PRIVATE_EXTERN" : "YES",
                                "DEBUG_INFORMATION_FORMAT": "dwarf",
                                "GCC_ENABLE_CPP_RTTI": "YES",
                                "GCC_ENABLE_CPP_EXCEPTIONS": "YES",
                                "OTHER_CPLUSPLUSFLAGS" : [ '-ObjC++', "-std=c++11", "-stdlib=libc++",  "-fobjc-arc", '-fvisibility=hidden'],
                                "OTHER_LDFLAGS": [ "-stdlib=libc++"],
                                "MACOSX_DEPLOYMENT_TARGET": "10.10"
                            }   
                        },
                        'Release': {
                            "xcode_settings": {
                                "GCC_OPTIMIZATION_LEVEL": "s",
                                "COPY_PHASE_STRIP": "YES",
                                "DEAD_CODE_STRIPPING": "YES",
                                "DEPLOYMENT_POSTPROCESSING": "YES",
                                "GCC_GENERATE_DEBUGGING_SYMBOLS": "YES",
        						"GCC_SYMBOLS_PRIVATE_EXTERN" : "YES",
                                "STRIP_INSTALLED_PRODUCT": "YES",
                                "STRIP_STYLE": "non-global",
                                "DEBUG_INFORMATION_FORMAT": "dwarf-with-dsym",
                                "LD_GENERATE_MAP_FILE" : "YES",
                                "GCC_ENABLE_CPP_RTTI": "YES",
                                "GCC_ENABLE_CPP_EXCEPTIONS": "YES",
                                "OTHER_CPLUSPLUSFLAGS" : [ '-ObjC++', "-std=c++11", "-stdlib=libc++",  "-fobjc-arc", '-fvisibility=hidden'],
                                "OTHER_LDFLAGS": [ "-stdlib=libc++"],
                                "MACOSX_DEPLOYMENT_TARGET": "10.10"
                            }   
                        }
                    },
                }],
                ['OS=="win"', {
                    'configurations': {
		                'Debug': {
		                    "msvs_settings": {
                                "VCCLCompilerTool": {
                                    "RuntimeLibrary": "3", 
			                        "RuntimeTypeInfo": "true"
                                },
		                    },
		                },
		                'Release': {
                            "msvs_settings": {
                                "VCCLCompilerTool": {
                                    "RuntimeLibrary": "2", 
			                        "RuntimeTypeInfo": "true"
                                },
                            },
		                },
	                }
                }]
            ] 
        },
		{
    		... #other targets
		}
    ]
}
```

"target"： binding.gyp允许你一次配置若干个addon，每一个addon均是一个target。对于一个target，必须要指定"target_name"，最终生成的addon会被命名为target_name.node。

"sources"：指定了需要编译的所有源文件。

"include_dirs"：指定头文件查找目录。是不是发现上面的include_dirs下有一行奇怪的东西 "<!(node -e \"require('nan')\")"，这是通过npm安装nan时自动写入的，<!(cmd)或<!@(cmd)实际上是在执行括号内的shell命令，将得到的nan路径"node_modules/nan"返回。

```sh
$ node -e "require('nan')"
node_modules/nan
```

"defines"：指定预处理宏。

"libraries"：指定需要链接的库，对于.a、.dylib这类非bundle结构形式的链接库，直接设置相对路径即可；对于具有完整bundle目录结构的framework，则可以通过"-F 'Framework Path' -framework 'Framework Name' "的方式来找到

"conditions"：允许你设置各种判断条件，它其实可以出现在整个.gyp文件的任何地方，并且当.gyp文件被加载后就会优先对内部的条件语句进行处理。这里主要用于区分系统平台，以便根据平台分别对xcode和vs进行相关设定

"configurations"：分别可以对debug版本和release版本进行编译选项的设定

其他gyp配置文件的具体格式可以参考gyp的官方文档： https://gyp.gsrc.io/docs/UserDocumentation.md ，不过很多找不到的还请努力谷歌。

​	

----



### c++ addon的编写

#### NAN的使用

先来说说NAN的方式编写addon。NAN的引入十分方便，根据nan的官方文档，只需要通过npm在工程目录下安装即可

```sh
npm init
npm install --save nan
```

Node.js c++ addon和nan的基本用法和封装示例均可以在 https://github.com/nodejs/node-addon-examples 和Node.js官方文档中找到，没有比这些更好的入门参考了。根据这些例子和文档，基本可以搞清楚v8引擎中类似isolate、context、handlescope、local/persistent handle的概念，基本上是必看的。这些例子就不贴出来了，下面提一下我们在应用中遇到的一些问题及处理方法。



#### 通过libuv异步回调js主线程



node.js是单线程的，网络及数据库异步请求的回包，需要在v8引擎主线程才能执行js回调方法。v8引擎底层依赖libuv实现了异步事件的循环和分发。libuv主要维护了一系列event loop，js代码均执行在默认主循环uv_default_loop中，native addon执行js回调都需要抛到该loop中执行，并且避免在uv_default_loop中执行耗时操作。
​	
因此，需要封装一个单独的类用于将所有异步任务传递给uv_default_loop。可以实现如下：

```cpp
struct AsyncHandleData {
    std::list<std::function<void()>> taskList;
    std::recursive_mutex mtx;
};

class V8TaskService {
public:
    V8TaskService() {
        _async.data = new AsyncHandleData;
        uv_async_init(uv_default_loop(), &_async, [](uv_async_t *handle){
            AsyncHandleData *handleData = (AsyncHandleData *)handle->data;
            std::list<std::function<void()>> excuteList;

            {
                std::unique_lock<std::recursive_mutex> lock(handleData->mtx);
                while (handleData->taskList.size() > 0) {
                    excuteList.push_back(handleData->taskList.front());
                    handleData->taskList.pop_front();
                }
            }

            while (excuteList.size() > 0) {
                excuteList.front()();
                excuteList.pop_front();
            }
        });
    }

    ~V8TaskService() {
        uv_close((uv_handle_t *)&_async, nullptr);
        delete (AsyncHandleData *)_async.data;
    }

    template<typename Func, typename... Args>
    void defaultLoopAsync(Func&& f, Args&&... args) {
        
        std::function<void()> task = [=]() {
            f(args...);
        };
        AsyncHandleData *handleData = (AsyncHandleData *)_async.data;
        {
            std::unique_lock<std::recursive_mutex> lock(handleData->mtx);
            handleData->taskList.push_back(task);
        }
        uv_async_send(&_async);
    }

private:
    uv_async_t _async;
};
```

首先，在V8TaskService的构造方法中创建一个uv_async_t类型的异步句柄，该句柄大概的结构如下。 

```c
//libuv - uv.h
struct uv_async_t {
  /* public */                                                                
  void* data;       //携带数据
  /* read-only */    
  uv_loop_t* loop;    //绑定执行loop
  uv_handle_type type;  
  /* private */   
  uv_async_cb async_cb;     
  int pending;  
  ...
};
```

句柄的data变量是唯一公开可以读写的数据字段，用于在不同循环间传递数据。在_async中，我们写入一个队列，存储native层回来的异步操作。
​	
接着调用uv_async_init()方法分别将js的主循环uv_default_loop和传入的callback绑定到句柄_async的loop和async_cb变量上。
​	
当有异步任务回来时，通过成员方法defaultLoopAsync()来切到js的主线程。该方法往_async的data里写入新的数据，并执行uv_async_send()方法，通知libuv的eventloop在主循环处理。

```c
//libuv - async.c
int uv_async_send(uv_async_t* handle) {

  if (ACCESS_ONCE(int, handle->pending) != 0)
    return 0;

  if (cmpxchgi(&handle->pending, 0, 1) == 0)
    uv__async_send(handle->loop);

  return 0;
}
```

由于网络和数据库层异步回调可能非常频繁，短时间内多次调用uv_async_send()方法，并不会获得相应次数的执行机会。查看uv_async_send()接口的实现，当handle本身的pending值还是非0值，就表明之前的发送事件并未得到处理，因此不会得到机会执行uv__async_send()这个内部接口来驱动目标loop。就因为这个机制，只能保证若有N个异步事件发送，至少能得到一次执行async_cb回调的机会。因为，我们需要自己维护一个队列，来保证每次触发时，native内所有发送的回调都能得到执行。



#### 异步js回调与Persistent Handle



解决了addon异步回调到js主线程的问题，还需要考虑的就是在执行js传递下来的的callback函数了。addon所有注册给js调用的接口，都只有一个v8::FunctionCallbackInfo\<v8::Value\>类型的参数，所有js实际传递的参数、返回值的引用及JS对象本身的引用都被包含在内。

js调用接口时传递的参数通过info[i]即可依次获取，一般进行类型检查后，即可转换成对应的Local Handle对象。前面提到过，在创建Local Handle对象前，需要声明一个HandleScope，用来管理这些Local Handle对象的生命周期。当HandleScope离开函数作用域而被销毁时，会回收所有Local Handle对象，自然包括js传下来的v8::Function类型的回调函数。这里需要用一个Persistent Handle对象来延长该闭包的生命周期，v8::Persistent\<T>默认是NonCopyablePersistentTraits，禁用了拷贝构造，无法直接被lambda值捕获，只能传递v8::Persistent\<v8::Function\>的指针；同时，NonCopyablePersistentTraits析构时不会主动为内存中对象染色，所以这里还需要传递一个deleter给智能指针，在析构时调用Reset()方法，便于v8的GC适时销毁这个persistent对象。

```cpp
NAN_METHOD(DemoSDK::someTimeConsumingMethod) {

    if(info[0]->IsNullOrUndefined() || !info[1]->IsFunction()) {
        Nan::ThrowError("invalid parameters");
    }
    Nan::HandleScope scope;
    std::string req(BIN(*v8::String::Utf8Value(info[0])));
    v8::Local<v8::Function> callback = v8::Local<v8::Function>::Cast(info[1]);
    auto persistentCallback = std::shared_ptr<Nan::Persistent<v8::Function> >(new Nan::Persistent<v8::Function>(callback),[](Nan::Persistent<v8::Function> * callback)  {
        callback->Reset();
        delete callback;
        callback = nullptr;
    });
    RealTimeConsumingMethod(req, [=] (RspListType* rspList, int error) {
        service->defaultLoopAsync([=](){
            Nan::HandleScope scope;
            std::vector<v8::Local<v8::Value>> argv;
            argv.emplace_back(Nan::New(errorcode));

            v8::Local<v8::Array> rspArray = Nan::New<v8::Array>();
            int index = 0;
            for (auto item : *rspList) {
            	rspArray->Set(index++, V8Item::newInstance(item));
            }
            argv.emplace_back(rspArray);
            argv.emplace_back(Nan::New(error));
            
            persistentCallback->Get(v8::Isolate::GetCurrent())->Call(Nan::GetCurrentContext()->Global(), argv.size(), argv.data());
        });
    });
}
```



#### Local Handle作为返回值



我们知道Local Handle的生命周期被容器Handle Scope管理，意味着当我们需要将Local Handle作为返回值传递时，由于Handle Scope被析构，内部所有Local Handle包括当前的返回值都随时会被GC清理。
​	
但是，就如下文会提到为自定义数据结构进行封装时，不可避免地会将Local Handle作为返回值处理。这个时候，我们需要用一种EscapableHandleScope来管理这些Local Handle，这种Scope允许Local Handle“逃离”自己的管理。通过Escape()方法，将要"逃离"的参数拷贝到一个封闭的scope中，之后当前的EscapableHandleScope仍然会标记其内部所有Local Handle继而被GC清理，实际返回刚才的副本。

```c++
v8::Local<v8::Value> V8Item::newInstance() {
	Nan::EscapableHandleScope scope;
	v8::Local<v8::Function> cons = Nan::New<v8::Function>(constructor);
	return scope.Escape(cons->NewInstance(Nan::GetCurrentContext()).ToLocalChecked());
}
```



#### Mac平台OC RunLoop与uv_default_loop



我们的addon中还引入了一些三方framework，在mac平台下有些framework有相当一部分模块使用Objective-C来实现，当我们提供这些三方库的Objective-C接口给js调用时，需要先用c++层包装oc接口，再进行v8层的封装。

不过，由于个别OC库的内部实现涉及通过GCD分发到Main Queue处理异步代码，libuv的事件系统并不会执行OC的MainRunLoop，从而会导致这类库无法使用。更改并维护三方库的代码并不是一个很好的方案，我们选择定时在libuv的uv_default_loop中手动插入RunLoop，使库中分发到主线程的异步回调得到执行的机会。RunLoop插入的时间间隔由addon层未完成的wns请求量来动态变化。为了避免卡住node.js的主线程，目前只能选择runMode:beforeDate:的方式来启动RunLoop，确保只执行一次，以处理所有GCD分发到Main Queue的任务。

```cpp
// WeiyunOCRunLoopContext.h
enum RunLoopFrequencyLevel {
    RunLoopFrequencyLevelLow = 0,
    RunLoopFrequencyLevelMedium,
    RunLoopFrequencyLevelHigh
};

class OCRunLoopContext {

public:
    OCRunLoopContext();
    ~OCRunLoopContext();
    void updateRunLoopFrequencyLevel(RunLoopFrequencyLevel level);

private:
    void initRunLoopContext();
    int delayForFrequencyLevel(RunLoopFrequencyLevel level);
    
    std::future<bool> _future;
    std::atomic<bool> _finish;
    std::atomic<RunLoopFrequencyLevel> _level;

    static std::shared_ptr<V8TaskService> _service;
    static std::vector<int> _delays;
};


// WeiyunOCRunLoopContext.mm
#define DEFAULT_OC_RUNLOOP_DELAY_HIGH    100
#define DEFAULT_OC_RUNLOOP_DELAY_MEDIUM  300
#define DEFAULT_OC_RUNLOOP_DELAY_LOW     500

std::shared_ptr<V8TaskService> OCRunLoopContext::_service = std::make_shared<V8TaskService>();
std::vector<int> OCRunLoopContext::_delays = { DEFAULT_OC_RUNLOOP_DELAY_LOW, DEFAULT_OC_RUNLOOP_DELAY_MEDIUM, DEFAULT_OC_RUNLOOP_DELAY_HIGH };

OCRunLoopContext::OCRunLoopContext(): _finish(false), _level(RunLoopFrequencyLevelHigh) {
    this->initRunLoopContext();
}

void OCRunLoopContext::initRunLoopContext() {

    _future = std::async(std::launch::async, [this](){
        while (true) {
            int loopDelay = _delays[_level.load(std::memory_order_relaxed)];
            std::this_thread::sleep_for(std::chrono::milliseconds(loopDelay));
            if (this->_finish.load(std::memory_order_relaxed)) {
                break;
            }
            _service->defaultLoopAsync([](){
                @autoreleasepool {
                    [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate date]];
                }
            });
        }
        return true;
    });
}

OCRunLoopContext::~OCRunLoopContext() {
    this->_finish.store(true, std::memory_order_relaxed);
    _future.get();
}

void OCRunLoopContext::updateRunLoopFrequencyLevel(RunLoopFrequencyLevel level) {
    this->_level.store(level, std::memory_order_relaxed);
}


```

​	

---



### N-API的尝试



项目中写过一些小的c++ addon模块，尝试使用了N-API的接口来实现。这种方式与Nan没有本质区别。要使用N-API接口，只需要引入node_api.h这个头文件即可。N-API接口和数据结构非常规范，将所有底层的v8原生数据结构封装起来，调用起来形式统一。比如要实现一个native的事件监控模块，内部有一个供js调用的EventMonitor类，并提供了添加注册监控的方法，该方法返回一个handler的id，代码可以有如下写法：

```c++
# NodeEventMonitor.h
class NodeEventMonitor {
public:
    static napi_value Init(napi_env env, napi_value exports);
    static void Destructor(napi_env env, void * nativeObject, void *finalize_hint);
private:
    explicit NodeEventMonitor();
    ~NodeEventMonitor();

    static napi_value New(napi_env env, napi_callback_info info);
    static napi_value RegisterMonitor(napi_env env, napi_callback_info info);
    static napi_ref constructor;
    napi_env env_;
    napi_ref wrapper_;
};
```

```c++
# NodeEventMonitor.mm

#define NAPI_CHECK_STATUS(x) assert(x == napi_ok);
#define DECLARE_NAPI_METHOD(name, func) { name, 0, func, 0, 0, 0, napi_default, 0 }

napi_ref NodeEventMonitor::constructor;
NodeEventMonitor::NodeEventMonitor(): env_(nullptr), wrapper_(nullptr) {
}

NodeEventMonitor::~NodeEventMonitor() {
    napi_delete_reference(env_, wrapper_);
}

void NodeEventMonitor::Destructor(napi_env env, void *nativeObject, void *finalize_hint) {
    reinterpret_cast<NodeEventMonitor *>(nativeObject)->~NodeEventMonitor();
}

napi_value NodeEventMonitor::Init(napi_env env, napi_value exports) {
    std::vector<napi_property_descriptor> properties = {
        DECLARE_NAPI_METHOD("registerMonitor", RegisterMonitor),
    };
    napi_value cons;
    NAPI_CHECK_STATUS(napi_define_class(env, "EventMonitor", NAPI_AUTO_LENGTH, New, nullptr, properties.size(), properties.data(), &cons));
    NAPI_CHECK_STATUS(napi_create_reference(env, cons, 1, &constructor));
    NAPI_CHECK_STATUS(napi_set_named_property(env, exports, "EventMonitor", cons));
    return exports;
}

napi_value NodeEventMonitor::New(napi_env env, napi_callback_info info) {

    napi_value target;
    NAPI_CHECK_STATUS(napi_get_new_target(env, info, &target));
    bool is_constructor = target != nullptr;

    if (is_constructor) {   // 'new NodeEventMonitor()'
        size_t argc = 1;
        napi_value args[1];
        napi_value js_this;
        NAPI_CHECK_STATUS(napi_get_cb_info(env, info, &argc, args, &js_this, nullptr));
        NodeEventMonitor *obj = new NodeEventMonitor();
        obj->env_ = env;
        NAPI_CHECK_STATUS(napi_wrap(env, js_this, reinterpret_cast<void *>(obj), NodeEventMonitor::Destructor, nullptr, &obj->wrapper_));
        return js_this;
    } else {
        size_t argc_ = 1;
        napi_value args[1];
        NAPI_CHECK_STATUS(napi_get_cb_info(env, info, &argc_, args, nullptr, nullptr));

        const size_t argc = 1;
        napi_value argv[argc] = { args[0] };
        napi_value cons;
        NAPI_CHECK_STATUS(napi_get_reference_value(env, constructor, &cons));

        napi_value instance;
        NAPI_CHECK_STATUS(napi_new_instance(env, cons, argc, argv, &instance));

        return instance;
    }
}

napi_value NodeEventMonitor::RegisterMonitor(napi_env env, napi_callback_info info) {

    size_t argc = 1;
    napi_value args[1];
    napi_value js_this;
    NAPI_CHECK_STATUS(napi_get_cb_info(env, info, &argc, args, &js_this, nullptr));

    napi_value callback = args[0];
    napi_ref callbackRef;
    NAPI_CHECK_STATUS(napi_create_reference(env, callback, 1, &callbackRef));

    uint32_t handlerId = [EventMonitor.defaultMonitor registerLocalMonitor:[=](int type) {

        napi_value result;

        napi_value argv[1];
        NAPI_CHECK_STATUS(napi_create_int32(env, type, argv));

        napi_value global;
        NAPI_CHECK_STATUS(napi_get_global(env, &global));

        napi_value callback;
        NAPI_CHECK_STATUS(napi_get_reference_value(env, callbackRef, &callback));

        NAPI_CHECK_STATUS(napi_call_function(env, global, callback, 1, argv, &result));
    }];

    napi_value retValue;
    NAPI_CHECK_STATUS(napi_create_uint32(env, handlerId, &retValue));

    return retValue;
}
```

大部分实现都参考自官方文档和例子，简单说明一下笔者的看法。首先看到NAPI_CHECK_STATUS宏定义，这里是笔者觉得不太习惯但又非常合理的地方：每一次N-API调用都会有一个napi_status类型的状态返回值，及时处理好每一次调用的可能发生的错误非常关键。所有node.js层传下来的参数、回调信息以及js对象指针都被保存在napi_callback_info中，这点和NAN和c++ Addon API的调用方式一致。napi_env笔者就理解为类似v8::Isolate和v8::Context这样的执行上下文环境，我们只要保存好并适时传递给N-API接口即可，无需过于关心具体代表的内容。napi_value封装了各种常见v8数据类型，是一个生命周期受napi_handle_scope控制的指针，类似于Local Handle的使用方式。那么如何将一个Local Handle持久化，笔者目前找到的方法是使用napi_ref类型来实现类似的持久化数据的效果，napi_ref可以为napi_value增加引用计数来延长其生命周期，也可以手动减少计数来使其标记等待GC，是否存在其他方式待后续工程中大范围使用时再深入研究一下。
​	
总体而言，N-API的接口简洁、规范，在该API稳定的node.js版本下可以考虑用于正式开发了，不过想要了解更好了解v8引擎的一些底层的实现和原理，同时可以考虑使用NAN和Node Addon API来加深理解。两种方式的开发效率就笔者个人而言，目前看来相差无几，当然在v8中的性能也自然没有差别。

​	

----



### c++ addon在mac平台的调试和编译优化

#### c++ addon的调试

mac平台上，若要调试被载入Electron或Node程序的c++ addon模块，就需要用到lldb调试工具。

```sh
# node
$ lldb node index.js
(lldb) target create "node"
Current executable set to 'node' (x86_64).
(lldb) settings set -- target.run-args  "index.js"
(lldb) run
...
```

```sh
# electron
$ lldb
(lldb) target create "./electron-app/node_modules/electron/dist/Electron.app"
Current executable set to './electron-app/node_modules/electron/dist/Electron.app' (x86_64).
(lldb) setting set -- target.run-args "./electron-app/xplatform/dist/"
(lldb) run
...
```

类似gdb，mac上的lldb十分强大。只需要在命令行中启动lldb，为其设置要执行的target app，再传入app启动所需的参数即可将lldb挂在到Node或Electron程序上。三条命令可以合并成一条，也可以分开执行。
​	
当然也可以用Xcode - Debug - Attach To Process将xcode挂到Electron进程中调试，这种方式笔者没有试过，只是提供一种想法。
​	
lldb具体的调试过程与xcode中调试ios或mac应用别无二致，一些命令和操作就不贴了，没啥意思，详情当然是参考lldb官网了 https://lldb.llvm.org/



#### c++ addon的编译选项优化

windows上编译链接相关的优化选项采用默认的方案即可，基本只需要指定运行时库类型。而在Mac平台上，不指定优化编译选项，会导致编译出来的release版本的addon模块变得非常大。这些优化选项就类似于xcode工程Build Setting里的设置，不过需要显式在.gyp中指定。上面贴出来的binding.gyp中已经包含了部分选项的指定，这里简单说明一些重要的选项：

```sh
"GCC_OPTIMIZATION_LEVEL": "s"
```

GCC_OPTIMIZATION_LEVEL：对应xcode工程Build Setting -> Optimization Level，.gyp中debug模式下此项设定为"0"，表示优化等级为None；release模式下此项为"s", 则优化等级为Fastest, Smallest

```sh
"DEPLOYMENT_POSTPROCESSING": "YES",
"COPY_PHASE_STRIP": "YES"，
"DEAD_CODE_STRIPPING": "YES",
"STRIP_INSTALLED_PRODUCT": "YES",
"STRIP_STYLE": "non-global"
```

DEPLOYMENT_POSTPROCESSING：开启所有strip开关，后面对各类不必要符号的strip操作能够执行

STRIP_STYLE：由于addon均认为是动态库，不能strip其需要重定位的全局符号，这里只能将类型设置为non-global

```sh
"GCC_GENERATE_DEBUGGING_SYMBOLS": "YES",
"DEBUG_INFORMATION_FORMAT": "dwarf-with-dsym"
```

"DEBUG_INFORMATION_FORMAT": 将调试符号存储到一个dSYM文件中，便于堆栈符号化

理论上，xcode工程里Build Setting内所有可配置选项均能够引入到.gyp文件中，需要的时候，可以从xcode工程里的project.pbxproj文件中参考对应的命令项填入。
​	
这是我们首次尝试应用Electron实现桌面客户端并引入、实现c++原生插件，部分方案是笔者尝试后给出的方案，能力有限难免存在缺陷，若有不足之处，恳请指出，谢谢。

​	

----



### 参考资料

https://nodejs.org/dist/latest-v8.x/docs/api/addons.html

https://v8.dev/docs

https://github.com/nodejs/nan

http://jayconrod.com/posts/55/a-tour-of-v8-garbage-collection

https://github.com/nodejs/node-addon-examples