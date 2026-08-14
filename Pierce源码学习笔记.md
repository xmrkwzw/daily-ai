# Pierce 源码学习笔记

> 记录从 Java 层 `Pierce.hook()` 到 native 层 ArtMethod 的学习过程。
> 每一步由浅入深，围绕 m4399-Pierce 工程实际代码展开。

---

## 第一课：Java 层 `Pierce.hook()` 完整调用链

### 一、三个入口（重载，逐层收口）

```java
// 入口1：按类名 + 方法名（Pierce.java:290）
hook(ClassLoader, String className, String methodName, MethodHook callback)
    → classLoader.loadClass(className) 后交给入口3

// 入口2：按 Class + 方法名（Pierce.java:310）
hook(Class clazz, String methodName, MethodHook callback)
    → 遍历 clazz.getDeclaredMethods() 找同名方法，交给入口3

// 入口3：核心（Pierce.java:342）
hook(Member method, MethodHook callback, boolean canInitDeclaringClass)
```

三个入口最终都汇聚到 `Pierce.java:342` 这个核心方法。前两个只是"帮你把 Method 找出来"的便利包装。

### 二、核心 hook() 的六步（Pierce.java:342-393）

| 步骤 | 代码位置 | 动作 |
|---|---|---|
| 1. 前置校验 | 344-362 | disableHooks 直接返回；判 null、判 abstract（抽象方法无 body 不能 hook）、判 `<clinit>`（静态初始化器不能 hook）、setAccessible(true) 绕过访问控制 |
| 2. 确保初始化 | 364 | ensureInitialized() → 首次调用走 initialize()（loadLibrary + init0） |
| 3. 取 ArtMethod 指针 | 373 | `long artMethod = PierceNative.getArtMethod(method)` —— 关键桥梁，把 Java 的 Method 对象换成 native 的 ArtMethod 结构体地址 |
| 4. 登记 HookRecord | 377-384 | 用 sHookLock 同步，去 sHookRecords（ConcurrentHashMap<Long, HookRecord>，Pierce.java:39）查；没有就 new HookRecord(method, artMethod) 存进去。**同一个 ArtMethod 只登记一次** |
| 5. 交给 handler | 386-387 | sHookHandler.handleHook(...) |
| 6. 回调监听 | 389-392 | hookListener.afterHook，返回 Unhook |

### 三、handleHook → hookNewMethod（Pierce.java:46-66 / 395-462）

默认 handler 在 `Pierce.java:46`。关键逻辑：

```java
if (newMethod) {
    hookNewMethod(hookRecord, hook, modifiers, canInitDeclaringClass);  // 首次才真正 hook
}
hookRecord.addCallback(hook);  // 无论首次与否，都把 callback 加进回调集合
```

这就是 **"一次 ArtMethod 只改一次入口，多个 callback 共享一个 HookRecord"** 的设计。第二次 hook 同一个方法，不再动入口，只是往 HookRecord.callbacks 里塞一个新的 MethodHook。

`hookNewMethod`（Pierce.java:395）的核心动作：

```java
boolean isInlineHook = mode == INLINE || mode == INLINE_WITHOUT_JIT;   // 398
if (isStatic && canInitDeclaringClass) resolve((Method) method);       // 401，静态方法先触发类初始化
final boolean jni = Modifier.isNative(modifiers);                      // 416
final boolean proxy = Proxy.isProxyClass(declaring);                   // 417
// inline 模式且非 jni/proxy 时尝试 compile0，失败回退 replacement
hookRecord.paramTypes = ...; hookRecord.paramNumber = ...;             // 436-442
genereateDexAndLoad(hookRecord, hook);                                 // 444，生成 bridge/backup dex
Method backup = PierceNative.hook0(...);                               // 448，native 边界！
```

`hook0`（Pierce.java:448）是**整条链路里 Java → native 的交接点**，从这开始进入 C++ 层。

### 四、genereateDexAndLoad（Pierce.java:464-585）：运行时造 bridge/backup

