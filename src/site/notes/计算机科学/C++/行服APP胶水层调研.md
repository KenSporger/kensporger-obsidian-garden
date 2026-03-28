---
{"dg-publish":true,"permalink":"/计算机科学/C++/行服APP胶水层调研/"}
---


## 方案选型

选择方案一的话，目前已有的是protovalue从native到JS的自动生成，但是函数调用还没有实现，需要写一个python脚本，自动生成node-addon-api的胶水层代码，主要是各种数据类型转换，参考tools/djifunproto/java。工作量大概至少一周。（不过看github是已经提供了自动生成脚本的，但迁移到CSDK，需要适配funcproto，还是有一定修改量的）


## nan风格

```cpp
#include <nan.h>

void Add(const Nan::FunctionCallbackInfo<v8::Value>& info) {
  // 获取上下文，类似JNI的ENV
  v8::Local<v8::Context> context = info.GetIsolate()->GetCurrentContext();

  if (info.Length() < 2) {
    Nan::ThrowTypeError("Wrong number of arguments");
    return;
  }

  if (!info[0]->IsNumber() || !info[1]->IsNumber()) {
    Nan::ThrowTypeError("Wrong arguments");
    return;
  }

  double arg0 = info[0]->NumberValue(context).FromJust();
  double arg1 = info[1]->NumberValue(context).FromJust();
  v8::Local<v8::Number> num = Nan::New(arg0 + arg1);

  info.GetReturnValue().Set(num);
}

void HelloWorld(const Nan::FunctionCallbackInfo<v8::Value>& info) {
  // Nan::New("world").ToLocalChecked() 创建一个world字符串
  // info.GetReturnValue().Set 设置函数的返回值
  info.GetReturnValue().Set(Nan::New("world").ToLocalChecked());
}

void Init(v8::Local<v8::Object> exports) {
  v8::Local<v8::Context> context =
      exports->GetCreationContext().ToLocalChecked();
  // 注册一个函数
  exports->Set(context,
              // 函数名是hello
               Nan::New("hello").ToLocalChecked(),
              //  将原生函数（通过 Nan::New<v8::FunctionTemplate>(HelloWorld) 创建）添加到 exports 对象上
               Nan::New<v8::FunctionTemplate>(HelloWorld)
                   ->GetFunction(context)
                   .ToLocalChecked());
  exports->Set(context,
              // 函数名是hello
               Nan::New("add").ToLocalChecked(),
               Nan::New<v8::FunctionTemplate>(Add)
                   ->GetFunction(context)
                   .ToLocalChecked());
}

// 注册一个原生插件, 插件名字叫“hello_test”
NODE_MODULE(test, Init)
```

虽然有一些C++的风格，但是`ToLocalChecked`、返回值和函数都通过`set`来设置的使用方式理解起来不那么直观。



## napi风格

```cpp
#include <assert.h>
#include <node_api.h>

static napi_value Method(napi_env env, napi_callback_info info) {
  napi_status status;
  napi_value world;
  status = napi_create_string_utf8(env, "world", 5, &world);
  assert(status == napi_ok);
  return world;
}

#define DECLARE_NAPI_METHOD(name, func)                                        \
  { name, 0, func, 0, 0, 0, napi_default, 0 }

static napi_value Init(napi_env env, napi_value exports) {
  napi_status status;
  napi_property_descriptor desc = DECLARE_NAPI_METHOD("hello", Method);
  status = napi_define_properties(env, exports, 1, &desc);
  assert(status == napi_ok);
  return exports;
}

NAPI_MODULE(NODE_GYP_MODULE_NAME, Init)
```

C语言风格，使用起来比较麻烦。



## node-addon-api

和Java JNI很像，使用起来比较简单，和JNI的区别在于，函数入参采用了类似JSON的解析方式，而JNI则是直接放在形参里的。

比如array转换：

