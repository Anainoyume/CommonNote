## constexpr Union 的限制

 C++ 标准规定：在一个 `constexpr` 构造函数中，如果所构造的类型是 `union`，**必须**在初始化列表里初始化它的**某一个**非静态成员。

------

## 问题演示

```cpp
struct nontrivial {
    constexpr nontrivial(int o) : u{o} {}
    int u;
};

union storage {
    nontrivial nt;  // nontrivial 没有默认构造函数
};

struct optional {
    storage s;      // 错误！storage 没有默认构造函数
};
```

1. `nontrivial` 提供了一个接受 `int` 的 `constexpr` 构造函数，**但没有**默认构造函数。

2. `storage` 因为没有任何成员能平凡默认构造，所以**没有**隐式生成的默认构造函数。

3. `optional` 里声明了一个 `storage s;`，尝试默认构造时，编译器报错：

   > “`storage` 没有可用的默认构造函数”

------

## 失败的尝试

```cpp
union storage {
    constexpr storage() {}  // 想手写一个默认构造
    nontrivial nt;
};
```

- 这个写法违背了标准：在 `constexpr` 构造函数里，**如果构造类型是 `union`**，**必须**在初始化列表里初始化其中一个成员。
- 上述 `storage() {}` 既没有写 `: nt(/*…*/)`，也没有初始化任何其他成员，所以**不合法**。

------

## 正确的解决方案

引入一个**空虚拟成员**，保证默认构造时至少有一个平凡类型被初始化：

```cpp
struct _Empty_byte {};  // 平凡、constexpr 可构造

union storage {
    constexpr storage()
      : empty{}    // ——初始化 empty，满足“至少初始化一个成员”
    {}

    _Empty_byte empty;  // “占位”成员，无实际开销
    nontrivial  nt;     // 真正需要延迟构造的成员
};
```

- `storage()` 的初始化列表里写了 `: empty{}`，**合法**地初始化了第一个成员。
- `_Empty_byte` 是一个完全平凡、零开销的类型；它的默认构造在任何上下文里都可用。
- 真正功能性的 `nt` 成员，可以通过手写其他构造或 placement-new 在需要时再激活。

------

## 与 `Uninitialized<Type, false>` 的关系

在你的 `Uninitialized<Type, false>` 特化中，也使用了同样的**占位概念**：

```cpp
struct _Empty_byte { };

union {
    _Empty_byte M_empty;   // “占位”成员，保证未来添加默认构造时至少有一个成员可初始化
    Type        M_storage; // 真正存储用户类型
};
```

这样做的好处：

1. **满足 `constexpr` Union 的标准要求**
    即当你手写 `constexpr` 默认构造时，可以写成 `: M_empty{}`，合法初始化一个平凡成员。
2. **为未来扩展预留空间**
    如果以后需要让 `Uninitialized` 支持无参 `constexpr` 构造，就不必再改动 `union` 布局——只要初始化 `M_empty` 即可。
3. **遵循标准库模式**
    类似于 `std::optional`/`std::variant` 中使用的“空字节”占位技巧，兼顾“延迟构造”和“`constexpr` 友好”两大需求。

------

### 小结

- **`constexpr` Union 构造**：必须在 `constexpr` 构造函数的初始化列表里初始化某个成员。
- **空虚拟成员**：用一个平凡类型占位，保证默认构造合法、无额外开销。
- **通用模式**：这是标准库（如 `std::optional`）中常见的实现手法，用于手动管理生命周期且兼顾编译期常量上下文。

以上即为“为什么在 `union` 中要特地放一个空成员 `_Empty_byte`”的最终整理。复制即可直接使用。

---

### 后记

实际实践中，发现类似下面这样的代码，也可以编译通过：

```cpp
struct nontrivial {
    constexpr nontrivial(int o) : u{o} {}
    int u;
};

union storage {
    constexpr storage() {}
    nontrivial  nt;     
};

struct optional {
    storage s;      
};

void foo() {
    optional o;
}
```

这考虑到可能是不能编译器实现的时候比较宽松，但标准是严格要求 constexpr 构造函数必须保证 union 有一个活跃的成员。