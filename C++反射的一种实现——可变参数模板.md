---
title: "C++反射的一种实现——可变参数模板"
author: 一张狗
lastmod: 2019-07-04 16:06:50
date: 2019-06-30 22:44:31
tags: []
---


C++反射的实现需要解决的核心问题是怎么保持一个存储了类名到构造函数的方法。在上一篇我使用了宏定义在服务启动的时候注册，但是使用宏的方法避免不了每次创建类的时候都需要写一段宏去注册，那有没有什么办法可以绕过它呢？本篇文章使用可变参数模板，结合static变量在程序开始运行时创建而在程序结束时销毁，并且不依赖于对象的存在的特性。

创建一个单例工厂类，维护一个map。针对一个Class，定义一个注册类RegClass（可以使用可变参数模板来把所有类的注册类归为一个，这样一种类型的类只需要一个工厂），在注册类的构造函数中注册此类。定义一个类的时候，继承动态注册类，并传入子类作为参数模板：class **MyClass** : public DynamicCreate<**MyClass**>{};


## 动态对象创建工厂

ClassFactory实现了单例工厂的注册（Regist）和返回构造函数（Create）方法。Regist给动态对象创建器DynamicCreator自身持有的**静态Register变量构造的时候**调用。  
 ClassFactory的typename B构造函数的返回，Targs用于构造函数的入参。所有的类名到构造函数的映射都保存在creator_map_中，这里其实会根据传入的参数拼接构造函数，所有基类相同且入参类型相同的类都保存在同一个ClassFactory里面，由同一个creator_map_持有。  
 两个要点：

- Regist用于把类名和与构造函数插入本单例的creator_map_里面；
- Create用于通过类名获取构造函数，这个方法将会被后面的工厂方法使用；

```
 namespace reflect  
 {  
 //动态对象创建工厂  
 template<typename B, typename ...Targs>  
 class ClassFactory  
 {  
 public:  
     static ClassFactory* Instance() {  
         std::cout << "static ClassFactory* Instance()" << std::endl;  
         if (nullptr == factory_) {  
             factory_ = new ClassFactory();  
         }  
         return(factory_);  
     }  
  
     virtual ~ClassFactory(){};  
     //将DynamicCreator的CreateObject方法（CreateObject将被设置为实例创建方法），使用strTypeName为key，  
     //注册到ClassFactory中（通过把string类名和构造函数的函数指针插入creator_map_里面）。之后使用的时候可以通过这个类名获取相应类的构造函数。  
     bool Regist(const std::string& strTypeName, std::function<B *(Targs&&... args)> pFunc) {  
         std::cout << "bool ClassFactory::Regist(const std::string& strTypeName, std::function<B*(Targs... args)> pFunc)" << std::endl;  
         if (nullptr == pFunc) {  
             return false;  
         }  
         std::string strRealTypeName = strTypeName;  
  
         bool bReg = creator_map_.insert(std::make_pair(strRealTypeName, pFunc)).second;  
         std::cout << "creator_map_.size() =" << creator_map_.size() << std::endl;  
         return(bReg);  
     }  
  
     // 通过类名找到注册的构造函数（DynamicCreator的CreateObject方法），并使用args作为构造函数的入参，调动构造函数完成实例创建，返回。  
     //该方法用于被外部调用。  
     B *Create(const std::string& strTypeName, Targs&&... args) {  
         std::cout << "Base* ClassFactory::Create(const std::string& strTypeName, Targs... args)" << std::endl;  
         auto iter = creator_map_.find(strTypeName);  
         if (iter == creator_map_.end()) {  
             return nullptr;  
         }  
         return(iter->second(std::forward<Targs>(args)...));  
     }  
  
 private:  
     ClassFactory(){std::cout << "ClassFactory construct" << std::endl;};  
     static ClassFactory<B, Targs...>* factory_;     
     std::unordered_map<std::string, std::function<B *(Targs&&...)> > creator_map_;  
 };  
  
 template<typename B, typename ...Targs>  
 ClassFactory<B, Targs...>* ClassFactory<B, Targs...>::factory_ = nullptr;
 ```