```cpp
// C++函数
static void ArrayConsumer(const std::vector<int>& vec) {
  for (const auto& item : vec) {
    cout << item << endl;
  }
}

// js to native, array convert
static Napi::Value AcceptArrayBuffer(const Napi::CallbackInfo& info) {
  if (info.Length() != 1) {
    Napi::Error::New(info.Env(), "Expected exactly one argument")
        .ThrowAsJavaScriptException();
    return info.Env().Undefined();
  }
  if (!info[0].IsArrayBuffer()) {
    Napi::Error::New(info.Env(), "Expected an ArrayBuffer")
        .ThrowAsJavaScriptException();
    return info.Env().Undefined();
  }

  Napi::ArrayBuffer buf = info[0].As<Napi::ArrayBuffer>();
  auto p = reinterpret_cast<int32_t*>(buf.Data());
  std::vector<int32_t> vec(p, p + buf.ByteLength() / sizeof(int32_t));
  ArrayConsumer(vec);

  return info.Env().Undefined();
}
```



## [genepi](https://github.com/Geode-solutions/genepi)

作为N-API wrapper的自动生成工具。使用自动生成工具需要考虑支持的数据类型转换范围，比如是否支持callback，是否支持STL等等。

genepi 是非侵入型的，但是是在编译阶段生成，不像CSDK JNI一样是跑脚本的，所以源代码看不到。

另外像Int32Array似乎没有支持。

## node java插件

```javascript
const java = require("java");
java.classpath.push(".");

var test = java.import("com.test.Test");
test.SayHello();
```

看起来很方便？


### JNI内存传递

uint8向Integer对齐，所以字节话时转为int

C++ 回调回去的数据是DjiValue，调用Value的Serialization转成字节流，例如BufferMsg，转字节流时前面4个字节为长度，后面才是具体内容。

C++ 接收的数据是byte数组，利用key索引到具体Value，再反序列化读取。


## JS调用CSDK方案

### JS调用 DJIKeyManager

js和java之间的数据类型做映射。主要难点在于DJIKeyInfo的获取，以及对Param的转换。

思路是js类型先tojson，然后利用java反射找到对应的converter进行fromjson构造，再toDJIValue，这样能够调用JNIKeyValue，但是无法直接调用DJIKeyManager.set，因为这个是依赖具体类型的，相当于JS的类型要和JAVA的类型做绑定关系，但由于模板推导是编译期的，所以必然无法通过反射、map映射的方式展开，所以只能通过脚本生成代码的方式。

对于回调参数：在回调里先通过frombytes读取，再通过tojson转为json字符串，再转到js类型。

因为Converter是有限的，所以我们可以通过instanceof来进行转换。

这个方式的工作量在于：

+ 为四种JNIKeyValue接口编写反射json转换代码

+ csdk提供的key接口信息中要包含模块、key的字符串信息

+ 由于java层对于基础类型是直接做valueof，非json字符串，以及BufferConvert实际没有实现fromjson方法，因此有坑点。

总之，java在最初代码设计时并没有考虑给其他上层语言留桥接接口，因此导致适配难度较高。

**咨询了一下唐询，反射还是比较耗性能的，定义一个map会比这个好一点。**



### JS调用jni native



### 其他可能性

唐询说可以从react 的engine入手，直接实现调用C++。



## 框架知识

android上原生语言是java，所有安卓应用框架都基于它，行服采用的是react框架，框架使用的语言是js。

ios上原生语言是OC，CR800用的是flutter框架，语言是Dart。


# APP的main语言
{ #997b92}


前端团队用 JavaScript/React 开发手机 App，通过 JNI 桥接 Java，再调用 C++ so 库。

这种情况下，main 函数实际是在 Java 层（Android App），由 Android 的 Java 虚拟机启动。JavaScript 代码运行在 WebView 或 React Native 等 JavaScript 引擎中，通过 React Native 的 Native Module 机制或 WebView 的 JSBridge 调用到 Java。

同一个进程里怎么同时有 Java 和 JavaScript 的 runtime？

- 在 Android 上，一个进程可以同时包含 **Java VM** 和 **JavaScript 引擎**（如 V8、JavaScriptCore、Hermes）。
- 两者通过 **JNI** 或 **C API** 互相调用。
- 候选人实现的“胶水层”是在 **JS → Java 方向** 使用 **反射 + JSON 序列化** 做数据转换，然后通过 JNI 进 C++；反之亦然。