这是 Pierce 最核心的 Java 侧机制。流程：

```java
// 1. 算 shorty（如 "VL" = void 返回 + 1 个对象参数）478, 600-614
// 2. 拼唯一 dex 文件名/类名/方法名 481-485
// 3. 调 native generateDex 生成 dex 文件 494
// 4. 用 DexClassLoader 加载这个 dex 525
// 5. loadClass 拿到生成的类，反射设 artTargetMethod 字段 534-541
// 6. 从类里找出 bridge 和 backup 两个 Method 543-557
// 7. 存进 hookRecord.bridge / hookRecord.backup 579-580
```

必须点透的两个概念（本课知识储备重点）：

- **bridge 方法**：生成的"新入口"。目标方法被 hook 后，entry_point 指向它，所有真实调用先进这里，再分发到你注册的 callback。
- **backup 方法**：生成的"原方法备份"。它才是真正执行目标方法原本逻辑的载体。hook 之后，原方法 body 已经不在原 ArtMethod 里了，靠 backup 调回。

`generateDex` 是 native 方法（PierceNative.java:59），用 slicer/DexBuilder 在 C++ 里手写 dex 字节码。

### 五、回调分发 handleCall（Pierce.java:1008-1077）

这是 hook 生效后，每次目标方法被调用时实际走的分发逻辑（before → 原方法 → after）：

```java
// 1026-1042：依次调所有 callback.beforeCall
// 1045-1051：若没 returnEarly，调 invokeOriginalMethod()（即走 backup）
// 1054-1070：倒序调 callback.afterCall
// 1073-1076：抛异常 or 返回结果
```

### 全景链路图

```
hook(Member, callback)
  └─ ensureInitialized()              // 首次：loadLibrary + init0
  └─ getArtMethod(method)             // Java Method → native ArtMethod 指针
  └─ 登记 HookRecord（一次 ArtMethod 一次）
  └─ hookNewMethod()
       ├─ resolve 静态方法 / compile 尝试
       ├─ genereateDexAndLoad()       // 运行时生成并加载 bridge/backup dex
       └─ hook0(...)  ←———— Java/native 边界
             （第二课：ART ArtMethod 结构 + entry_point 改写）
```

### 第一课核心记忆点

1. `getArtMethod` 拿到的 `long` 是 `ArtMethod` 结构体在内存里的地址。
2. 一个目标方法只有一份 HookRecord + 一个 bridge + 一个 backup，多个 callback 共用。
3. `hook0` 是 Java 与 native 的分界，往下全在 C++。

---

## 第二课：ArtMethod 是什么

### 一、定义

**ART 虚拟机里，每一个 Java 方法（含构造器）在 C++ 堆内存里都对应一个 `art::ArtMethod` 结构体，它是"方法"在运行时真正的实体。**

Java 层拿到的 `java.lang.reflect.Method` 对象只是一个**壳**。壳里面藏了一个指针，指向这个 native 结构体。方法一旦被加载到内存，它的"身份"就由这个结构体决定，跟 Java 反射对象无关。

所以第一课 `getArtMethod(method)` 拿到的那个 `long`，本质就是**这个结构体在内存里的起始地址**（pine.cpp:371-374）。整个 hook 就是围着这个地址做文章。

### 二、结构体关键字段

ArtMethod 结构体里有几个决定方法"如何被执行"的字段，是 hook 的全部抓手：

| 字段 | 类型 | 作用 | 对 hook 的意义 |
|---|---|---|---|
| `access_flags_` | uint32 | 方法的修饰标志（static/native/private/构造器…） | 判断方法类型、决定 hook 策略 |
| `entry_point_from_compiled_code_` | void* | 已编译机器码的入口地址 | **hook 的核心目标**：改它 = 劫持调用 |
| `entry_point_from_jni_` | void* | JNI 方法的 native 入口 | hook JNI 方法时改它 |
| `entry_point_from_interpreter_` | void* | 解释器入口 | 反优化时用 |
| `declaring_class` | uint32 | 所属类（GCRoot） | 定位类、更新声明类 |

