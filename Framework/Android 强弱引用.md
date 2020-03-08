# Android 强弱引用

> 软件：Source Insight 4.0、starUml



代码目录：

```java
system/core/include/utils/RefBase.h
system/core/include/utils/StrongPointer.h
system/core/libutils/RefBase.cpp
```

<a name="6ddda5c2"></a>
# 一、简介

从media服务的main函数入口中，可以发现

```cpp
sp<ProcessState> proc(ProcessState::self());
```

sp其实就是强指针或强引用(strong pointer)，与 sp 对应的是 wp，也就是弱指针或弱引用(weak pointer)，Android Native 层(C++)中大量使用了sp/wp，了解sp/wp的源码及使用对理解Android Native层是很有帮助的。

智能指针最大的作用就是自动销毁对象。有这么一个场景，对象 A 和 对象 B 互相引用，系统无法回收这两个对象，任意回收其中一个对象，都会发现被另外一个对象引用，导致“死锁”无法回收，针对这种情况，可以使用强弱指针，对A对象使用强指针计数，对B对象使用弱指针计数，只要强指针计数为0，不管弱引用计数是否为0，都可以回收这个对象。一个对象只有弱指针引用时，必须升级为强指针，才能使用这个对象。

<a name="f8d30eeb"></a>
# 二、sp 源码分析

sp 强指针就是一个模板类，有点类似于Java的泛型，先看下 sp 的头文件 `StrongPointer.h`：

```cpp
template<typename T>
class sp {
public:
    inline sp() : m_ptr(0) { }

    sp(T* other);  // 方式1
    sp(const sp<T>& other); // 方式2
    sp(sp<T>&& other); // 方式3
    template<typename U> sp(U* other);  // NOLINT(implicit)
    template<typename U> sp(const sp<U>& other);  // NOLINT(implicit)
    template<typename U> sp(sp<U>&& other);  // NOLINT(implicit)

    ~sp();

    // Assignment

    sp& operator = (T* other); // 方式4
    sp& operator = (const sp<T>& other); // 方式5
    sp& operator = (sp<T>&& other); // 方式6

    template<typename U> sp& operator = (const sp<U>& other);
    template<typename U> sp& operator = (sp<U>&& other);
    template<typename U> sp& operator = (U* other);
    //! Special optimization for use by ProcessState (and nobody else).
    void force_set(T* other);
    // Reset
    void clear();
    // Accessors
    inline T&       operator* () const     { return *m_ptr; }
    inline T*       operator-> () const    { return m_ptr;  }
    inline T*       get() const            { return m_ptr; }
    inline explicit operator bool () const { return m_ptr != nullptr; }
    // Operators
    COMPARE(==)
    COMPARE(!=)
    COMPARE(>)
    COMPARE(<)
    COMPARE(<=)
    COMPARE(>=)
private:    
    template<typename Y> friend class sp;
    template<typename Y> friend class wp;
    void set_pointer(T* ptr);
    T* m_ptr;
};
```

<a name="4c361381"></a>
## 2.1 sp 构造函数

【StrongPointer.h】

sp 初始化的方式有6种：

```c
 sp(T* other);  // 方式1
 sp(const sp<T>& other); // 方式2
 sp(sp<T>&& other); // 方式3
 sp& operator = (T* other); // 方式4
 sp& operator = (const sp<T>& other); // 方式5
 sp& operator = (sp<T>&& other); // 方式6
```

具体方法如下：

方法1

```cpp
template<typename T>
sp<T>::sp(T* other)
        : m_ptr(other) {
    if (other)
        other->incStrong(this);
}
```

方法2

```cpp
template<typename T>
sp<T>::sp(const sp<T>& other)
        : m_ptr(other.m_ptr) {
    if (m_ptr)
        m_ptr->incStrong(this);
}
```

方法3

```cpp
template<typename T>
sp<T>::sp(sp<T>&& other)
        : m_ptr(other.m_ptr) {
    other.m_ptr = nullptr;
}
```