## 动态对象创建器

要每次不需要主动声明就能在内存中持有一个从类名到构造函数的映射，我们需要使用static变量在类构造前先初始化的特性。在动态对象创建器中声明一个Register的静态变量，该Register的构造函数中调用创建工厂的Regist把类名和构造函数传入创建工厂的creator_map_里面。这里有一点要注意的是，DynamicCreator创建器的构造函数需要调用register的do_nothing()方法，不然编译器不会构造register（被编译器优化了）。

```
// 动态对象创建器，B必须是T的基类，B用于返回指针引用  
 template<typename B, typename T, typename ...Targs>  
 class DynamicCreator {  
 public:  
     struct Register {  
         Register() {//在Register的构造函数里面把子类的字符串和对应子类的  
             std::cout << "DynamicCreator.Register construct" << std::endl;  
             char* szDemangleName = nullptr;  
             std::string strTypeName;  
             szDemangleName = abi::__cxa_demangle(typeid(T).name(), nullptr, nullptr, nullptr);  
             if (nullptr != szDemangleName) {  
                 strTypeName = szDemangleName;  
                 free(szDemangleName);  
             }  
  
             ClassFactory<B, Targs...>::Instance()->Regist(strTypeName, CreateObject);  
         }  
         inline void do_nothing()const { };  
     };  
     DynamicCreator() {  
         std::cout << "DynamicCreator construct" << std::endl;  
         //必须有do_nothing()，不然编译器不会构造 s_registor  
         s_registor.do_nothing();  
     }  
     virtual ~DynamicCreator(){s_registor.do_nothing();};  
  
     // 动态创建实例的方法，用args调用T构造函数。  
     static B* CreateObject(Targs&&... args) {  
         std::cout << "static T* DynamicCreator::CreateObject(Targs... args)" << std::endl;  
         return new T(std::forward<Targs>(args)...);//用args调用构造T构造函数  
     }  
  
     virtual void Say() {  
         std::cout << "DynamicCreator say" << std::endl;  
     }  
     static Register s_registor;  
 };  
  
 template<typename B, typename T, typename ...Targs>  
 typename DynamicCreator<B, T, Targs...>::Register DynamicCreator<B, T, Targs...>::s_registor;</div></div>
 ```

如何使用：

1. 先定义一个基类，用于这一类子类的父类，比如计算是一类，词典是一类。

```
class A {  
     public:  
     virtual void Say() = 0;  
     virtual ~A(){}  
 };
 ```

2. 定义几个子类，继承其基类以及**DynamicCreator**。当继承DynamicCreator的时候，DynamicCreator的静态register变量就会用入参调用ClassFactory单例的Regist把类名和构造函数插入ClassFactory的creator_map_里面。

```
class B : public A, public DynamicCreator<A,B> {  
     public:  
     B() {std::cout << "Creating B" << std::endl;}  
     virtual void Say() {  
         std::cout << "I am B" << std::endl;  
     }  
 };  
 class C: public A, public DynamicCreator<A,C,int> {  
     public:  
     C(int x) {  
         std::cout << "Creating C with " << x << std::endl;  
         x_ = x;  
      }  
      virtual void Say() {  
          std::cout << "I am C" << std::endl;  
      }  
     private:  
       int x_;  
 };  
 class D : public A, public DynamicCreator<A,D,std::string> {  
     public:  
     D(std::string x) {std::cout << "Creating D with" << x << std::endl;  
     x_ = x;}  
     virtual void Say() {  
         std::cout << "I am D" << std::endl;  
     }  
     private:  
     std::string x_;  
 };
 ```

3. 定义后需要使用ClassFactory的Create方法，那么这里我们用A和构造函数入参来组装调用ClassFactory的Create。