其中 **`entry_point_from_compiled_code_` 就是 hook 的命门**：ART 每次调用一个方法，都会读这个字段、跳转到它指向的机器码去执行。你把它改成一个"跳板"的地址，所有对原方法的调用就会被拦下来——这就是"entry-point replacement"模式的本质（对应 Java 层 HookMode.REPLACEMENT）。

### 三、本工程怎么"看到"它——不硬编码结构体

这是整份代码最关键的工程决策。

Android 每个大版本的 ArtMethod **字段排列顺序、偏移量都不一样**（Android 5.0 的 entry_point 是 8 字节，Android 6 起才改成指针；N 之前是 4 字节对齐，O 之后对齐规则又变）。如果在 C++ 里写死一个 struct：

```cpp
struct ArtMethod {
    uint32_t access_flags_;
    void* entry_point_...;   // ← 偏移在 Android 5 和 Android 13 上完全不同，直接崩溃
};
```

那么一份 .so 只能跑在一个版本上。所以工程的做法是——**不定义真实 struct，而是维护一张"字段名 → 偏移量"的映射表，偏移量在运行时动态探测出来。**

载体就是两个东西：

1. **ArtMethod 类**（art_method.h:24）：只有方法，没有真实字段。所有字段都以 `static Member<ArtMethod, void*> entry_point_from_compiled_code_;` 的形式声明（art_method.h:344）。

2. **Member 模板**（member.h:12）：它只存一个 `int32_t offset`，加两个操作：
   - `Get(instance)` → safeReadValue → `memcpy((char*)instance + offset, ...)` 读（member.h:31）
   - `Set(instance, value)` → memcpy((char*)instance + offset, ...) 写（member.h:43）

   本质就是**"基址 + 偏移"的指针运算**，没有魔法。所谓"读取 entry_point_from_compiled_code_"，翻译成机器行为就是：**拿 ArtMethod 的地址 + 探测到的偏移，从那个内存位置读 8 个字节出来当指针用。**

### 四、偏移量怎么探测（InitMembers）

`art_method.cpp:125` 的 `InitMembers(m1, m2, m3, access_flags)` 负责在初始化阶段把这张表填出来。核心手法是**内存扫描**：

- **算结构体大小**：`size = |m2 - m1|`，用两个相邻 ArtMethod 的地址差得出每个方法占多少字节（art_method.cpp:139）。
- **找 access_flags_**：用 Memory::FindOffset（memory.h:31）在 m1 的内存里从 0 开始逐字节（步长 2）扫，找哪个位置的值恰好等于已知的 access_flags → 那个位置就是偏移。
- **找 entry_point 系列**：更精妙——传进来的 m1 是一个 native 方法，它的 entry_point_from_jni_ 指向一个已知函数 Ruler_m1。扫描找 Ruler_m1 这个指针值出现的位置，就定位到 entry_point_from_jni_，再 + sizeof(void*) 顺藤摸瓜得到 entry_point_from_compiled_code_（art_method.cpp:198-247）。

如果扫描失败（某些冷门 ROM），就回退到一张**写死的默认偏移表**（art_method.h:263-321，按 Android 版本 + 32/64 位穷举）。所以代码有两道保险：**优先动态探测，失败用默认表兜底。**

### 五、getArtMethod 拿到指针的两条路（art_method.cpp:99）

```cpp
ArtMethod* FromReflectedMethod(JNIEnv* env, jobject javaMethod) {
    if (Android::version >= Android::kR) return GetArtMethodForR(env, javaMethod);
    return reinterpret_cast<ArtMethod*>(env->FromReflectedMethod(javaMethod));  // < R
}
```

- **Android R 之前**：JNI 的 FromReflectedMethod 返回的 jmethodID **就是** ArtMethod 指针，直接强转。
- **Android R 及之后**：Google 改了 jmethodID 的结构（不再直接是 ArtMethod 指针，还带了标志位），所以改用反射读 Java 对象 Executable 里的 artMethod 字段（art_method.h:35-42）。