方法4

```cpp
template<typename T>
sp<T>& sp<T>::operator =(T* other) {
    T* oldPtr(*const_cast<T* volatile*>(&m_ptr));
    if (other) other->incStrong(this);
    if (oldPtr) oldPtr->decStrong(this);
    if (oldPtr != *const_cast<T* volatile*>(&m_ptr)) sp_report_race();
    m_ptr = other;
    return *this;
}
```

方法5

```cpp
template<typename T>
sp<T>& sp<T>::operator =(const sp<T>& other) {
    // Force m_ptr to be read twice, to heuristically check for data races.
    T* oldPtr(*const_cast<T* volatile*>(&m_ptr));
    T* otherPtr(other.m_ptr);
    if (otherPtr) otherPtr->incStrong(this);
    if (oldPtr) oldPtr->decStrong(this);
    if (oldPtr != *const_cast<T* volatile*>(&m_ptr)) sp_report_race();
    m_ptr = otherPtr;
    return *this;
}
```

方法6

```cpp
template<typename T>
sp<T>& sp<T>::operator =(sp<T>&& other) {
    T* oldPtr(*const_cast<T* volatile*>(&m_ptr));
    if (oldPtr) oldPtr->decStrong(this);
    if (oldPtr != *const_cast<T* volatile*>(&m_ptr)) sp_report_race();
    m_ptr = other.m_ptr;
    other.m_ptr = nullptr;
    return *this;
}
```

简单总结下上面六种方法：

- 方法1、2、3都是采用括号的形式直接定义 sp 对象，再调用目标对象的 `incStrong()` 方法。
- 方法4、5、6都是采用等号赋值的形式定义 sp 对象，先调用目标对象的 `incStrong()` 方法，再调用原来对象的 `decStrong()` 方法。

<a name="70ab6ea7"></a>
## 2.2 sp 实例分析

Android Native层有很多使用了 sp 对象，比如开头提到的 `ProcessState` 对象和后面会分析开机动画的 `BootAnimation` 对象。

```cpp
sp<ProcessState> proc(ProcessState::self());    // 方法2
sp<BootAnimation> boot = new BootAnimation(new AudioAnimationCallbacks());    // 方法4
```

- ProcessState::self() 返回的是`sp<ProcessState> gProcess` 对象，因此它使用的是方法2
- new BootAnimation() 明显返回的是 BootAnimation 指针，因此它使用的是方法4

接下来继续分析 ProcessState 的创建过程，`ProcessState.h`中可以看出 ProcessState 继承的是类 RefBase

```cpp
class ProcessState : public virtual RefBase
```

new ProcessState() 会先初始化 RefBase 类。

<a name="e0ab8af0"></a>
## 2.3 RefBase 类

<a name="12d5aa73"></a>
### 2.3.1 RefBase 构造函数

【RefBase.cpp】

```cpp
RefBase::RefBase()
    : mRefs(new weakref_impl(this))
{
}
```

RefBase 初始化的时候，会创建一个成员变量 `mRefs`，即创建了一个weakref_impl对象，其中 this 指向 ProcessState 。

<a name="0d059e5e"></a>
### 2.3.2 weakref_impl 类

【RefBase.cpp :: weakref_impl】

```cpp
class RefBase::weakref_impl : public RefBase::weakref_type
{
public:
    std::atomic<int32_t>    mStrong;
    std::atomic<int32_t>    mWeak;
    RefBase* const          mBase;
    std::atomic<int32_t>    mFlags;

//  DEBUG_REFS=0 默认是release版本，非DEBUG版本
#if !DEBUG_REFS   
    explicit weakref_impl(RefBase* base)
        : mStrong(INITIAL_STRONG_VALUE)   // 强引用计数，默认是2>>28 = 0x1000 0000
        , mWeak(0)    // 弱引用计数，默认是 0
        , mBase(base)    // 这里是ProcessState 指针
        , mFlags(0)   // 默认是OBJECT_LIFETIME_STRONG=0x0000
    {
    }
    // release版本以下方法都为空，直接忽略
    void addStrongRef(const void* /*id*/) { }
    void removeStrongRef(const void* /*id*/) { }
    void renameStrongRefId(const void* /*old_id*/, const void* /*new_id*/) { }
    void addWeakRef(const void* /*id*/) { }
    void removeWeakRef(const void* /*id*/) { }
    void renameWeakRefId(const void* /*old_id*/, const void* /*new_id*/) { }
    void printRefs() const { }
    void trackMe(bool, bool) { }
#else
```

