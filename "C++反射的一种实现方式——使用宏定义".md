---
title: "C++反射的一种实现方式——使用宏定义"
author: 一张狗
lastmod: 2019-07-06 03:29:17
date: 2019-05-30 16:27:59
tags: []
---


C++中没有java和C#等各种语言中支持的反射，但是我们可以使用各种方式来实现一个伪的反射，中心思想是持有一个全局的注册map(类名->构造函数)。创建对象的时候，传入类名，然后在其中找到构造函数，然后创建对象。

本篇文章使用宏定义来解决注册的问题。


# 一、注册辅助类

***ClassRegistry***：模板函数，用于data、module、contextdata的注册，其中的函数解析：

*create_object*：从RegistryMap里找到传入name对应的RegistryNode(RegistryNode保存了名字和构造函数)，调用构造函数返回。

*register_class*：用传入的name和constructor注册RegistryMap,只在Register的构造函数里面调用，后面会在ClassRegister<IData> DataRegister、ClassRegister<IModule> ModuleRegister、ClassRegister<IContextData> ContextDataRegister用到。RegistryMap里面的数据是从register_class这个方法插入数据进去的，后面会在IMPLEMENT_XXX中调用到这个。

*fill_name_array*：找到RegistryMap里面注册的name，插入传入参数。


# 二、使用到的宏定义


## 2.1 data

REGISTER_DATA：声明构造data_class的函数 __construct_##name##_data() ，其中调用了data_calss的构造函数；  
 声明获取class的get_##name，函数体的get_data从 sign_data_map里面获取到对应的IData
    
```
#define REGISTER_DATA(data_class, name)                                       \
  inline ::wmf::IData* __construct_##name##_data() { return new data_class; } \
  namespace wmf {                                                             \
  namespace internal {                                                        \
  inline data_class* get_##name() { return get_data<data_class>(#name); }     \
  }                                                                           \
  }  // wmf::internal
```

IMPLEMENT_DATA：调用DataRegister的构造函数。声明变量__##name##_module_register，这里会将输入的name和构造函数__construct_##name##_data注册到RegistryMap中；

```
#define IMPLEMENT_DATA(name)                                \
  ::wmf::internal::DataRegister __##name##_module_register( \
      #name, __construct_##name##_data)
```

使用：  
 在需要用到的.cpp文件的的.h文件的位置调用REGISTER_DATA,声明构造函数和获取data的get_xxx函数。  
 在每个service的cpp文件视线中调用IMPLEMENT_DATA，注入RegistryMap。  
 在每个service的cpp文件的InitInjection中，INJECT_DATA_MODULE_DEPENDENCY把这个词典注入到module中。


## 2.2 module

REGISTER_MODULE：声明__construct_#name##_module()，返回new module_class；  
 声明获取class的get_##name，函数体里面返回ModuleMap中保存的对象（cast_module从ModuleMap里面找到其对应的对象，如果找不到，则从RegisterMap里面找到其构造函数，并调用create_object之后插入ModuleMap，并返回新建的对象（RegisterMap里面的数据从IMPLEMENT_XXX来的））

```
#define REGISTER_MODULE(module_class, name)              \
  inline ::wmf::IModule* __construct_##name##_module() { \
    return new module_class;                             \
  }                                                      \
  namespace wmf {                                        \
  namespace internal {                                   \
  inline module_class* get_##name(::wmf::Context& ctx) { \
    return ctx.cast_module<module_class>(#name);         \
  }                                                      \
  }                                                      \
  }  // wmf::internal
```

IMPLEMENT_MODULE：声明__##name##_module_register变量，以插入RegistryMap。

```
#define IMPLEMENT_MODULE(name)                                \
  ::wmf::internal::ModuleRegister __##name##_module_register( \
      #name, __construct_##name##_module)
```

使用：

在新增module的.h文件最后调用REGISTER_MODULE声明了在IMPLEMENT_MODULE中会用到的构造函数，以及声明了从ModuleMap中获取其对象的get_xxx函数。  
 在service的最后调用IMPLEMENT_MODULE，把module注册到RegistryMap中。


## 2.3 context data

REGISTER_CONTEXT_DATA：声明__construct_##name##_context_data()，新建data_class；  
 声明获取class的get_##name，函数体里面通过name查找到ContextDataMap保存的名字签名对应的IContextData，转换为data_class返回。

```
#define REGISTER_CONTEXT_DATA(data_class, name)                     \
  inline ::wmf::IContextData* __construct_##name##_context_data() { \
    return new data_class;                                          \
  }                                                                 \
  namespace wmf {                                                   \
  namespace internal {                                              \
  inline data_class* get_##name(const ::wmf::Context& ctx) {        \
    return ctx.cast_context_data<data_class>(#name);                \
  }                                                                 \
  }                                                                 \
  }  // wmf::internal
```

IMPLEMENT_CONTEXT_DATA：声明__##name##_context_data变量，这里会将输入的name和构造函数__construct_##name##_context_data注册到RegistryMap中；