这就是为什么 pine.cpp:371 的 Pine_getArtMethod 能拿到一个 long 地址返回给 Java 层。

### 六、本课必须掌握的清单

| # | 必须掌握的点 | 判断标准 |
|---|---|---|
| 1 | ArtMethod 是"方法"在 ART 里的 native 实体，Java 的 Method 只是壳 | 能解释 getArtMethod 返回的 long 是什么 |
| 2 | entry_point_from_compiled_code_ 是 hook 的命门，改它 = 劫持调用 | 能说出 REPLACEMENT 模式改的是哪个字段 |
| 3 | 本工程不硬编码 struct，用"Member 偏移表 + 动态扫描"适配多 Android 版本 | 能说清为什么不能写死 struct、偏移从哪来 |

这三点是后续所有课（entry_point 改写、trampoline、生成 dex）的**地基**。第三点尤其关键，它解释了为什么 art_method.cpp 里到处都是 SetOffset、GetOffset 而不是直接 .field 访问。

---

## 附：本课涉及的关键源码位置

| 文件 | 关键内容 |
|---|---|
| `module-pierce/src/main/java/com/m4399/pierce/core/Pierce.java` | Java 层 hook 主入口、HookRecord、genereateDexAndLoad、handleCall |
| `module-pierce/src/main/java/com/m4399/pierce/core/PierceNative.java` | 所有 native 方法声明 |
| `module-pierce/src/main/cpp/art/art_method.h` | ArtMethod 类、字段 Member 声明、默认偏移表 |
| `module-pierce/src/main/cpp/art/art_method.cpp` | InitMembers 偏移探测、FromReflectedMethod、BackupFrom/AfterHook |
| `module-pierce/src/main/cpp/utils/member.h` | Member 模板（基址+偏移读写） |
| `module-pierce/src/main/cpp/utils/memory.h` | Memory::FindOffset / safeReadValue |
| `module-pierce/src/main/cpp/pine.cpp` | Pine_getArtMethod 等 JNI 实现 + gMethods 注册表 |

---

## 第三课：答疑澄清（针对 ArtMethod 的三个提问）

### 问1：本工程 hook 的核心是不是就是改 entry_point_from_compiled_code 入口地址为自定义函数，从而改变方法执行？

答：对 REPLACEMENT 模式而言是，但"整个工程"不止这一种手法。精确分三种：

| hook 手法 | 修改位置 | 触发条件 |
|---|---|---|
| REPLACEMENT（入口替换） | 改 entry_point_from_compiled_code_ 指向 bridge | 默认模式（Android 8+）、INLINE 失败回退 |
| INLINE（内联跳板） | 不改字段，覆盖已编译机器码前几条指令为跳转 | INLINE / INLINE_WITHOUT_JIT 且方法已编译 |
| JNI 入口替换 | 改 entry_point_from_jni_ | 目标是 native 方法 |

所以"改 entry_point_from_compiled_code_ 为自定义函数地址"是 REPLACEMENT 模式的核心本质，也是 Android 8+ 的默认主路径；但 INLINE 模式改的是机器码本身，根本不动这个字段。

### 问2：为什么别的解释说是 entry_point_from_quick_compiled_code_？

答：是同一个字段，年代不同、名字不同：

- Android 5.0/5.1 时代 ART 用 quick 编译器，字段名是 entry_point_from_quick_compiled_code_。
- Android 6.0 起编译体系重构（quick/portable 区分被废弃），字段重命名为 entry_point_from_compiled_code_。

两者指同一件事：已编译机器码的入口。你看到的旧资料（Android 5.x hook 教程）用的是旧名。

工程内部证据：成员变量命名 entry_point_from_compiled_code_（art_method.h:344），但默认偏移函数保留旧名 GetDefaultEntryPointFromQuickCompiledCodeOffset（art_method.h:293）——作者跟着 AOSP 命名演化走的痕迹。