weakref_impl 继承的是 weakref_type 类，可以调用 weakref_type 类中的方法。mBase 是前面 ProcessState 的指针

<a name="f37b5963"></a>
### 2.3.3 weakref_type 类

【RefBase.h :: weakref_type】

```cpp
class weakref_type
    {
    public:
        RefBase*            refBase() const;
        void                incWeak(const void* id);
        void                decWeak(const void* id);
        bool                attemptIncStrong(const void* id);
        bool                attemptIncWeak(const void* id);
        int32_t             getWeakCount() const;
        void                printRefs() const;
        void                trackMe(bool enable, bool retain);
    };
```

主要调用到的是 incWeak 和 decWeak 方法，后面再详细介绍这两个方法。

sp 初始化过程中类的继承关系理清之后，再来看前面提到的 sp 初始化的几种方式，不管是哪种方式，最后都会调用到 `incStrong`。

<a name="231cc26a"></a>
### 2.3.4 incStrong 方法

【RefBase.cpp :: incStrong】

```cpp
void RefBase::incStrong(const void* id) const
{
    weakref_impl* const refs = mRefs;
    refs->incWeak(id);    // 调用的是weakref_type的incWeak方法
    refs->addStrongRef(id);  // release版本忽略
    // 强引用计数+1，返回旧值c
    const int32_t c = refs->mStrong.fetch_add(1, std::memory_order_relaxed);
   // 如果不是第一次引用则直接返回
    if (c != INITIAL_STRONG_VALUE)  { // 强引用初始值 0x1000 0000
        return;
    }
    // mStrong = 1
    int32_t old __unused = refs->mStrong.fetch_sub(INITIAL_STRONG_VALUE, std::memory_order_relaxed);
    // 第一次引用的时候调用 onFirstRef
    refs->mBase->onFirstRef();
}
```

- `refs->incWeak(id)` 调用的是前面提到的 weakref_type 的 incWeak 方法
```cpp
void RefBase::weakref_type::incWeak(const void* id)
{
    weakref_impl* const impl = static_cast<weakref_impl*>(this);
    impl->addWeakRef(id); // release 版本忽略
    // 弱引用+1
    const int32_t c __unused = impl->mWeak.fetch_add(1,
            std::memory_order_relaxed);
}
```

- <br />mWeak初始值是 0 ，调用完后弱引用 mWeak 计数 +1，`mWeak = 1`
- `c = refs->mStrong.fetch_add(1, std::memory_order_relaxed)` 表示 mStrong + 1，fetch_add 的返回值是旧值，也就是原来的 mStrong，mStrong 初始值 = 0x1000 0000，调用该行后，mStrong = 0x1000 0000 + 1 = 0x1000 0001
<br />c = 0x1000 0000。下面的判断如果 c != 0x1000 0000，说明该对象不是第一次被强引用，则不会调用下面的onFirstRef()初始化方法，而直接返回
- `refs->mStrong.fetch_sub(INITIAL_STRONG_VALUE, std::memory_order_relaxed)`
<br />上一步得出mStrong = 0x1000 0001，fetch_sub() 后 mStrong = 0x1000 0001 - 0x1000 0000 = 1，所以
<br />`mStrong = 1`
- `refs->mBase->onFirstRef()` 如果是第一次引用，派生类可以重载该方法，做一些初始化的设置

