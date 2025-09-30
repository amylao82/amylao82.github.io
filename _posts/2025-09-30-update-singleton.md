---
layout: post
title:  更新可释放单例模板
tags:   blogging
image: blog.png
---
之前实现的单例模板,里面使用一个int类型的变量ref_count_来记录上层的请求, 然而
{{ more }}
使用变量的方法,需要考虑在多线程环境下的计数加减原子性.

如果二个线程同时进入,一个进入加,一个进入减,很有可能会出问题.


### 新的做法

想到计数,第一时间想到的应该就是智能指针了.智能指针就是通过计数的方式来实现指针是否需要释放的判断.

既然已经有现成的,自己再写一个计数逻辑,就觉得有点多余.还不如直接使用成熟的实现方式.


因此,把计数器修改为智能指针.
然而,如果纯用智能指针,无法达到自动释放的目的,需要结合一个弱指针一起使用.

### 实现代码

代码如下, 使用全局static保存的,是一个弱引用指针.
而真正的智能指针,乃是保存在对象的成员变量`std::shared_ptr<T> instance;`里,这个成员随着singleton_ref_count_guard对象的析构而减少计数.

在第一次请求时,智能指针会返回给请求者,而内部只保留弱指针.
第二次请求时,通过弱指针取得强指针.如果取得,智能指针计数会加1;如果没取得,说明这个弱指针已经失效,已经被释放,需要重新创建构造一个新的对象.

因为在外部拿到的是智能指针,如果释放到计数器减为0,就会进入析构流程,而不用关注singleton_ref_count_guard里的弱指针,这样的机制可以保证进入释放单例的流程.

```
template <typename T>
class singleton_ref_count_guard
{
public:
    template <typename... Args>
    singleton_ref_count_guard(Args&&... args)
    {
        std::shared_ptr<T> current_instance = s_weak_instance.lock();

        if (!current_instance)
        {
            std::lock_guard<std::mutex> lock(s_instance_mutex);
            current_instance = s_weak_instance.lock();
            if (!current_instance)
            {
                current_instance = std::shared_ptr<T>(new T(std::forward<Args>(args)...));
                s_weak_instance = current_instance;
            }
        }
        instance = current_instance;
    }

    T* operator->() const
    {
        if (!instance)
        {
            throw std::runtime_error("Attempt to dereference a null singleton instance.");
        }
        return instance.get();
    }

    T& operator*() const
    {
        if (!instance)
        {
            throw std::runtime_error("Attempt to dereference a null singleton instance.");
        }
        return *instance;
    }

    std::shared_ptr<T> get() const
    {
        return instance;
    }

private:
    static std::weak_ptr<T> s_weak_instance;
    static std::mutex s_instance_mutex;

    //每个类,都有一个智能指针.
    std::shared_ptr<T> instance;
};

template <typename T>
std::weak_ptr<T> singleton_ref_count_guard<T>::s_weak_instance;

template <typename T>
std::mutex singleton_ref_count_guard<T>::s_instance_mutex;

```

### 另一个改进
对于单例类,最好的方法是让类的构造只有一个入口,而不能让用户通过不同的方式,构造出多个实例.

因此,在这个模板类里,增加了一个小改动,也就是增加一个制约: ** 要接受singleton_ref_count_guard管理的类,需要把构造函数设置为protected,并且把singleton_ref_count_guard设置为友元类**

为什么要设置这样的规则?
假设写一个类,把构造函数设置为public.

```
class test_obj
{
    public:
        test_obj();
        friend class singleton_ref_count_guard<test_obj>;
}; 
```

这个类里,因为上层应用有多个途径创建对象,将无法实现单例.

比如这样写:

```
singleton_ref_count_guard<test_obj> obj1;

singleton_ref_count_guard<test_obj> obj2;

test_obj obj3;

test_obj* obj4 = new test_obj();

```

这四行代码,创建了三个对象,只有前二个创建的是单例,而后面的二个,将与前二句创建的对象毫无瓜葛.

如何解决这个问题?
很简单,把test_obj这个类的构造函数设置为保护模式(protected),上面的第三,第四句将无法编译通过,只留下前二句可以成功编译.从而从编码期间即把问题消解于无形.

友元类`friend class singleton_ref_count_guard<test_obj>` 那一句,是为了让singleton_ref_count_guard能够访问类的保护模式下的构造函数而增加的. **一定要加上**




### 正确的单例类的实现方式

这里给出一个例子,结合singleton_ref_count_guard的正确单例类的实现.


```
class test_obj
{
    public:
        ~test_obj();
        void func1();
        friend class singleton_ref_count_guard<test_obj>;
    protected:
        test_obj();
}; 
```

重点:
1. 构造函数一定放到保护模式下
2. 一定要声明singleton_ref_count_guard为友元类

这样的规则,在后续的调查过程中,也有帮助,当看到有声明singleton_ref_count_guard作为友元时,即可知道这个类是单例.

接下来,在整个应用程序的生命周期内,如果要调用单类,就直接定义一个singleton_ref_count_guard对象即可.

```
singleton_ref_count_guard<test_obj> obj1;
obj1->func1();

// another thread 
singleton_ref_count_guard<test_obj> obj2;
obj2->func1();

```

### 效果保证

这个单例模板,在项目中已经正常使用,保证任务正确完成.


### 局限

这个模板的局限,只能适用于无参数的类,如果一个类有参数,则无法进行接管.

有参数类的问题,一时还无解.