### 问3：你说拿偏移是通过"反射读 Executable 的 artMethod 字段"，最后又说"不硬编码 struct、用 Member 偏移表 + 动态扫描"，两者什么关系？

答：是递进的两层，不矛盾，之前讲混了，澄清如下：

**第一层：找到 ArtMethod 这个对象的地址（FromReflectedMethod，art_method.cpp:99）**
- Android R 之前：env->FromReflectedMethod(javaMethod) 返回的 jmethodID 就是 ArtMethod 指针，直接强转。
- Android R 及之后：反射读 Java 对象 Executable 的 artMethod 字段（GetLongField）。

这一步只解决一件事：从一个 Java Method 对象，找到它对应的 ArtMethod 结构体在内存的起始地址。

**第二层：找到 ArtMethod 内部某个字段的偏移（Member 偏移表 + 动态扫描）**
- 拿到 ArtMethod 起始地址后，因为不同 Android 版本内部字段排列不同，不写死 struct，而是动态扫描出每个字段相对起始地址的 offset，再用 Member 模板"基址 + offset"读写。

这一步解决的是：知道 ArtMethod 地址后，怎么安全读写它内部某个字段（比如 entry_point_from_compiled_code_）。

比喻：第一层 = 找到这栋楼的地址（ArtMethod 指针）；第二层 = 找到楼里某房间的门牌号（字段相对偏移）。先有楼地址，再有门牌号，才能进房间改东西。

### 问4：getArtMethod 具体怎么拿到 ArtMethod 地址？

关键澄清：getArtMethod 这一行没有"动态扫描"，它只是把地址从 Java 对象里"抠"出来。真正的动态扫描发生在 InitMembers（另一处）。完整链路 5 步：

| 步骤 | 位置 | 动作 |
|---|---|---|
| 1 | Pierce.java:373 | Java 调 PierceNative.getArtMethod(method) |
| 2 | pine.cpp:596 | 注册表把 Java 名 getArtMethod 映射到 C 函数 Pine_getArtMethod |
| 3 | pine.cpp:371-374 | Pine_getArtMethod 调 ArtMethod::FromReflectedMethod，返回的指针强转成 jlong 回 Java |
| 4 | art_method.cpp:99-104 | FromReflectedMethod 分两条路 |
| 5 | art_method.h:35-42 | R 及之后走 GetArtMethodForR 反射读字段 |

第 4 步两条路是核心：

```cpp
// art_method.cpp:99
ArtMethod* FromReflectedMethod(JNIEnv* env, jobject javaMethod) {
    if (Android::version >= Android::kR) return GetArtMethodForR(env, javaMethod);   // R+
    return reinterpret_cast<ArtMethod*>(env->FromReflectedMethod(javaMethod));        // < R
}
```

- Android R 之前：JNI 标准 API env->FromReflectedMethod(javaMethod) 返回的 jmethodID 本身就是 ArtMethod 指针，直接强转，一行搞定，没有任何扫描。
- Android R 及之后：GetArtMethodForR（art_method.h:35-42）用 env->GetLongField(javaMethod, ...artMethod) 读 Java 对象 Executable 里的 artMethod 字段。这个字段 ID 在 WellKnownClasses::Init（well_known_classes.cpp:11-16）里用标准 API GetFieldID("java/lang/reflect/Executable", "artMethod", "J") 拿到——也是查询，不是扫描。

结论：getArtMethod 拿地址 = JNI 标准 API 查询，全程无内存扫描。之前记的"动态扫描找偏移"是 InitMembers 干的事，作用对象是"ArtMethod 内部的字段偏移"，跟"拿 ArtMethod 地址"是两回事。

一句话区分：
- 找这栋楼在哪 → getArtMethod → JNI 查询
- 找楼里房间门牌号 → InitMembers → 内存扫描

### 问5：bridge / backup 是 C 还是 Java 还是工程提出的？

三者都不是语言概念，是 hook 框架的设计概念。