**incStrong() 方法的主要作用**：

- weakref_impl 的强引用计数mStrong + 1，弱引用计数 mWeak + 1。
- 第一次调用incStrong，会调用目标对象如 ProcessState 的 onFirstRef() 方法。

<a name="33e0b076"></a>
## 2.4 sp析构函数

【StrongPointer.h :: ~sp】

```cpp
template<typename T>
sp<T>::~sp() {
    if (m_ptr)
        m_ptr->decStrong(this);
}
```

<a name="f0512102"></a>
### 2.4.1 decStrong 方法

【RefBase.cpp :: decStrong】

```cpp
void RefBase::decStrong(const void* id) const
{
    weakref_impl* const refs = mRefs;
    refs->removeStrongRef(id);  // release版本直接忽略
    // 强引用计数 mStrong - 1
    // 调用前 mStrong = 1，调用后 mStrong = 0, c=1
    const int32_t c = refs->mStrong.fetch_sub(1, std::memory_order_release);
    if (c == 1) {
        std::atomic_thread_fence(std::memory_order_acquire);
        refs->mBase->onLastStrongRef(id);
        int32_t flags = refs->mFlags.load(std::memory_order_relaxed);
        if ((flags&OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_STRONG) {
            delete this;   // flags = OBJECT_LIFETIME_STRONG时，回收掉目标对象
        }
    }
    refs->decWeak(id);
}
```

- `c = refs->mStrong.fetch_sub(1, std::memory_order_release)`
<br />前面知道mStrong = 1，该行执行后，**强引用计数 -1**， `mStrong = 0`，`c = 1`。
- c == 1表明强引用计数为0，对象可能要被delete了，回调目标对象ProcessState的onLastStrongRef()方法

<a name="f115ce00"></a>
### 2.4.2 mFlags 取值

```cpp
【RefBase.h】
    enum {
        OBJECT_LIFETIME_STRONG  = 0x0000,
        OBJECT_LIFETIME_WEAK    = 0x0001,
        OBJECT_LIFETIME_MASK    = 0x0001
    };
 【RefBase.cpp】
  void RefBase::extendObjectLifetime(int32_t mode)
{
    mRefs->mFlags.fetch_or(mode, std::memory_order_relaxed);
}
```

- 先讨论下 mFlags 的取值，mFlags 是由 extendObjectLifetime() 确定的，取值范围在枚举中，默认值是 0 即OBJECT_LIFETIME_STRONG
- 上面的decStrong()中，当 flag 是默认值 OBJECT_LIFETIME_STRONG时，会调用`delete this`回收掉目标对象，此时 RefBase 析构函数也会被调用

<a name="9a09e585"></a>
### 2.4.3 RefBase 析构函数

```cpp
RefBase::~RefBase()
{
    int32_t flags = mRefs->mFlags.load(std::memory_order_relaxed);
    if ((flags & OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_WEAK) { 
        if (mRefs->mWeak.load(std::memory_order_relaxed) == 0) { 
            // 当 flags = OBJECT_LIFETIME_WEAK，弱引用控制生命周期，且弱引用计数 mWeak=0，则删除 weakref_impl对象
            delete mRefs;
        }
    } else if (mRefs->mStrong.load(std::memory_order_relaxed)
            == INITIAL_STRONG_VALUE) { // mStrong = 0x1000 0000 会删除mRefs
        delete mRefs;
    }
    const_cast<weakref_impl*&>(mRefs) = NULL;
}
```

`delete mRefs` 的条件：

- 当生命周期被弱引用掌握时(flags = OBJECT_LIFETIME_WEAK = 0x0001) 且 弱引用计数 = 0，则删除weakref_impl对象。
- 当生命周期被强引用掌握时，且没被强引用引用过，也会删除weakref_impl对象。

此时 flags = OBJECT_LIFETIME_STRONG，且被强引用引用过，因此不会删除 mRefs。

