---
layout: post
title:  实现一个可释放的单例模板
tags:   blogging
image: blog.png
---
通常的单例,其生命周期都与应用程序生命周期相同.一旦创建,就没法释放,直到程序生命结束,才会进入释放流程.
{{ more }}


### 通常做法
通常情况下,我们实现单例的方式,大概有二种.
这二种方法都可以实现单例模式.

#### 指针法
使用一个指针,在第一次调用时,检查是否为空,如果为空则创建一个对象.

后续的调用,因为指针已经不为空,所以不会再次进入创建流程,而是直接返回指向之前创建对象的指针.

```
class singleton {
public:
    static singleton* get_instance() {
        if (instance_ == nullptr) { // 第一次检查
            std::lock_guard<std::mutex> lock(mutex_);
            if (instance_ == nullptr) { // 第二次检查
                instance_ = new singleton();
            }
        }
        return instance_;
    }

private:
    singleton() {} // 私有构造函数

    static singleton* instance_;
    static std::mutex mutex_;
};

singleton* singleton::instance_ = nullptr;
std::mutex singleton::mutex_;
```

#### 静态变量法

另外一种方法,使用静态变量方法.

第一次调用时,初始化静态变量并返回,后续的调用返回的都是这个静态变量的引用.

```
class singleton {
public:
    static singleton& get_instance() {
        static singleton instance; // 静态局部变量
        return instance;
    }
    
    // 删除拷贝构造函数和赋值运算符，确保单例唯一
    singleton(const singleton&) = delete;
    singleton& operator=(const singleton&) = delete;
    
private:
    singleton() {} // 私有构造函数
    // 私有析构函数 (可选)
    // ~singleton() {}
};
```

### 新的需求
通用做法,通常不能控制单例的生命周期,单例创建后,将会一直伴随到整个程序的生命结束.

在实际应用中,有时候需要动态控制一个单例的提前结束.此时,通用的单例将不能满足需求.

比如在嵌入式开发中.有一些临界资源,做成单例模式,而这些资源在运行的过程中,需要消耗资源.为了节省资源,需要提前结束这些单例.

举一个例子,最近做的一个项目:通过4个摄像头采集图像数据,并使用不同的编码器编码为H264码流,提供给RTSP,RTMP,UVC使用.

如果使用通常的单例模式,这些编码器在创建后,将一直伴随整个应用,占用系统资源.

1. 比如, 刚开始时,RTSP,RTMP都请求1号摄像头.此时使用单例创建一个编码器是正确的,二个协议都拿到同一份编码流.
2. 接下来,RTSP,RTMP转而请求2号摄像头,此时系统内就要新创建一个编码器,用于2号摄像头的图像编码. 但是因为单例没有释放机制,1号摄像头的编码器没法释放.
3. 系统处理不同的请求,最后,所有的编码器都被创建,而无法释放.即使使用不再有客户端请求图像,系统内的编码器资源仍然被占用.


### 新的实现
针对新的需求,需要实现另外一套机制,让单例可以进入到释放流程里.

查了一下C++的标准库里,一时找不到合适的类来实现这个功能.因此,自己花点时间写了一个.

主要就是接管单例的创建,并增加一个计数器,在计数器变为0时,进入单例的释放流程.

直接放代码

```
#pragma once

#include <iostream>
#include <mutex>
#include <atomic>
#include <memory>

// T: 必须是一个实现了 get_instance() 和 destroy_instance() 静态方法的类
template <typename T>
class singleton_ref_count_guard {
public:
    singleton_ref_count_guard() {
        std::lock_guard<std::mutex> lock(counter_mtx_);
        if (ref_count_ == 0) {
            // 通过类型 T 的静态方法来获取和创建实例
            T::get_instance();
        }
        ref_count_++;
    }

    ~singleton_ref_count_guard() {
        std::lock_guard<std::mutex> lock(counter_mtx_);
        ref_count_--;
        if (ref_count_ == 0) {
            // 通过类型 T 的静态方法来销毁实例
            T::destroy_instance();
        }
    }

    // 重载 -> 操作符
    T* operator->() const {
        return &T::get_instance();
    }

    // 重载 * 操作符 (解引用)
    T& operator*() const {
        return T::get_instance();
    }

private:
    static std::atomic<int> ref_count_;
    static std::mutex counter_mtx_;
};

// 静态成员变量的定义（每个模板实例都有一套独立的静态变量）
template <typename T>
std::atomic<int> singleton_ref_count_guard<T>::ref_count_ = 0;

template <typename T>
std::mutex singleton_ref_count_guard<T>::counter_mtx_;

```