| 维度 | 归属 |
|---|---|
| 概念提出 | Pine（canyie/Pine）框架的架构设计，本工程是 Pine 二次开发，继承而来 |
| 落地形式 | Java 方法——HookRecord 里是 public Method bridge; public Method backup;（Pierce.java:1180-1181），是 Java 反射对象 |
| 生成方式 | genereateDexAndLoad 时，用 C++ 里的 slicer 库手写一个 dex，DexClassLoader 加载后得到这两个 Java 方法 |
| 底层实体 | 每个 Java 方法加载后都有对应的 ArtMethod，所以 bridge/backup 也各有一个 ArtMethod 结构体 |

不是 ART 虚拟机自带的，也不是 C/C++ 语言特性——是 Pine 这套 hook 方案自己设计出来的两个"中间方法"，用来解决"原方法被改了入口后，还得有个地方保留原逻辑（backup）、有个地方先拦下来分发回调（bridge）"的问题。

同类框架也都有这个思路：YAHFA、SandHook、Whale 的 backup/trampoline 概念一脉相承。

---

## 第四课：REPLACEMENT 模式 native 链路（hook0 → BackupFrom / AfterHook）

> 讲 target、bridge、backup 三者如何在 native 层被串起来。

### 一、hook0 开头：拿到三个 ArtMethod（pine.cpp:186-204）

```cpp
jobject Pine_hook0(...) {
    ...
    auto target = art::ArtMethod::FromReflectedMethod(env, javaTarget);   // 202 原方法
    auto bridge = art::ArtMethod::FromReflectedMethod(env, javaBridge);   // 203 桥接方法
    auto backup = art::ArtMethod::FromReflectedMethod(env, javaBackup);   // 204 备份方法
```

Java 层 hook0 传进来的三个 Method 对象，在这里各自转成对应的 ArtMethod 指针。三个实体齐了，后面就围绕它们做文章。

### 二、判断 inline 还是 replacement（pine.cpp:226-247）

```cpp
bool is_inline_hook = JBOOL_TRUE(isInlineHook);   // 226，Java 层算好的
if (is_inline_hook && (trampoline_installer->IsReplacementOnly() || !target->IsCompiled())) {
    is_inline_hook = false;    // 233-240：目标没编译/仅支持替换 → 强制回退 replacement
}
if (is_inline_hook && trampoline_installer->CannotSafeInlineHook(target)) {
    is_inline_hook = false;    // 242-247：不安全 → 回退
}
```

Android 8+ 默认走 REPLACEMENT（Java 层 hookMode 已定）。

### 三、挂起 VM（pine.cpp:257）

```cpp
ScopedSuspendVM suspend_vm(thread);   // 关键！修改前先 STW
```

ArtMethod 是多线程共享的全局数据结构，直接改会跟其他线程读取产生竞态（可能读到改了一半的入口地址导致崩溃）。必须先挂起所有线程（stop-the-world），改完再恢复。这是 hook 库通用铁律。

### 四、改 target 入口（trampoline_installer.cpp:165-188）

replacement 分支调 InstallReplacementTrampoline(target, bridge)，核心三行：

```cpp
void *target_origin_code_entry = target->GetEntryPointFromCompiledCode();   // 168：先存原入口
void *method_jump_trampoline = CreateMethodJumpTrampoline(bridge);          // 169：造跳向 bridge 的跳板
target->SetEntryPointFromCompiledCode(method_jump_trampoline);              // 181：改 target 入口！
return target_origin_code_entry;                                            // 187：原入口作为返回值
```

这里就是 hook 命门的落地：SetEntryPointFromCompiledCode 把 target 的 entry_point_from_compiled_code_ 从"原机器码"改成"跳板"。此后任何对原方法的调用，都会先跳到 trampoline，再由 trampoline 跳进 bridge。

细节：改的是跳板（trampoline）地址，不是直接改成 bridge 入口——trampoline 是一段纯汇编，负责摆好寄存器/调用约定后再跳 bridge。