<a name="4556b5d8"></a>
### 2.4.4 decWeak 方法

继续看 decStrong() 中的  refs->decWeak(id);

【RefBase.cpp :: weakref_type :: decWeak】

```cpp
void RefBase::weakref_type::decWeak(const void* id)
{
    weakref_impl* const impl = static_cast<weakref_impl*>(this);
    impl->removeWeakRef(id);
    // 弱引用计数 mWeak - 1。
    // 调用前 mWeak = 1, mStrong=0，调用后 c=1，mWeak=0, mStrong=0,
    const int32_t c = impl->mWeak.fetch_sub(1, std::memory_order_release);
    if (c != 1) return;
    atomic_thread_fence(std::memory_order_acquire);

    // mWeak = 0, mStrong=0, mFlags=0
    int32_t flags = impl->mFlags.load(std::memory_order_relaxed);
    if ((flags&OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_STRONG) {
        if (impl->mStrong.load(std::memory_order_relaxed)
                == INITIAL_STRONG_VALUE) {
            // 当 flags = OBJECT_LIFETIME_STRONG(强引用控制生命周期)，且没有被强指针引用过，则什么都不做。
        } else {
           // 当 flags = OBJECT_LIFETIME_STRONG(强引用控制生命周期)，且被强指针引用过，则删除 weakref_impl对象
            delete impl;
        }
    } else {
        // 当 flags = OBJECT_LIFETIME_WEAK，弱引用控制生命周期，则删除实际对象。
        impl->mBase->onLastWeakRef(id);
        delete impl->mBase;
    }
}
```

decWeak() 的主要作用：

- 弱引用计数 mWeak - 1，此时 mStrong=0，mWeak=0
- 当 flags = OBJECT_LIFETIME_STRONG，强引用控制生命周期，且没有被强指针引用过，则什么都不做。
<br />当 flags = OBJECT_LIFETIME_STRONG，强引用控制生命周期，且被强指针引用过，则delete weakref_impl对象
<br />当 flags = OBJECT_LIFETIME_WEAK，弱引用控制生命周期，则delete实际对象ProcessState。前面分析过，删除实际对象会调用 RefBase 的析构函数，此时弱引用计数=0，weakref_impl也会被delete。

<a name="5e2a0bfe"></a>
## 2.5 sp 小结

sp 构造完成后：

- 创建了一个 weakref_impl的指针 mRefs，管理强弱引用计数。
- incStrong() 中强引用计数 +1 ，incWeak() 中弱引用计数 +1

sp 析构完成后：

- decStrong() 中强引用计数 -1，decWeak() 中弱引用计数 -1
- flags 控制生命周期

<a name="619c2a5a"></a>
# 三、wp 源码分析

【RefBase.h】