```
class ComputerFactory {  
     public:  
     template<typename ...Args>  
     static A* CreateComputerA(const std::string& str_type_name, Args... args){  
       A* p = ClassFactory<A, Args...>::Instance()->Create(str_type_name, std::forward<Args>(args)...);  
       return p;  
     }  
 };  
  
 }
 ```

4. 有了ComputerFactory，我们就可以直接使用：

```
int main()  
 {  
     std::cout << "****************** here is test ************************************" << std::endl;  
     reflect::A* p1 = reflect::ComputerFactory::CreateComputerA(std::string("reflect::B"));  
     p1->Say();  
     std::cout << "----------------------------------------------------------------------" << std::endl;  
     reflect::A* p2 = reflect::ComputerFactory::CreateComputerA(std::string("reflect::C"), 1002);  
     p2->Say();  
     std::cout << "----------------------------------------------------------------------" << std::endl;  
     std::string haha="hahaha";  
     reflect::A* p3 = reflect::ComputerFactory::CreateComputerA(std::string("reflect::D"),haha);  
     p3->Say();  
     return(0);  
 }
 ```

编译：

`g++ -std=c++11 DynamicCreate.cpp -o DynamicCreate`

输出：

```
DynamicCreator.Register construct  
 static ClassFactory* Instance()  
 ClassFactory construct  
 bool ClassFactory::Regist(const std::string& strTypeName, std::function<B*(Targs... args)> pFunc)  
 creator_map_.size() =1  
 DynamicCreator.Register construct  
 static ClassFactory* Instance()  
 ClassFactory construct  
 bool ClassFactory::Regist(const std::string& strTypeName, std::function<B*(Targs... args)> pFunc)  
 creator_map_.size() =1  
 DynamicCreator.Register construct  
 static ClassFactory* Instance()  
 bool ClassFactory::Regist(const std::string& strTypeName, std::function<B*(Targs... args)> pFunc)  
 creator_map_.size() =2  
 ****************** here is test ************************************  
 static ClassFactory* Instance()  
 B* ClassFactory::Create(const std::string& strTypeName, Targs... args)  
 static T* DynamicCreator::CreateObject(Targs... args)  
 DynamicCreator construct  
 Creating B  
 I am B  
 ----------------------------------------------------------------------  
 static ClassFactory* Instance()  
 B* ClassFactory::Create(const std::string& strTypeName, Targs... args)  
 static T* DynamicCreator::CreateObject(Targs... args)  
 DynamicCreator construct  
 Creating C with 1002  
 I am C  
 ----------------------------------------------------------------------  
 static ClassFactory* Instance()  
 B* ClassFactory::Create(const std::string& strTypeName, Targs... args)  
 static T* DynamicCreator::CreateObject(Targs... args)  
 DynamicCreator construct  
 Creating D with1234  
 I am D
 ```

可以看到在main函数开始执行第一行代码之前，先是DynamicCreator的register变量初始化，把类名和构造函数注入ClassFactory的creator_map_里面。这里入参相同的ClassFactory使用同一个单例，所以C和D使用的是同一个ClassFactory（creator_map_的size为2）。然后在main函数中，获取ClassFactory的单例的Create方法，找到creator_map_里面本类名对应的构造函数，然后调用之。


## 小结

本篇文章总结的使用可变参数模板实现的反射的思路是，首先需要有一个全局的map保存类名和构造函数的映射关系，那么我们需要有ClassFactory类来持有这样一个map，该类依照构造函数的不同类型入参分别构造一个单例，因为不同入参的构造函数不能在同一个map里面。然后我们需要有一个static变量在初始化自身的时候把反射中用到的类名和构造函数插入ClassFactory的map里面，所以我们需要DynamicCreator持有一个static的Register变量，Register在自身的构造函数中调用ClassFactory的Regist把类名和构造函数插入map中。最后我们必须在每个类声明的时候调用到DynamicCreator的Register的构造函数，自然而然就想到每个类需要可以继承DynamicCreator并且Register必须声明为static，这样程序在一开始就把类名和构造函数都插入到map中。


