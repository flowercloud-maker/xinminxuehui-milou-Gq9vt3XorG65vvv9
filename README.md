
　　右值引用 移动语义 完美转发具体是什么，就不说了，网上一搜一大堆，主要介绍下std::move和std::forward


## std::move std::forward


　　查下源码，gcc版本:gcc version 7\.3\.0 (GCC),grep \-r "forward(" /usr/include/c\+\+/7\.3\.0/bits/,move和forward都在/usr/include/c\+\+/7\.3\.0/bits/move.h文件中，源码如下：


　　


![](https://images.cnblogs.com/OutliningIndicators/ContractedBlock.gif)![](https://images.cnblogs.com/OutliningIndicators/ExpandedBlockStart.gif)


```
/**
 92    *  @brief  Convert a value to an rvalue.
 93    *  @param  __t  A thing of arbitrary type.
 94    *  @return The parameter cast to an rvalue-reference to allow moving it.
 95   */
 96   template
 97     constexpr typename std::remove_reference<_Tp>::type&&
 98     move(_Tp&& __t) noexcept
 99     { return static_cast::type&&>(__t); }
 
 
 /**
 66    *  @brief  Forward an lvalue.
 67    *  @return The parameter cast to the specified type.
 68    *
 69    *  This function is used to implement "perfect forwarding".
 70    */
 71   template
 72     constexpr _Tp&&
 73     forward(typename std::remove_reference<_Tp>::type& __t) noexcept
 74     { return static_cast<_Tp&&>(__t); }
 75
 76   /**
 77    *  @brief  Forward an rvalue.
 78    *  @return The parameter cast to the specified type.
 79    *
 80    *  This function is used to implement "perfect forwarding".
 81    */
 82   template
 83     constexpr _Tp&&
 84     forward(typename std::remove_reference<_Tp>::type&& __t) noexcept
 85     {
 86       static_assert(!std::is_lvalue_reference<_Tp>::value, "template argument"
 87             " substituting _Tp is an lvalue reference type");
 88       return static_cast<_Tp&&>(__t);
 89     }
```


move forward
　　本质就是强制类型转换，move并不进行所谓的“移动”


　　用c\+\+14实现一下，更简单，如下：


![](https://images.cnblogs.com/OutliningIndicators/ContractedBlock.gif)![](https://images.cnblogs.com/OutliningIndicators/ExpandedBlockStart.gif)


```
// C++14 version of std::move
template
constexpr decltype(auto)
move(_Tp&& __t) noexcept
{
    return static_cast&&>(__t);
}

// C++14 version of std::forward for lvalues
template
constexpr decltype(auto)
forward(std::remove_reference_t<_Tp>& __t) noexcept
{
    return static_cast<_Tp&&>(__t);
}

// C++14 version of std::forward for rvalues
template
constexpr decltype(auto)
forward(std::remove_reference_t<_Tp>&& __t) noexcept
{
    static_assert(!std::is_lvalue_reference_v<_Tp>, "template argument substituting _Tp is an lvalue reference type");
    return static_cast<_Tp&&>(__t);
}
```


c\+\+14 move forward
　　写了一个测试程序，如下：


　　


![](https://images.cnblogs.com/OutliningIndicators/ContractedBlock.gif)![](https://images.cnblogs.com/OutliningIndicators/ExpandedBlockStart.gif)


```
#include 
#include   // for std::move, std::forward
#include   // for remove_reference_t, is_lvalue_reference_v

// C++14 version of std::move
template
constexpr decltype(auto)
move(_Tp&& __t) noexcept
{
    return static_cast&&>(__t);
}

// C++14 version of std::forward for lvalues
template
constexpr decltype(auto)
forward(std::remove_reference_t<_Tp>& __t) noexcept
{
    return static_cast<_Tp&&>(__t);
}

// C++14 version of std::forward for rvalues
template
constexpr decltype(auto)
forward(std::remove_reference_t<_Tp>&& __t) noexcept
{
    static_assert(!std::is_lvalue_reference_v<_Tp>, "template argument substituting _Tp is an lvalue reference type");
    return static_cast<_Tp&&>(__t);
}

// Test class with move and copy constructors
class Widget {
public:
    Widget() { std::cout << "Widget default constructor\n"; }

    Widget(const Widget&) {
        std::cout << "Widget copy constructor\n";
    }

    Widget(Widget&&) noexcept {
        std::cout << "Widget move constructor\n";
    }
};

// Function to test std::forward
template 
void forward_test(T&& arg) {
    Widget w = std::forward(arg);
}

int main() {
    // Test std::move
    Widget widget1;
    std::cout << "Using std::move:\n";
    Widget widget2 = std::move(widget1);  // Should call move constructor

    // Test std::forward with lvalue
    std::cout << "\nUsing std::forward with lvalue:\n";
    Widget widget3;
    forward_test(widget3);  // Should call copy constructor

    // Test std::forward with rvalue
    std::cout << "\nUsing std::forward with rvalue:\n";
    forward_test(Widget());  // Should call move constructor

    return 0;
}
```


test
　　因为is\_lvalue\_reference\_v c\+\+17才支持，所以编译：g\+\+ test\_move\_forward.cpp \-o test\_move\_forward \-std\=c\+\+17


## 标签分发


　　　　有个全局的names,需要定义两个函数，一个是函数模板用的万能引用，一个函数的参数是普通的int(通过id检索到name，省略此实现)，代码如下：


　　


![](https://images.cnblogs.com/OutliningIndicators/ContractedBlock.gif)![](https://images.cnblogs.com/OutliningIndicators/ExpandedBlockStart.gif)


```
#include 
#include 
#include   // for std::forward
#include 

// 全局数据结构
std::unordered_setstring> names;

// 日志函数
void log(const char* message) {
    std::cout << "Log: " << message << std::endl;
}

// 模板版本
template
void logAndAdd(T&& name) {
    log("logAndAdd (perfect forwarding)");
    names.emplace(std::forward(name));
}

void logAndAdd(int idx) {
    log("logAndAdd (int version)");
    // 处理 int 类型的逻辑
}

int main() {
    std::string name = "Alice";
    int idx = 42;

    // 测试左值
    logAndAdd(name);  // 应该调用模板版本

    // 测试右值
    logAndAdd(std::string("Bob"));  // 应该调用模板版本

    // 测试 int 类型
    logAndAdd(idx);  

    // 测试 short 类型
    short idx2 = 222;
    logAndAdd(idx2); 

    return 0;
}
```


标签分发
　　上面的代码，没有测试 short 类型的那两行代码，是没问题的，但测试 short 类型的会匹配到完美转发那个函数，下面先用标签分发解决一下，代码如下：


　　


![](https://images.cnblogs.com/OutliningIndicators/ContractedBlock.gif)![](https://images.cnblogs.com/OutliningIndicators/ExpandedBlockStart.gif)


```
#include 
#include 
#include 
#include 
#include   // for std::forward, std::move>
#include <string>

// 全局数据结构
std::unordered_setstring> names;

// 日志函数
void log(const char* message) {
    auto now = std::chrono::system_clock::now();
    auto time = std::chrono::system_clock::to_time_t(now);
    std::cout << "Log [" << std::ctime(&time) << "]: " << message << std::endl;
}

// 完美转发版本
template
auto logAndAddImpl(T&& name) -> std::enable_if_t<
    !std::is_convertible_vint>,
    void
> {
    log("logAndAdd (perfect forwarding)");
    names.emplace(std::forward(name));
}

// 普通版本，专门处理 int 类型及其可隐式转换为 int 的类型
void logAndAddImpl(int idx) {
    log("logAndAdd (int version)");
    // 处理 int 类型的逻辑
    // 例如，将 int 转换为字符串并添加到集合中
    names.insert(std::to_string(idx));
}

// 分发函数
template
void logAndAdd(T&& name) {
    if constexpr (std::is_convertible_vint>) {
        logAndAddImpl(static_cast<int>(std::forward(name)));
    } else {
        logAndAddImpl(std::forward(name));
    }
}

// 额外的非模板版本，专门处理 int 类型
void logAndAdd(int idx) {
    logAndAddImpl(idx);
}

int main() {
    std::string name = "Alice";
    int idx = 42;
    short idx2 = 222;

    // 测试左值
    std::cout << "Testing lvalue:\n";
    logAndAdd(name);  // 应该调用完美转发版本

    // 测试右值
    std::cout << "\nTesting rvalue:\n";
    logAndAdd(std::string("Bob"));  // 应该调用完美转发版本

    // 测试 int 类型
    std::cout << "\nTesting int type:\n";
    logAndAdd(idx);  // 应该调用普通版本

    // 测试 short 类型
    std::cout << "\nTesting short type:\n";
    logAndAdd(idx2);  // 应该调用普通版本

    // 打印全局数据结构中的名字
    std::cout << "\nNames in the global set:\n";
    for (const auto& name : names) {
        std::cout << name << std::endl;
    }

    return 0;
}
```


标签分发2
　　


### SFINAE (enable\_if)


　　代码如下：


　　


![](https://images.cnblogs.com/OutliningIndicators/ContractedBlock.gif)![](https://images.cnblogs.com/OutliningIndicators/ExpandedBlockStart.gif)


```
#include 
#include 
#include 
#include 
#include   // for std::forward, std::move>
#include <string>

// 全局数据结构
std::unordered_setstring> names;

// 日志函数
void log(const char* message) {
    auto now = std::chrono::system_clock::now();
    auto time = std::chrono::system_clock::to_time_t(now);
    std::cout << "Log [" << std::ctime(&time) << "]: " << message << std::endl;
}

// 完美转发版本
template
auto logAndAdd(T&& name) -> std::enable_if_t<
    !std::is_convertible_vint>,
    void
> {
    log("logAndAdd (perfect forwarding)");
    names.emplace(std::forward(name));
}

// 普通版本，专门处理 int 类型及其可隐式转换为 int 的类型
template
auto logAndAdd(T&& idx) -> std::enable_if_t<
    std::is_convertible_vint>,
    void
> {
    log("logAndAdd (int version)");
    // 处理 int 类型的逻辑
    // 例如，将 int 转换为字符串并添加到集合中
    names.insert(std::to_string(static_cast<int>(idx)));
}

// 额外的非模板版本，专门处理 int 类型
void logAndAdd(int idx) {
    log("logAndAdd (int version)");
    names.insert(std::to_string(idx));
}

int main() {
    std::string name = "Alice";
    int idx = 42;
    short idx2 = 222;

    // 测试左值
    std::cout << "Testing lvalue:\n";
    logAndAdd(name);  // 应该调用完美转发版本

    // 测试右值
    std::cout << "\nTesting rvalue:\n";
    logAndAdd(std::string("Bob"));  // 应该调用完美转发版本

    // 测试 int 类型
    std::cout << "\nTesting int type:\n";
    logAndAdd(idx);  // 应该调用普通版本

    // 测试 short 类型
    std::cout << "\nTesting short type:\n";
    logAndAdd(idx2);  // 应该调用普通版本

    // 打印全局数据结构中的名字
    std::cout << "\nNames in the global set:\n";
    for (const auto& name : names) {
        std::cout << name << std::endl;
    }

    return 0;
}
```


SFINAE
　　还有一种方式模板特化，就不写代码了，写的脑壳疼


## 总结


　一入模板深似海，推荐两本书:Effective Modern C\+\+,C\+\+ Templates，有大佬有好的书，可以评论区推荐，感谢


 


 本博客参考[悠兔机场](https://xinnongbo.com)。转载请注明出处！