```cpp
template <typename T>
class wp
{
public:
    typedef typename RefBase::weakref_type weakref_type;
    inline wp() : m_ptr(0) { }
    
    wp(T* other);  // NOLINT(implicit)
    wp(const wp<T>& other);
    explicit wp(const sp<T>& other);
    template<typename U> wp(U* other);  // NOLINT(implicit)
    template<typename U> wp(const sp<U>& other);  // NOLINT(implicit)
    template<typename U> wp(const wp<U>& other);  // NOLINT(implicit)

    ~wp();
    wp& operator = (T* other);
    wp& operator = (const wp<T>& other);
    wp& operator = (const sp<T>& other);
    template<typename U> wp& operator = (U* other);
    template<typename U> wp& operator = (const wp<U>& other);
    template<typename U> wp& operator = (const sp<U>& other);
    void set_object_and_refs(T* other, weakref_type* refs);
    sp<T> promote() const;

    void clear();
    inline  weakref_type* get_refs() const { return m_refs; }

    inline  T* unsafe_get() const { return m_ptr; }

    COMPARE_WEAK(==)
    COMPARE_WEAK(!=)
    COMPARE_WEAK(>)
    COMPARE_WEAK(<)
    COMPARE_WEAK(<=)
    COMPARE_WEAK(>=)
    inline bool operator == (const wp<T>& o) const {
        return (m_ptr == o.m_ptr) && (m_refs == o.m_refs);
    }
    template<typename U>
    inline bool operator == (const wp<U>& o) const {
        return m_ptr == o.m_ptr;
    }
    inline bool operator > (const wp<T>& o) const {
        return (m_ptr == o.m_ptr) ? (m_refs > o.m_refs) : (m_ptr > o.m_ptr);
    }
    template<typename U>
    inline bool operator > (const wp<U>& o) const {
        return (m_ptr == o.m_ptr) ? (m_refs > o.m_refs) : (m_ptr > o.m_ptr);
    }

    inline bool operator < (const wp<T>& o) const {
        return (m_ptr == o.m_ptr) ? (m_refs < o.m_refs) : (m_ptr < o.m_ptr);
    }
    template<typename U>
    inline bool operator < (const wp<U>& o) const {
        return (m_ptr == o.m_ptr) ? (m_refs < o.m_refs) : (m_ptr < o.m_ptr);
    }
                         inline bool operator != (const wp<T>& o) const { return m_refs != o.m_refs; }
    template<typename U> inline bool operator != (const wp<U>& o) const { return !operator == (o); }
                         inline bool operator <= (const wp<T>& o) const { return !operator > (o); }
    template<typename U> inline bool operator <= (const wp<U>& o) const { return !operator > (o); }
                         inline bool operator >= (const wp<T>& o) const { return !operator < (o); }
    template<typename U> inline bool operator >= (const wp<U>& o) const { return !operator < (o); }

private:
    template<typename Y> friend class sp;
    template<typename Y> friend class wp;

    T*              m_ptr;
    weakref_type*   m_refs;
};
```

wp 跟 sp 类差不多，wp 多定义了一个 weakref_type类型的指针 m_refs，接下来看最简单的构造函数。

<a name="f5548582"></a>
## 3.1 wp 构造函数

```cpp
template<typename T>
wp<T>::wp(T* other)
    : m_ptr(other)
{
    if (other) m_refs = other->createWeak(this);
}
```

这里调用的是RefBase类的createWeak方法，RefBase类的构造前面介绍过，直接来看 createWeak 方法。

<a name="2fd69112"></a>
### 3.2.1 createWeak 方法

【RefBase.cpp :: createWeak】

```cpp
RefBase::weakref_type* RefBase::createWeak(const void* id) const
{
    mRefs->incWeak(id);
    return mRefs;
}
```

createWeak 方法主要作用是 调用了 incWeak 方法。

<a name="f1a99287"></a>
### 3.2.2 incWeak 方法

【RefBase.cpp :: weakref_type :: incWeak】

```cpp
void RefBase::weakref_type::incWeak(const void* id)
{
    weakref_impl* const impl = static_cast<weakref_impl*>(this);
    impl->addWeakRef(id);   // release 版本忽略
    // 弱引用计数 mWeak + 1
    const int32_t c __unused = impl->mWeak.fetch_add(1,
            std::memory_order_relaxed);
}
```

incWeak 方法主要作用是 弱引用计数 mWeak + 1，默认mWeak = 0，调用后 mWeak = 1

<a name="4de94d3c"></a>
## 3.2 wp 析构函数

【RefBase.h】

```cpp
template<typename T>
wp<T>::~wp()
{
    if (m_ptr) m_refs->decWeak(this);
}
```

前面介绍过 decWeak 方法，最终 弱引用计数 mWeak - 1，是否要删除对象根据 flag 来决定。

<a name="a2b97596"></a>
## 3.3 wp 升级为 sp

弱指针 wp 最大的特点就是不能直接操作目标对象，这是因为 wp 没有重载 . 和->操作符号，而 sp 重载了这两个操作符号。wp 要操作目标对象，必须要通过 `promote()` 方法升级为强指针。