在这个类里,单例的get_instance不再由应用层调用.转而在singleton_ref_count_guard里调用,每调用一次,即增加一次计数器. 而在类释放时,减少一次计数,计数为0时,进入单例的释放流程.

使用的方法很简单. 二个服务,只要拿一个编码器即可以做自己的事.不必关注其他线程是否存在.

而使用完成后,即可释放singleton_ref_count_guard<venc_for_cam1>, 而至于venc是否真正释放,singleton_ref_count_guard会自己处理.

```
void thread_rtsp()
{
    singleton_ref_count_guard<venc_for_cam1> venc;
    venc->do_something();
}

void thread_rtmp()
{
    singleton_ref_count_guard<venc_for_cam1> venc;
    venc->do_something();
}

```

### 注意事项

事物都有二面性,这种方法虽然可以解决单例的释放,但是在使用的过程中,需要注意二个事项.


#### 单例的释放函数

因为singleton_ref_count_guard需要调用单例的释放函数.所以,每个单例都要实现一个释放函数,并且函数的名字需要限定.本例中的名字是`destroy_instance`.

如果单例不实现这个函数, 将不能正常工作.

#### 单例调用的统一

单例的使用,在整个程序中都必须统一使用singleton_ref_count_guard来接管.否则会引起计数的混乱.

比如如下的程序,会带来单例释放的不确定性.

```
void thread_rtsp()
{
    singleton_ref_count_guard<venc_for_cam1> venc;
    venc->do_something();
}

void thread_rtmp()
{
    //程序自己管理单例的生命周期.
    venc_for_cam1 venc = venc_for_cam1::get_instance();
    venc->do_something();

    venc_for_cam1::destroy_instance();
}
```

----

虽然该类的使用存在限制,但是在资源有限的系统里,引入这个类还是有一定的实际意义的.

类似于shared_ptr, 使用计数器,实现对同一个对象的共用.但是shared_ptr有一个好处,在混用时,指向的不是同一个对象.因此不会造成混乱.

"不以规矩,无以成方圆",凡事都需要在一定的规则下使用.

只要在规则下使用这个类,可以减少很多工作量.

但是如果不明这个规则限制,那将会造成一个混乱的局面.

### 类比模型
singleton_ref_count_guard这个类在完成后,我对它进行了一番思考.这个类的机制在现实中是否有其意义?如果没有意义,这个类将不具有存在的必要.

最后想明白了,这可能是一个租借模型.

比如,你家里有一个打铁熔炉,这个熔炉要保持高温,才能炼化金属.

你要提供对外服务,周围几十个村落的人,要打造铁器时,都需要找你借熔炉使用.

此时,可以设计一个模式,派出几个代理人,在有客户需要打铁器时,就派一个代理人对接.

你每天在家计点代理人人数,如果今天所有的代理人都在家,那这个熔炉就停火,减少燃料的消耗.

如果代理人人数不足,即使缺一个,也是要保存熔炉工作在高温状态.

这样的模型,可能在设计模式里有吧,可能我还不知道,要再找设计模式的书再重温一遍看看.


### 模板编程

对于模板编程,最早接触的应该是VC6的STL模板.因为当时的编程能力不强,对模板总爱不起来.只要出错,报出来的错误信息极长极长,小脑瓜里完全无法分析错误在哪.因此在当时,能不用模板就不用模板.

而随着年岁的增长,现在也在尝试着去了解模板编程的相关知识.

在Scott Meyers 《中文版Effective STL:50条有效使用STL的经验》的"第49条：学会分析与STL相关的编译器诊断信息"文章里，有教一些STL错误分析的方法,看完之后有种醍醐灌顶的感觉.(STL大多使用模板实现,因此,针对STL的调试分析方法,可以适用于模板).

现在,在开发的过程中,也会试着使用一些模板开发的知识,但是还是不敢太放肆.后续再慢慢加强.