```
#define IMPLEMENT_CONTEXT_DATA(name)                            \
  ::wmf::internal::ContextDataRegister __##name##_context_data( \
      #name, __construct_##name##_context_data)
```


## 2.4 index_data

DECLARE_INDEX_DATA：N for name, VT for VersionIndex 类型。声明类型C为用类型VT组装，path、name、desc用N组装的VIAdaptor类型。

```
#define DECLARE_INDEX_DATA(VT, C, N)                                        \
  extern const char __index_##N##_path[];                                   \
  extern const char __index_##N##_name[];                                   \
  extern const char __index_##N##_desc[];                                   \
  typedef wmf::VIAdaptor<argument_type<void(VT)>::type, __index_##N##_path, \
                         __index_##N##_name, __index_##N##_desc>            \
      C
```

DEFINE_INDEX_DATA：N for name，这里是声明一堆string变量，用于data的path、name、desc。

```
#define DEFINE_INDEX_DATA(N)                        \
  const char __index_##N##_path[] = #N "_path";     \
  const char __index_##N##_name[] = #N "_name";     \
  const char __index_##N##_desc[] = #N "_desc";     \
  DEFINE_string(N##_path, "", "index " #N " path"); \
  DEFINE_string(N##_name, "", "index " #N " name"); \
  DEFINE_string(N##_desc, "index_" #N, "index " #N " desc")
```


## 2.5 injection

DEFINE_INJECTION：定义一个把object_ref变量设置为class_type*类型的传入变量的函数。

```
#define DEFINE_INJECTION(injection_name, class_type, object_ref) \
  void set_##injection_name(class_type* module) { object_ref = module; }
```

INJECT_OBJECT_OBJECT_DEPENDENCY：调用object_to这个对象的set_##injection_name方法，传入参数是object_from的引用。结合DEFINE_INJECTION就是把object_from设置到object_to这个对象里面。
    
```
#define INJECT_OBJECT_OBJECT_DEPENDENCY(injection_name, object_from, \
                                        object_to)                   \
  (object_to).set_##injection_name(&(object_from))
```

INJECT_MODULE_DEPENDENCY：在上下文context中找到module_from的变量，注入到同一个上下文的module_from里面。

```
#define INJECT_MODULE_DEPENDENCY(injection_point, context, module_from, \
                                 module_to)                             \
  ::wmf::internal::get_##module_to(context)->set_##injection_point(     \
      ::wmf::internal::get_##module_from(context));
```

INJECT_DATA_MODULE_DEPENDENCY：把data注入到通过上下文context获取的module_to中。

```
#define INJECT_DATA_MODULE_DEPENDENCY(injection_point, context, data, \
                                      module_to)                      \
  ::wmf::internal::get_##module_to(context)->set_##injection_point(   \
      ::wmf::internal::get_##data());
```

INJECT_MODULE_OBJECT_DEPENDENCY：通过上下文context获取的module_from注入到object_to中。

```
#define INJECT_MODULE_OBJECT_DEPENDENCY(injection_point, context, module_from, \
                                        object_to)                             \
  (object_to).set_##injection_point(                                           \
      ::wmf::internal::get_##module_from(context));
```

INJECT_OBJECT_MODULE_DEPENDENCY ：object_from注入到通过上下文获取的module_to中。

```
#define INJECT_OBJECT_MODULE_DEPENDENCY(injection_point, context, object_from, \
                                        module_to)                             \
  ::wmf::internal::get_##module_to(context)->set_##injection_point(            \
      &(object_from))
```
    
使用： 在上下文相关的session中调用INJECT_MODULE_DEPENDENCY、INJECT_DATA_MODULE_DEPENDENCY； INJECT_MODULE_DEPENDENCY用于把session相关的信息（比如session_docs、request、response）注入到module中，module的意思是这个请求需要过的模块名。 INJECT_DATA_MODULE_DEPENDENCY用于把data注入到module中。


# 三、总结


## 3.1 新增一个module

在新增module的.h文件最后调用 REGISTER_MODULE 声明了在 IMPLEMENT_MODULE 中会用到的构造函数，以及声明了从ModuleMap中获取其对象的get_xxx函数。  
 在service的最后调用IMPLEMENT_MODULE，把module注册到RegistryMap中。

在上下文相关的session中调用INJECT_MODULE_DEPENDENCY、INJECT_DATA_MODULE_DEPENDENCY；  
 INJECT_MODULE_DEPENDENCY用于把session相关的信息（比如session_docs、request、response）注入到module中，module的意思是这个请求需要过的模块名。  
 INJECT_DATA_MODULE_DEPENDENCY用于把data注入到module中。


## 3.2 代码回顾

ClassRegistry用于给第二项的一堆宏使用。module于类的映射关键在于RegistryMap，新增一个module的时候，服务会去RegistryMap里面找名字对应的构造函数。RegistryMap里面的数据是在IMPLEMENT_MODULE的时候注入进来的name和类的对应关系。配置文件里面配的是module的链条，比如需要过AModule,BModule，这时候就在init的时候把所有module都插进去，然后在schedule_impl里面调用每个module的run函数。