<a name="881519a1"></a>
### 3.3.1 promote 方法

【RefBase.h :: promote】

```cpp
template<typename T>
sp<T> wp<T>::promote() const
{
    sp<T> result;
    if (m_ptr && m_refs->attemptIncStrong(&result)) {
        result.set_pointer(m_ptr);
    }
    return result;
}
```

升级的关键就是 attemptIncStrong 方法的返回值了。

<a name="214f1387"></a>
### 3.3.2 attemptIncStrong 方法

【RefBase.cpp :: weakref_type :: attemptIncStrong 】

```cpp
bool RefBase::weakref_type::attemptIncStrong(const void* id)
{
    // 弱引用计数 + 1
    incWeak(id);
    weakref_impl* const impl = static_cast<weakref_impl*>(this);
    int32_t curCount = impl->mStrong.load(std::memory_order_relaxed);
    // 尝试让强引用计数 +1
    // 强引用计数 > 0 且 不等于 INITIAL_STRONG_VALUE，强引用计数+1，升级强指针成功
    while (curCount > 0 && curCount != INITIAL_STRONG_VALUE) {
        if (impl->mStrong.compare_exchange_weak(curCount, curCount+1,
                std::memory_order_relaxed)) {
            break;
        }
    }
    // 强引用计数 <= 0或者等于INITIAL_STRONG_VALUE
    if (curCount <= 0 || curCount == INITIAL_STRONG_VALUE) {
        int32_t flags = impl->mFlags.load(std::memory_order_relaxed);
        // 强引用控制生命周期
        if ((flags&OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_STRONG) {
            if (curCount <= 0) { 
                // 强引用计数<=0，即对象被释放掉了，已经不存在，弱引用计数 -1，升级失败
                decWeak(id);
                return false;
            }
            // 强引用计数=INITIAL_STRONG_VALUE，未被强指针引用过
            while (curCount > 0) {
                // 强引用计数 +1
                if (impl->mStrong.compare_exchange_weak(curCount, curCount+1,
                        std::memory_order_relaxed)) {
                    break;
                }
            }
	// 再次处理升级失败的情况
            if (curCount <= 0) {
                decWeak(id);
                return false;
            }
        } else { // 弱引用控制生命周期
            // onIncStrongAttempted=(flags&FIRST_INC_STRONG) ? true : false;
            // 下面flags赋值是FIRST_INC_STRONG，所以返回值是true，!true即false，条件不成立
            if (!impl->mBase->onIncStrongAttempted(FIRST_INC_STRONG, id)) {
                decWeak(id);
                return false;
            }
            // 强引用计数 +1
            curCount = impl->mStrong.fetch_add(1, std::memory_order_relaxed);
            if (curCount != 0 && curCount != INITIAL_STRONG_VALUE) {
                impl->mBase->onLastStrongRef(id);
            }
        }
    }
    impl->addStrongRef(id);  // release 版本忽略
    if (curCount == INITIAL_STRONG_VALUE) {
        impl->mStrong.fetch_sub(INITIAL_STRONG_VALUE,
                std::memory_order_relaxed);
    }
    return true;
}
```

- incWeak() 弱引用计数 +1
- 尝试让强引用计数+1，分两种情况：一种是目标对象正在被强指针引用，强引用计数 > 0 且不等于INITIAL_STRONG_VALUE；另外一种是目标对象没有被任何强指针引用，强引用计数 <= 0 或者等于INITIAL_STRONG_VALUE。
- 第一种情况很简单，说明目标对象肯定存在，弱指针可以成功升级为强指针，弱引用计数 +1，强引用计数 +1
- 第二种情况比较复杂，目标对象可能存在，可能不存在。强引用计数 <= 0或者等于INITIAL_STRONG_VALUE，
  - 当强引用控制生命周期，如果强引用计数 <= 0，说明目标对象被释放，已经不存在了，弱引用计数 -1，升级失败；如果强引用计数等于INITIAL_STRONG_VALUE，未被强指针引用过，则强引用计数+1，升级成功
  - 当弱引用控制生命周期，先判断能否升级强指针，如果可以强引用计数+1，升级成功；如果不能弱引用计数-1，升级失败。