返回值 target_origin_code_entry（原机器码入口）就是传给 BackupFrom 的 call_origin（pine.cpp:259）。

### 五、BackupFrom 串起 backup（art_method.cpp:250-308）

```cpp
backup->BackupFrom(target, call_origin, is_inline_hook, is_native, is_proxy);  // pine.cpp:270
```

BackupFrom 干三件事：

① 复制（art_method.cpp:251-255）
```cpp
if (copy_from) copy_from(this, source, sizeof(void*));
else memcpy(this, source, size);
```
把 target 的全部字段复制到 backup，让 backup 拥有原方法的 access_flags、declaring_class、code_item 等身份信息。

② 改 access_flags（art_method.cpp:257-268）
```cpp
if ((access_flags & kStatic) == 0) {
    access_flags &= ~(kPublic | kProtected);   // 去掉 public/protected
    access_flags |= kPrivate;                  // 改成 private
}
access_flags &= ~kConstructor;                 // 去掉 constructor 标志
```
把 backup 伪装成一个可直接调用的 private 实例方法，避免 ART 调用时做多余检查。

③ 设入口为原机器码（art_method.cpp:300-301）
```cpp
SetEntryPointFromCompiledCode(entry);   // entry = call_origin = 原方法机器码入口
```
这是 backup 的灵魂：backup 复制了原方法内容，但入口必须重新指向原方法机器码，这样调用 backup = 执行原方法原本逻辑。

中间还有 JIT info 引用处理（art_method.cpp:280-307），目的是防止 GC 回收原机器码导致 backup 悬空，属进阶细节，本课先记住"它在保护 backup 不指向被回收内存"即可。

### 六、AfterHook 修 target（art_method.cpp:310-354）

```cpp
target->AfterHook(is_inline_hook, is_native_or_proxy);   // pine.cpp:271
```

AfterHook 只改 target 的 access_flags，不碰入口（入口已在第四步改过）：

```cpp
access_flags |= kAccCompileDontBother;   // 315-318：禁止 JIT 再编译 target
// debug 模式加 kNative，防止 ART 强制走解释器忽略入口  320-327
access_flags &= ~kFastInterpreterToInterpreterInvoke;    // 329-334：刷新快速解释器缓存
access_flags &= ~kFastNative; ... ~kCriticalNative;      // 336-348：native 方法去 fast 标志
SetAccessFlags(access_flags);                            // 350
```

核心是 kAccCompileDontBother——不加的话 JIT 可能重新编译 target 并把入口重置回原机器码，hook 就失效了。

### 七、三者串联全景图

```
外部调用 target（原方法）
        │
        ▼
  entry_point_from_compiled_code_  ← 已被改成 trampoline
        │
        ▼
  trampoline（汇编跳板）
        │
        ▼
  bridge（桥接方法，新入口）
        │
        ▼
  handleCall（Java 层回调分发）
        │ 调 callback.beforeCall
        │ 调 backup.invoke
        │ 调 callback.afterCall
        ▼
  backup（备份方法，入口 = 原机器码）
        │
        ▼
  原方法真实机器码（call_origin）
```

三者分工一句话：
- target：入口被"改道"，从原机器码 → 跳板。
- bridge：新的"接待大厅"，所有调用先到它，再走 handleCall 分发回调。
- backup：复制了原方法全部字段、入口指向原机器码，负责执行真正的原逻辑。

### 八、本课记忆点

| # | 必须掌握 |
|---|---|
| 1 | hook 前必须 ScopedSuspendVM 挂起所有线程，防竞态 |
| 2 | REPLACEMENT = SetEntryPointFromCompiledCode 把 target 入口改成 trampoline，trampoline 跳 bridge |
| 3 | BackupFrom = 复制 + 改 flags + 设入口为原机器码，让 backup 保留原逻辑 |
| 4 | AfterHook = 改 target 的 flags，核心是 kAccCompileDontBother 防 JIT 重置入口 |