<a name="4b4b0090"></a>
## 3.4 wp 小结

- wp 构造后，incWeak() 中弱引用计数 +1
- wp 析构后，decWeak() 中弱引用计数 -1

<a name="2d325051"></a>
# 四、总结

<a name="e64da002"></a>
## 4.1 类图

![](https://cdn.nlark.com/yuque/0/2020/jpeg/389485/1583647054807-5274f446-6255-4ec5-b6f6-0be669d7a236.jpeg#align=left&display=inline&height=942&originHeight=942&originWidth=704&size=0&status=done&style=none&width=704)

- sp/wp 是模板类
- RefBase 类是 sp/wp 模板类中对象的父类
- weakref_type 是 weakref_impl的父类，weakref_impl指针mRefs，管理引用计数

<a name="727b8a61"></a>
## 4.2 sp/wp
| 引用类型 | 强引用计数 | 弱引用计数 |
| :---: | :---: | :---: |
| sp 构造 | +1 | +1 |
| wp 构造 |  | +1 |
| sp 析构 | -1 | -1 |
| wp 析构 |  | -1 |


- flags 默认值是 `OBJECT_LIFETIME_STRONG`：
  - sp 初始化时，不仅会构造一个`实际对象`(如ProcessState)，还会创建一个 `weakref_impl` 对象，管理强弱引用计数。同时强引用计数 +1，弱引用计数 +1。
  - 强引用计数 = 0时，decStrong() 中 delete `实际对象`。
  - 弱引用计数 = 0时，decWeak() 中 delete `weakref_impl` 对象。
- 第一次调用 incStrong() 方法，会调用实际对象的 onFirstRef() 做一些初始化设置。 最后一次调用decStrong()，则会调用实际对象的 onLastStrongRef() 做一些收尾的工作。
- 弱指针 wp 最大的特点就是不能直接操作目标对象，这是因为 wp 没有重载 . 和->操作符号，而 sp 重载了这两个操作符号。wp 要操作目标对象，必须要通过 `promote()` 方法升级为强指针
  - promote升级成功，则强引用计数+1，弱引用计数+1。
  - 目标对象可能被释放掉或其他原因都会导致升级失败。

<a name="0ebdd31c"></a>
## 4.3 flags 与生命周期

- 强引用控制生命周期：flags = OBJECT_LIFETIME_STRONG。
  - 强引用控制实际对象的生命周期，弱引用控制 weakref_impl对象的生命周期。
  - 强引用计数 = 0时，decStrong() 中 delete `实际对象`。此时要使用 wp 指针调用对象，必须要升级为 sp。
  - 弱引用计数 = 0时，decWeak() 中 delete `weakref_impl` 对象。
- 弱引用控制生命周期：flags = OBJECT_LIFETIME_WEAK。
  - 强引用计数 = 0时，弱引用计数 ≠ 0 ，实际对象不会被 delete。
  - 弱引用计数 = 0时，`实际对象`和 `weakref_impl` 对象会被同时 delete。其中 decWeak() 中 delete 实际对象， delete 实际对象会调用 RefBase 的析构函数，此时弱引用计数=0，weakref_impl也会被delete。

<a name="2424932f"></a>
## 4.4 sp/wp 优缺点

sp/wp 的优点：

- C++层操作完对象后，不需要再手动delete对象，系统会自动回收对象。

sp/wp 的缺点：

- 增加了引用计数的管理对象 weakref_impl，执行效率会比普通的指针低。


<a name="eUIzg"></a>
# 五、参考文献

- Gityuan 博客：[理解Refbase强弱引用](http://gityuan.com/2015/12/05/android-refbase/)
- 深入理解Android卷I
