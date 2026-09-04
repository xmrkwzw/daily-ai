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

| 步骤                 | 代码位置 | 动作                                                                                                                                                                          |
| -------------------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1. 前置校验          | 344-362  | disableHooks 直接返回；判 null、判 abstract（抽象方法无 body 不能 hook）、判 `<clinit>`（静态初始化器不能 hook）、setAccessible(true) 绕过访问控制                            |
| 2. 确保初始化        | 364      | ensureInitialized() → 首次调用走 initialize()（loadLibrary + init0）                                                                                                          |
| 3. 取 ArtMethod 指针 | 373      | `long artMethod = PierceNative.getArtMethod(method)` —— 关键桥梁，把 Java 的 Method 对象换成 native 的 ArtMethod 结构体地址                                                   |
| 4. 登记 HookRecord   | 377-384  | 用 sHookLock 同步，去 sHookRecords（ConcurrentHashMap<Long, HookRecord>，Pierce.java:39）查；没有就 new HookRecord(method, artMethod) 存进去。**同一个 ArtMethod 只登记一次** |
| 5. 交给 handler      | 386-387  | sHookHandler.handleHook(...)                                                                                                                                                  |
| 6. 回调监听          | 389-392  | hookListener.afterHook，返回 Unhook                                                                                                                                           |

### 二·附：initialize() 内部五步（Pierce.java:151-203）

> 六步表里第 2 步 `ensureInitialized()` 首次调用会走进 `initialize()`，这里是"Java 装前流程"里最后一块没展开的细节，按代码顺序逐条看。

```java
private static void initialize() {
    PierceFileLogUtils.init(PierceConfig.context);   // 152 初始化文件日志

    if (PierceConfig.disableHooks) {                 // 158 ① 全局禁用开关
        PierceLogUtils.alwaysLogI("initialize: no limit by disableHooks");
        return;                                      // 160 直接返回，.so 都不加载
    }

    int sdkLevel = PierceConfig.sdkLevel;            // 163
    if (sdkLevel < Build.VERSION_CODES.KITKAT) {     // 164 ② 版本下限校验
        String message = "Unsupported android sdk level " + sdkLevel;
        throw new RuntimeException(message);         // 167
    }

    String vmVersion = System.getProperty("java.vm.version");   // 170
    if (vmVersion == null || !vmVersion.startsWith("2")) {      // 171 ③ 确认是 ART
        throw new RuntimeException("Only supports ART runtime"); // 174
    }

    initGenerateDex(PierceConfig.context);           // 177 ④ 准备 dex 生成

    hookMode = sdkLevel < Build.VERSION_CODES.O
            ? HookMode.INLINE_WITHOUT_JIT : HookMode.REPLACEMENT;  // 179 ⑤ 定默认模式

    try {
        LibLoader libLoader = PierceConfig.libLoader;
        if (libLoader != null) libLoader.loadLib();  // 184 加载 libpierce.so
        PierceNative.init0(sdkLevel, ...);           // 186 扫描 ArtMethod 字段偏移
        if (PierceConfig.useFastNative && sdkLevel >= LOLLIPOP) {
            PierceNative.enableFastNative();         // 194 改 FastNative 减少 JNI 开销
        }
        disableJitInlineAfterInit();                 // 197 关闭 JIT 内联优化
    } catch (Exception e) {
        throw new RuntimeException("pierce init error", e);  // 201
    }
}
```

**五步的作用（按代码顺序）**：

| 步 | 代码 | 做什么 | 为什么 |
|----|------|--------|--------|
| ① | 158-161 | `disableHooks` 全局开关，true 直接 return | 全局禁用时连 .so 都不加载，省开销 |
| ② | 163-168 | `sdkLevel < KITKAT`（4.4）抛异常 | 4.4 之前是 Dalvik 虚拟机，没有 ArtMethod 结构，原理失效 |
| ③ | 170-175 | `java.vm.version` 不以 "2" 开头抛异常 | ART 版本号以 "2" 开头（Dalvik 是 "1.x"），这是判断当前跑在哪个虚拟机的土办法 |
| ④ | 177 | `initGenerateDex(context)` | 找 HandleCallWrapClass 里的 `handleCall` 方法 + 建 `pierce_gen_dex` 目录（第五课讲的 dex 生成前置准备） |
| ⑤ | 179-197 | 定 hookMode + loadLib + init0 + 关 JIT 内联 | 定默认模式、加载 .so、init0 扫描偏移、关内联防 hook 被绕过 |

**关键点**：

- **②③ 是在"选虚拟机"**：Android 4.4~5.0 之间 ART 和 Dalvik 并存，所以既查 `sdkLevel`（4.4 分界）又查 `vmVersion`（"2" 开头才是 ART），双重确认跑在 ART 上。
- **⑤ 的 hookMode 决策是全局一次性的**（179 行），和 `setHookMode` 里 AUTO 分支（Pierce.java:214-227）逻辑完全一致：`sdkLevel < O` 用 INLINE_WITHOUT_JIT（O 以下有 sharpening 优化，改入口不生效），`>= O` 用 REPLACEMENT（更稳）。
- **init0（186 行）是 Java 踏入 C 的第一脚**：native 层在这里扫描 ArtMethod 结构、动态定位 `entry_point_from_compiled_code_`/`access_flags_` 等字段的偏移（第七课讲的 InitMembers）。没有这一步，后面 hook0 根本不知道字段在内存哪个位置。

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

| 字段                              | 类型   | 作用                                            | 对 hook 的意义                       |
| --------------------------------- | ------ | ----------------------------------------------- | ------------------------------------ |
| `access_flags_`                   | uint32 | 方法的修饰标志（static/native/private/构造器…） | 判断方法类型、决定 hook 策略         |
| `entry_point_from_compiled_code_` | void\* | 已编译机器码的入口地址                          | **hook 的核心目标**：改它 = 劫持调用 |
| `entry_point_from_jni_`           | void\* | JNI 方法的 native 入口                          | hook JNI 方法时改它                  |
| `entry_point_from_interpreter_`   | void\* | 解释器入口                                      | 反优化时用                           |
| `declaring_class`                 | uint32 | 所属类（GCRoot）                                | 定位类、更新声明类                   |

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

1. **ArtMethod 类**（art*method.h:24）：只有方法，没有真实字段。所有字段都以 `static Member<ArtMethod, void\*> entry_point_from_compiled_code*;` 的形式声明（art_method.h:344）。

2. **Member 模板**（member.h:12）：它只存一个 `int32_t offset`，加两个操作：
   - `Get(instance)` → safeReadValue → `memcpy((char*)instance + offset, ...)` 读（member.h:31）
   - `Set(instance, value)` → memcpy((char\*)instance + offset, ...) 写（member.h:43）

   本质就是**"基址 + 偏移"的指针运算**，没有魔法。所谓"读取 entry*point_from_compiled_code*"，翻译成机器行为就是：**拿 ArtMethod 的地址 + 探测到的偏移，从那个内存位置读 8 个字节出来当指针用。**

### 四、偏移量怎么探测（InitMembers）

`art_method.cpp:125` 的 `InitMembers(m1, m2, m3, access_flags)` 负责在初始化阶段把这张表填出来。核心手法是**内存扫描**：

- **算结构体大小**：`size = |m2 - m1|`，用两个相邻 ArtMethod 的地址差得出每个方法占多少字节（art_method.cpp:139）。
- **找 access*flags***：用 Memory::FindOffset（memory.h:31）在 m1 的内存里从 0 开始逐字节（步长 2）扫，找哪个位置的值恰好等于已知的 access_flags → 那个位置就是偏移。
- **找 entry_point 系列**：更精妙——传进来的 m1 是一个 native 方法，它的 entry*point_from_jni* 指向一个已知函数 Ruler*m1。扫描找 Ruler_m1 这个指针值出现的位置，就定位到 entry_point_from_jni*，再 + sizeof(void\*) 顺藤摸瓜得到 entry*point_from_compiled_code*（art_method.cpp:198-247）。

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

| #   | 必须掌握的点                                                           | 判断标准                                |
| --- | ---------------------------------------------------------------------- | --------------------------------------- |
| 1   | ArtMethod 是"方法"在 ART 里的 native 实体，Java 的 Method 只是壳       | 能解释 getArtMethod 返回的 long 是什么  |
| 2   | entry*point_from_compiled_code* 是 hook 的命门，改它 = 劫持调用        | 能说出 REPLACEMENT 模式改的是哪个字段   |
| 3   | 本工程不硬编码 struct，用"Member 偏移表 + 动态扫描"适配多 Android 版本 | 能说清为什么不能写死 struct、偏移从哪来 |

这三点是后续所有课（entry_point 改写、trampoline、生成 dex）的**地基**。第三点尤其关键，它解释了为什么 art_method.cpp 里到处都是 SetOffset、GetOffset 而不是直接 .field 访问。

---

## 附：本课涉及的关键源码位置

| 文件                                                                  | 关键内容                                                         |
| --------------------------------------------------------------------- | ---------------------------------------------------------------- |
| `module-pierce/src/main/java/com/m4399/pierce/core/Pierce.java`       | Java 层 hook 主入口、HookRecord、genereateDexAndLoad、handleCall |
| `module-pierce/src/main/java/com/m4399/pierce/core/PierceNative.java` | 所有 native 方法声明                                             |
| `module-pierce/src/main/cpp/art/art_method.h`                         | ArtMethod 类、字段 Member 声明、默认偏移表                       |
| `module-pierce/src/main/cpp/art/art_method.cpp`                       | InitMembers 偏移探测、FromReflectedMethod、BackupFrom/AfterHook  |
| `module-pierce/src/main/cpp/utils/member.h`                           | Member 模板（基址+偏移读写）                                     |
| `module-pierce/src/main/cpp/utils/memory.h`                           | Memory::FindOffset / safeReadValue                               |
| `module-pierce/src/main/cpp/pine.cpp`                                 | Pine_getArtMethod 等 JNI 实现 + gMethods 注册表                  |

---

## 第三课：答疑澄清（针对 ArtMethod 的三个提问）

### 问1：本工程 hook 的核心是不是就是改 entry_point_from_compiled_code 入口地址为自定义函数，从而改变方法执行？

答：对 REPLACEMENT 模式而言是，但"整个工程"不止这一种手法。精确分三种：

| hook 手法               | 修改位置                                       | 触发条件                                 |
| ----------------------- | ---------------------------------------------- | ---------------------------------------- |
| REPLACEMENT（入口替换） | 改 entry*point_from_compiled_code* 指向 bridge | 默认模式（Android 8+）、INLINE 失败回退  |
| INLINE（内联跳板）      | 不改字段，覆盖已编译机器码前几条指令为跳转     | INLINE / INLINE_WITHOUT_JIT 且方法已编译 |
| JNI 入口替换            | 改 entry*point_from_jni*                       | 目标是 native 方法                       |

所以"改 entry*point_from_compiled_code* 为自定义函数地址"是 REPLACEMENT 模式的核心本质，也是 Android 8+ 的默认主路径；但 INLINE 模式改的是机器码本身，根本不动这个字段。

### 问2：为什么别的解释说是 entry*point_from_quick_compiled_code*？

答：是同一个字段，年代不同、名字不同：

- Android 5.0/5.1 时代 ART 用 quick 编译器，字段名是 entry*point_from_quick_compiled_code*。
- Android 6.0 起编译体系重构（quick/portable 区分被废弃），字段重命名为 entry*point_from_compiled_code*。

两者指同一件事：已编译机器码的入口。你看到的旧资料（Android 5.x hook 教程）用的是旧名。

工程内部证据：成员变量命名 entry*point_from_compiled_code*（art_method.h:344），但默认偏移函数保留旧名 GetDefaultEntryPointFromQuickCompiledCodeOffset（art_method.h:293）——作者跟着 AOSP 命名演化走的痕迹。

### 问3：你说拿偏移是通过"反射读 Executable 的 artMethod 字段"，最后又说"不硬编码 struct、用 Member 偏移表 + 动态扫描"，两者什么关系？

答：是递进的两层，不矛盾，之前讲混了，澄清如下：

**第一层：找到 ArtMethod 这个对象的地址（FromReflectedMethod，art_method.cpp:99）**

- Android R 之前：env->FromReflectedMethod(javaMethod) 返回的 jmethodID 就是 ArtMethod 指针，直接强转。
- Android R 及之后：反射读 Java 对象 Executable 的 artMethod 字段（GetLongField）。

这一步只解决一件事：从一个 Java Method 对象，找到它对应的 ArtMethod 结构体在内存的起始地址。

**第二层：找到 ArtMethod 内部某个字段的偏移（Member 偏移表 + 动态扫描）**

- 拿到 ArtMethod 起始地址后，因为不同 Android 版本内部字段排列不同，不写死 struct，而是动态扫描出每个字段相对起始地址的 offset，再用 Member 模板"基址 + offset"读写。

这一步解决的是：知道 ArtMethod 地址后，怎么安全读写它内部某个字段（比如 entry*point_from_compiled_code*）。

比喻：第一层 = 找到这栋楼的地址（ArtMethod 指针）；第二层 = 找到楼里某房间的门牌号（字段相对偏移）。先有楼地址，再有门牌号，才能进房间改东西。

### 问4：getArtMethod 具体怎么拿到 ArtMethod 地址？

关键澄清：getArtMethod 这一行没有"动态扫描"，它只是把地址从 Java 对象里"抠"出来。真正的动态扫描发生在 InitMembers（另一处）。完整链路 5 步：

| 步骤 | 位置                  | 动作                                                                                |
| ---- | --------------------- | ----------------------------------------------------------------------------------- |
| 1    | Pierce.java:373       | Java 调 PierceNative.getArtMethod(method)                                           |
| 2    | pine.cpp:596          | 注册表把 Java 名 getArtMethod 映射到 C 函数 Pine_getArtMethod                       |
| 3    | pine.cpp:371-374      | Pine_getArtMethod 调 ArtMethod::FromReflectedMethod，返回的指针强转成 jlong 回 Java |
| 4    | art_method.cpp:99-104 | FromReflectedMethod 分两条路                                                        |
| 5    | art_method.h:35-42    | R 及之后走 GetArtMethodForR 反射读字段                                              |

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

| 维度     | 归属                                                                                                              |
| -------- | ----------------------------------------------------------------------------------------------------------------- |
| 概念提出 | Pine（canyie/Pine）框架的架构设计，本工程是 Pine 二次开发，继承而来                                               |
| 落地形式 | Java 方法——HookRecord 里是 public Method bridge; public Method backup;（Pierce.java:1180-1181），是 Java 反射对象 |
| 生成方式 | genereateDexAndLoad 时，用 C++ 里的 slicer 库手写一个 dex，DexClassLoader 加载后得到这两个 Java 方法              |
| 底层实体 | 每个 Java 方法加载后都有对应的 ArtMethod，所以 bridge/backup 也各有一个 ArtMethod 结构体                          |

不是 ART 虚拟机自带的，也不是 C/C++ 语言特性——是 Pine 这套 hook 方案自己设计出来的两个"中间方法"，用来解决"原方法被改了入口后，还得有个地方保留原逻辑（backup）、有个地方先拦下来分发回调（bridge）"的问题。

同类框架也都有这个思路：YAHFA、SandHook、Whale 的 backup/trampoline 概念一脉相承。

---

## 第四课：hookNewMethod（内容讲解 + 答疑）

### 一、hookNewMethod 完整讲解（Pierce.java:395-463）

**为什么要有它**：前面的 `hook()` 只负责登记 HookRecord，真正"改入口"的动作全在 `hookNewMethod`。一个方法**第一次**被 hook 才走这里；之后重复 hook 同一个方法，`hook()` 里 `newMethod == false`，只 addCallback 不再改入口（见 handleHook，Pierce.java:46-66）。

完整源码：

```java
// Pierce.java:395
static void hookNewMethod(HookRecord hookRecord, MethodHook hook, int modifiers, boolean canInitDeclaringClass) {
    Member method = hookRecord.target;
    final int mode = hookMode;                                        // 398
    boolean isInlineHook = mode == HookMode.INLINE || mode == HookMode.INLINE_WITHOUT_JIT;

    long thread = PierceNative.currentArtThread0();                   // 400

    if ((hookRecord.isStatic = Modifier.isStatic(modifiers)) && canInitDeclaringClass) {  // 401
        resolve((Method) method);                                     // 402
        if (PierceConfig.sdkLevel >= Build.VERSION_CODES.Q) {
            PierceNative.makeClassesVisiblyInitialized(thread);       // 410
        }
    }

    Class<?> declaring = method.getDeclaringClass();                  // 414

    final boolean jni = Modifier.isNative(modifiers);                 // 417
    final boolean proxy = Proxy.isProxyClass(declaring);              // 418

    // Only try compile target method when trying inline hook.
    if (isInlineHook) {                                               // 421
        if (!(jni || proxy)) {                                        // 423
            if (mode == HookMode.INLINE) {
                boolean compiled = PierceNative.compile0(thread, method);  // 425
                if (!compiled) {
                    PierceLogUtils.alwaysLogW("hookNewMethod " + method + " Cannot compile the target method, force replacement mode.");
                    isInlineHook = false;                             // 428，编译失败回退
                }
            }
        } else {
            isInlineHook = false;                                     // 432，native/proxy 不能 inline
        }
    }

    if (method instanceof Method) {                                   // 437
        hookRecord.paramTypes = ((Method) method).getParameterTypes();
    } else {
        hookRecord.paramTypes = ((Constructor<?>) method).getParameterTypes();
    }
    hookRecord.paramNumber = hookRecord.paramTypes.length;            // 443

    genereateDexAndLoad(hookRecord, hook);                            // 445

    hookRecord.skipUpdateDeclaringClass = true;                       // 447

    Method backup = PierceNative.hook0(thread,                        // 449
            declaring, hookRecord, method,
            hookRecord.bridge, hookRecord.backup,
            isInlineHook, jni, proxy);

    if (backup == null) {                                             // 459
        throw new RuntimeException("Failed to hook method " + method);
    }
}
```

**逐段讲解（顺序即代码从上到下的执行顺序）：**

**第 1 步：签名与 hook 模式（395-398）**

```java
Member method = hookRecord.target;   // 目标方法（反射对象，可能是 Method 或 Constructor）
final int mode = hookMode;           // 全局 hook 模式，setHookMode 设的
boolean isInlineHook = mode == HookMode.INLINE || mode == HookMode.INLINE_WITHOUT_JIT;
```

作用：先定好"这次 hook 用内联还是入口替换"。`isInlineHook` 为 true 表示走内联（INLINE 或 INLINE*WITHOUT_JIT），false 表示走 REPLACEMENT（入口替换）。这个布尔值会一路传给最后的 `hook0`，决定 native 层是改 `entry_point_from_compiled_code*`（replacement）还是覆盖机器码（inline）。

**第 2 步：静态方法先触发类初始化（400-412）**

```java
long thread = PierceNative.currentArtThread0();   // 拿 art::Thread*（第四课答疑问2详解）
if ((hookRecord.isStatic = Modifier.isStatic(modifiers)) && canInitDeclaringClass) {
    resolve((Method) method);                     // 触发类初始化
    if (PierceConfig.sdkLevel >= Build.VERSION_CODES.Q) {
        PierceNative.makeClassesVisiblyInitialized(thread);  // 让类"可见初始化"
    }
}
```

为什么要有 `resolve()`：**ART 对静态方法的入口有个"类初始化陷阱"机制**。类还没初始化时，静态方法的 entry_point 指向的是 ART 的"类初始化桩"（resolver），第一次调用才真正初始化类并修好入口。如果这时候直接改入口，改到一半类初始化把入口重置了，hook 就失效。所以先 `resolve()` 触发一次（`resolve()` 内部是故意用错误参数 invoke 该方法，靠抛出的 `IllegalArgumentException` 这个副作用来强制完成类初始化——见 Pierce.java:678-694）。

Q+ 的 `makeClassesVisiblyInitialized` 是同一件事的加强版：Android R 引入"visibly initialized"新类状态，类初始化后会 `FixupStaticTrampolines` 重置入口，所以 hook 前先让类"可见初始化"，防止入口被重置。代码注释里写了这特性官方 Q 没有，但某些 ROM cherry-pick 了那个 commit。

**第 3 步：区分方法类型（414-418）**

```java
Class<?> declaring = method.getDeclaringClass();   // 声明这个方法的类
final boolean jni = Modifier.isNative(modifiers);  // 是不是 native 方法
final boolean proxy = Proxy.isProxyClass(declaring); // 是不是动态代理类
```

作用：`jni` 和 `proxy` 是两种"特殊方法"，它们**不能走内联 hook**（下面第 4 步会用到）。原因：

- native 方法没有 Java 字节码、没有可覆盖的已编译机器码，只能改 `entry_point_from_jni_`（JNI 入口替换）。
- 代理方法由 JVM 运行时动态生成，结构特殊，inline 覆盖机器码会出问题。

**第 4 步：inline 模式先编译（420-434）**

```java
if (isInlineHook) {
    if (!(jni || proxy)) {              // 非 native 非 proxy 才尝试编译
        if (mode == HookMode.INLINE) {
            boolean compiled = PierceNative.compile0(thread, method);  // JIT 编译
            if (!compiled) {
                isInlineHook = false;   // 编译失败 → 回退 replacement
            }
        }
    } else {
        isInlineHook = false;           // native/proxy 强制回退 replacement
    }
}
```

为什么要有 `compile0`：内联 hook 的原理是"覆盖方法已编译机器码的前几条指令为跳转"。但**方法可能还没被 JIT 编译**（还停在解释执行），就没有机器码可覆盖。`compile0` 就是主动触发一次 JIT 编译，把目标方法先编译成机器码。编译失败（比如某些方法 ART 拒绝编译），就 `isInlineHook = false` 回退到入口替换。

注意 `INLINE_WITHOUT_JIT` 和 `INLINE` 的区别：前者**不主动 compile0**（方法没编译就自然回退 replacement），后者主动编译。所以注释里写"Only try compile target method when trying inline hook"，且 `compile0` 只在 `mode == INLINE` 时调。

**第 5 步：记录参数类型和个数（436-443）**

```java
if (method instanceof Method) {
    hookRecord.paramTypes = ((Method) method).getParameterTypes();
} else {
    hookRecord.paramTypes = ((Constructor<?>) method).getParameterTypes();
}
hookRecord.paramNumber = hookRecord.paramTypes.length;
```

作用：记下参数的 Class 数组和个数。这是给后面 `handleCall` 用的——回调分发时，要把传入的参数装进 `Object[]`（bridge 里 `new Object[]{a, b}` 就是靠它知道要装几个、各是什么类型）。注释里有个 WARNING：这段代码会导致参数类型和返回类型被初始化（反射 getParameterTypes 会触发类加载）。

**第 6 步：生成 bridge/backup dex（445）**

```java
genereateDexAndLoad(hookRecord, hook);
```

作用：运行时生成并加载含 bridge/backup 的 dex，把两个 Method 存进 `hookRecord.bridge` / `hookRecord.backup`。这是**第五课**讲的内容。

**第 7 步：标记跳过声明类更新（447）**

```java
hookRecord.skipUpdateDeclaringClass = true;
```

作用：设一个标记，告诉后续流程不要再更新 `declaring_class` 字段。这是防递归/防重复处理的一个状态位（native 层 BackupFrom 之后会用到）。

**第 8 步：native 改入口（449-461）—— 全链路核心交接点**

```java
Method backup = PierceNative.hook0(thread,        // art::Thread*（第 2 步拿的）
        declaring,                                 // 声明类
        hookRecord,                                // 含 artMethod / bridge / backup 等
        method,                                    // 目标方法
        hookRecord.bridge,                         // 第 6 步生成的 bridge
        hookRecord.backup,                         // 第 6 步生成的 backup
        isInlineHook, jni, proxy);                 // 三个策略标志

if (backup == null) {
    throw new RuntimeException("Failed to hook method " + method);
}
```

**这是整条链路 Java → native 的交接点**。`hook0` 在 Java 侧只是声明（PierceNative.java），真正的 C++ 实现在 `pine.cpp` 的 `Pine_hook0`（pine.cpp:186-302）。它拿齐了"目标方法 + bridge + backup + 策略标志"，去 native 层执行真正的改入口动作，最后返回一个 backup Method（native 层可能对 backup 做二次处理后的版本）。

为什么入参里 `hookRecord.bridge` / `hookRecord.backup` 都齐了：因为第 6 步 `genereateDexAndLoad` 刚把它们造出来。**这也就是为什么顺序不能乱——先生成 dex，再拿生成出的两个 Method 去改入口。**

`backup == null` 判空是兜底：native 改入口失败会返回 null，Java 层立刻抛异常，不让半成品 hook 记录残留。

### 二、答疑

### 问1：HookMode 的 mode 有哪些？分别什么意思？

定义在 **Pierce.java:1136-1162**：

| 模式               | 值  | 原理                                              | 现状                      |
| ------------------ | --- | ------------------------------------------------- | ------------------------- |
| AUTO               | 0   | 按 Android 版本自动选                             | 默认值                    |
| INLINE             | 1   | 覆盖方法已编译机器码前几条指令为跳转              | 已废弃（手动 JIT 易崩溃） |
| REPLACEMENT        | 2   | 改 `entry_point_from_compiled_code_` 字段指向跳板 | Android 8+ 主路径         |
| INLINE_WITHOUT_JIT | 3   | 内联 hook，方法没编译就回退 REPLACEMENT           | Android 8 以下主路径      |

"REPLACEMENT = 改变 entry point 指针入口" 理解正确，但有个必须抠细的点：改 `entry_point_from_compiled_code_`，改成的**不是直接 bridge 的入口，而是 trampoline（跳板）的地址**。跳板是一段纯汇编，负责摆好寄存器/调用约定后再跳 bridge。中间多一层跳板是因为 ART 调用方法时有固定寄存器约定（this 在哪个寄存器、参数怎么排），直接指 bridge 会破坏约定。

AUTO 的决策在 `setHookMode`（Pierce.java:203-211）：

```java
newHookMode = sdkLevel < O ? INLINE_WITHOUT_JIT : REPLACEMENT;
```

原因（Pierce.java:204-207 注释）：Android N 及以下 `entry_point_from_compiled_code_` 可能被硬编码进机器码（sharpening），改字段无效，用 inline；O+ 不做此优化，用更稳的 REPLACEMENT。

### 问2：art::Thread\* 详解，和 ArtMethod 有关联吗？

**无字段级别关联**：ArtMethod 是"方法"结构体（一个方法一个），art::Thread 是"线程"结构体（一个线程一个）。两者是 ART 两个不同维度的核心对象。但在 hook 里紧密配合：Thread 是"执行者身份凭据"（挂起 VM、JIT 编译都要它），ArtMethod 是"被修改目标"。

**art::Thread 是什么**：每个 Java 线程在 native 层对应一个 art::Thread 结构体，保存线程状态标志（state_and_flags）、栈、JNI 本地引用表、异常等。它和 java.lang.Thread 的关系 = Method/ArtMethod 的关系（Java 对象是壳，art::Thread 是 native 真身）。

**怎么拿到当前线程**：`currentArtThread0`（pine.cpp:441-442）一行 `reinterpret_cast<jlong>(art::Thread::Current(env))`。真正逻辑在 `Thread::Current`（thread.h:31-59），4 种方式按优先级降级：

1. **直接调 ART 符号**（thread.h:33-34）：`current()` = `CurrentFromGdb()`，在 Thread::Init（thread.cpp:24-25）从 libart.so 按符号解析。
2. **Java 反射读 nativePeer 字段**（thread.h:35-48）：`Thread.currentThread()` 拿对象，再 GetLongField 读 nativePeer（ART 把 art::Thread\* 藏在这个字段）。
3. **读 TLS 槽位 7**（thread.h:49-50）：`__get_tls()[7]`（TLS_SLOT_ART_THREAD_SELF）。`__get_tls` 是内联汇编（thread.h:16-24）读 arm64 的 TLS 基址寄存器 `tpidr_el0`，再取第 7 槽。
4. **pthread key**（thread.h:51-52）：`pthread_getspecific(*key_self)`，最老版本兜底。

方式③的汇编是 OS 原理重点：`tpidr_el0` 是 arm64 的线程指针寄存器，CPU 用它指向当前线程 TLS 块；ART 把 art::Thread\* 存在该块固定槽位 7，一条 `mrs` 指令读完，最快路径。

### 问3：bridge / backup 举例说明

以 hook `Calculator.add(int a, int b) { return a + b; }` 为例。

**hook 前**：`calculator.add(1,2)` → 读 entry_point → 真实机器码 → 执行 a+b → 返回 3。

**hook 时**生成两个方法（签名和 add 一致）：

- **bridge（新入口）**：body 是 `return handleCall(artTargetMethod, this, new Object[]{a, b})`，负责"接住 + 分发"给 handleCall（handleCall 依次调 beforeCall → backup 执行原逻辑 → afterCall）。
- **backup（原方法替身）**：body = 原 add 的字节码 `return a + b;`，保留原逻辑。

**hook 后完整链路**：

```
calculator.add(1,2)
  → entry_point 改成 trampoline
  → trampoline 跳 bridge
  → bridge 调 handleCall(artTargetMethod, this, [1,2])
        ├─ beforeCall 回调（可改参数）
        ├─ backup.invoke(1,2) = 3   ← 原逻辑在这里执行
        └─ afterCall 回调（可改返回值）
  → 返回结果
```

**为什么必须拆两个**：如果直接把原入口改成你的回调，你的回调执行完想调"原逻辑"时，原入口已被覆盖成你自己，再调就是无限递归。所以 bridge（新入口，先接住）+ backup（原逻辑复制品，还能调回真正原方法）。

店类比：原方法=加法店，hook 后门牌（entry_point）换成传达室（bridge）；传达室先问改不改订单（beforeCall）→ 转给后厨（backup，原加法师傅）→ 包装交给顾客（afterCall）。原师傅没被赶走，只是搬进后厨。

---

## 第五课：genereateDexAndLoad（Pierce.java:465-586）

### 一、为什么要有它

hook 要改目标方法入口指向 bridge，但 bridge/backup 此刻还不存在。因为目标方法签名（参数类型/个数/返回值/是否 static/是否构造器）各不相同，bridge 必须签名完全一致才能接住调用，编译期无法预写，只能**运行时按真实签名动态生成 dex**。

类比：`Proxy.newProxyInstance` 按接口不同动态生成代理类字节码，genereateDexAndLoad 同理生成 dex，且一次生成 bridge + backup 两个方法。

作用：生成「bridge + backup + artTargetMethod 字段」的 dex → DexClassLoader 加载 → 反射取出 bridge/backup → 存进 hookRecord，供下一步 hook0（Pierce.java:449）使用。

### 二、逐段详解

**第 1 段：入参 + 目录校验（465-476）**

```java
Member targetMethod = hookRecord.target;      // 目标方法（反射对象）
long artTargetMethod = hookRecord.artMethod;  // 目标方法的 ArtMethod 地址（第二课 getArtMethod 拿的）

File dirFile = new File(dexDirPath);
if (!dirFile.exists()) {
    throw new RuntimeException("genereateDexAndLoad dexDirPath=" + dexDirPath + " doesn't exist");
}
if (!dirFile.canWrite()) {
    throw new RuntimeException("genereateDexAndLoad dexDirPath=" + dexDirPath + " can't write");
}
```

- `artTargetMethod` 这个 `long` 就是后面要写进生成类字段的 ArtMethod 地址，让 bridge 运行时能反查 HookRecord。
- 目录校验是前置防御：dex 要落盘才能被 DexClassLoader 加载，目录有问题早抛早发现（`dexDirPath` 在 `initSaveDexDirPath`，Pierce.java:121-127 里初始化）。

**第 2 段：算 shorty + 拼唯一命名（478-486）**

```java
String generateShorty = computeShorty(targetMethod);   // 短签名
String callbackClassName = hook.getClass().getName();  // 回调类名
String fileName = "pg_vc_" + BuildConfig.VERSION_CODE + "_" + callbackClassName + "_" + generateShorty + ".dex";
String generateClassName = callbackClassName + "_G_" + generateShorty;
String generateMethodName = (targetMethod instanceof Constructor) ? "constructor" : targetMethod.getName();
String generateBackupMethodName = generateMethodName + "Backup";
String generateFieldName = "artTargetMethod";
```

- `generateShorty`：用 `computeShorty` 算出的短签名（详见下文第三部分）。
- `generateMethodName`：bridge 方法名 = 原方法名；如果是构造器则统一叫 `constructor`。
- `generateBackupMethodName`：backup 方法名 = 原方法名 + `Backup`。
- `generateFieldName`：生成的类里存 ArtMethod 地址的静态字段名，固定叫 `artTargetMethod`。
- 为什么要拼唯一文件名/类名：同一进程可能 hook 多个方法、甚至同一个方法配不同 callback 类，文件名塞进 `VERSION_CODE + callbackClassName + shorty` 保证不互相覆盖。

**第 3 段：生成 dex（带缓存）（488-513）**

```java
File dexFile = new File(dirFile, fileName);
String dexFilePath = dexFile.getAbsolutePath();
if (dexFile.exists()) {
    PierceLogUtils.alwaysLogI("... already exist.");   // 已存在，直接复用
} else {
    long startGenerateDexMillis = System.currentTimeMillis();
    boolean generateRet = PierceNative.generateDex(dexFilePath,
            targetMethod, hookRecord.isStatic, handleCallMethod,
            generateClassName, generateMethodName, generateBackupMethodName,
            generateShorty, generateFieldName);
    if (!generateRet) {
        throw new RuntimeException("... can't generate.");
    }
    long generateDexDuration = System.currentTimeMillis() - startGenerateDexMillis;
    PierceLogUtils.alwaysLogI("... generateDuration=" + generateDexDuration + "ms");
}
```

- 为什么缓存：生成 dex 要 native 走 slicer 库逐条写字节码，耗时长（日志里专门记 `generateDuration`）。同一"回调类 + 签名"反复 hook 时，dex 已落盘就直接复用，省一次生成。
- `PierceNative.generateDex` 是 native 方法，Java 侧只是声明，真正的 C++ 实现在 pine.cpp:516-538 → `DexBuilderUtil::generateDexFile`（external/DexBuilder/DexBuilderUtil.cpp:12）——**这是后续 native 课的重点**，本课先记住：它按传入的 shorty、类名、方法名在磁盘产出合法 dex。
- 注意 `handleCallMethod` 这个入参：它在 `initGenerateDex`（Pierce.java:101-112）里初始化，从 `HandleCallWrapClass` 反射找到那个 `handleCall(Object, Object, Object[])` 方法（Pierce.java:1001），native 生成 bridge 字节码时要把对它的调用写进去。

**第 4 段：校验 odex 优化目录（515-524）**

```java
if (dexOptDirPath != dexDirPath) {
    File dirOptDirFile = new File(dexOptDirPath);
    if (!dirOptDirFile.exists()) {
        throw new RuntimeException("... dexOptDirPath ... doesn't exist");
    }
    if (!dirOptDirFile.canWrite()) {
        throw new RuntimeException("... dexOptDirPath ... can't write");
    }
}
```

DexClassLoader 加载 dex 时要一个"优化目录"放 odex 优化产物。Android 5.0/5.1 上这个目录和 dex 同目录会有冲突（CLAUDE.md 记过，`initSaveDexDirPath` 按 sdkLevel 分支处理），所以 `dexOptDirPath` 单独存、单独校验。

**第 5 段：加载 dex + 取类 + 写 artTargetMethod 字段（525-542）**

```java
ClassLoader parentClassLoader = Pierce.class.getClassLoader();
DexClassLoader dexClassLoader = new DexClassLoader(dexFilePath, dexOptDirPath, null, parentClassLoader);
Class<?> generatedClass = dexClassLoader.loadClass(generateClassName);

Field generatedField = generatedClass.getDeclaredField(generateFieldName);  // "artTargetMethod"
generatedField.setAccessible(true);
generatedField.set(null, artTargetMethod);   // 把 ArtMethod 地址写进生成类静态字段
```

- `DexClassLoader` 是 Android 标准 API，把磁盘上的 dex 当类加载进来——没魔法，跟普通动态加载插件/热修复一样。
- 关键动作是 `generatedField.set(null, artTargetMethod)`：把目标方法 ArtMethod 地址（那个 `long`）写进生成类的静态字段 `artTargetMethod`。为什么？

  因为 bridge 被调用时要反查 HookRecord（详见下文第四部分）。

**第 6 段：反射捞出 bridge 和 backup 两个 Method（544-566）**

```java
Method bridgeMethod = null;
Method backupMethod = null;
for (Method method : generatedClass.getDeclaredMethods()) {
    if (generateMethodName.equals(method.getName())) {       // 原方法名 → bridge
        bridgeMethod = method;
        bridgeMethod.setAccessible(true);
        continue;
    }
    if (generateBackupMethodName.equals(method.getName())) { // 原名+Backup → backup
        backupMethod = method;
        backupMethod.setAccessible(true);
        continue;
    }
}
if (null == bridgeMethod) {
    throw new RuntimeException("... failed to find bridgeMethod ...");
}
if (null == backupMethod) {
    throw new RuntimeException("... failed to find backupMethod ...");
}
```

生成类里有两个方法：一个叫原方法名（bridge），一个叫原方法名+`Backup`（backup）。这里按名字把它们反射捞出来，找不到就抛异常（说明 dex 生成有问题）。

**第 7 段：Q+ 再做一次可见初始化（569-578）**

```java
if (PierceConfig.sdkLevel >= Build.VERSION_CODES.Q) {
    long thread = PierceNative.currentArtThread0();
    PierceNative.makeClassesVisiblyInitialized(thread);
}
```

和 `hookNewMethod` 第 2 步（401-412）是同一件事：Android R 有"visibly initialized"新类状态，类初始化后会 reset 入口。这里对**生成的类**再补一次，防止生成的 bridge 入口被 reset。注释写了这特性官方 Q 没有，是某些 ROM cherry-pick 的。

**第 8 段：存进 hookRecord（580-581）**

```java
hookRecord.bridge = bridgeMethod;
hookRecord.backup = backupMethod;
```

到这里 `hookRecord` 凑齐三个关键字段：`artMethod`（目标方法地址）、`bridge`（新入口）、`backup`（原逻辑替身）。下一步 `hook0`（Pierce.java:449）就是用这三个东西去改入口——这就是为什么必须先讲完 `genereateDexAndLoad`，`hook0` 的入参 `hookRecord.bridge` / `hookRecord.backup` 才有着落。

### 三、shorty（重点，非纯 Java 概念）

dex 格式的「短签名」，单字母描述返回值+每个参数类型。computeShorty（601-614）+ toShorty（588-599）算出：

| 字母 | 类型    |     | 字母 | 类型         |
| ---- | ------- | --- | ---- | ------------ |
| V    | void    |     | J    | long         |
| Z    | boolean |     | F    | float        |
| B    | byte    |     | D    | double       |
| C    | char    |     | L    | 所有引用类型 |
| S    | short   |     |      |              |
| I    | int     |     |      |              |

举例（类比 JNI 签名精简版）：

- `int add(int a, int b)` → `III`
- `void log(String tag, String msg)` → `VLL`
- `String getName()` → `LL`

JNI 签名 `(Ljava/lang/String;I)V` 是冗长形式，shorty 是精简版：native 生成 dex 只需知道「返回啥、几个参数、大类」，引用类型统一 L。

### 四、artTargetMethod 字段的作用（关键）

`generatedField.set(null, artTargetMethod)`（542 行）把目标方法 ArtMethod 地址写进生成类静态字段。因为 bridge 被调用时要反查 HookRecord：

```java
// HandleCallWrapClass.handleCall（1001-1006）
HookRecord hookRecord = sHookRecords.get((long) artTargetMethod);
```

类比：给每个外卖骑手（bridge）配订单号（ArtMethod 地址），到调度中心（handleCall）凭订单号查系统（sHookRecords）找订单（HookRecord）。

### 五、记忆点

1. 存在原因：bridge/backup 签名必须匹配目标方法，无法编译期写死，只能运行时动态生成 dex（类比 Proxy.newProxyInstance）。
2. shorty = dex 紧凑签名，单字母描述返回值+参数，引用统一 L。
3. 生成类含 3 样：bridge（原方法名）、backup（原名+Backup）、artTargetMethod 静态字段。
4. 产物落进 hookRecord.bridge/backup，是下一步 hook0 改入口的输入。

### 六、核心理解（躯体 / 灵魂 比喻）

genereateDexAndLoad 不是"只为创建 bridge/backup"，完整职责是"准备材料"：生成（bridge + backup + artTargetMethod 字段）+ 加载（让它们活起来）+ 注入（写 ArtMethod 地址）+ 找回（反射取回）。

| 比喻       | 实际                                                        |
| ---------- | ----------------------------------------------------------- |
| 躯体       | 磁盘上的 `.dex` 文件（innerGenerateDexFile 写出的字节流）   |
| 灵魂       | 加载后每个方法拥有的 `ArtMethod` 结构体（真正的"方法实体"） |
| 完整的人   | bridge/backup 成为能独立存在、可被 ART 调用的方法           |
| 交给 hook0 | 把这两个"活方法"的 ArtMethod 指针交出去改入口               |

顺序：`innerGenerateDexFile`（躯体）→ `DexClassLoader` + `loadClass`（ART 解析 dex，分配 ArtMethod，赋予灵魂）→ 反射取回 → 交给 hook0。

### 七、暂略方法清单（后续回归）

以下 native/底层方法在讲解时暂时略过，全部讲完主线后回归：

| 方法                            | Java 位置         | native 实现                          | 评估               | 原因                                                           |
| ------------------------------- | ----------------- | ------------------------------------ | ------------------ | -------------------------------------------------------------- |
| `generateDex`                   | PierceNative.java | pine.cpp:516 → DexBuilderUtil.cpp:12 | **必须详细讲**     | 运行时生成 dex 是 Pierce 核心，涉及 dex 字节码格式 + slicer 库 |
| `compile0`                      | PierceNative.java | pine.cpp（Pine_compile0）            | **需要讲**         | 与 inline hook 直接相关，涉及 ART JIT 机制                     |
| `hook0`                         | PierceNative.java | pine.cpp:186（Pine_hook0）           | **已讲（第六课）** | 改入口的核心                                                   |
| `handleCall`                    | Pierce.java:1009  | 无（纯 Java）                        | **必须详细讲**     | 回调分发核心（before→原方法→after）                            |
| `resolve()`                     | Pierce.java:678   | 无（纯 Java）                        | 简要讲             | Java 反射技巧，逻辑简单（故意错误参数触发类初始化）            |
| `makeClassesVisiblyInitialized` | PierceNative.java | pine.cpp（native）                   | 简要讲             | 涉及 ART 内部类状态，工程面窄，知道作用即可                    |

---

## 第六课：hook0 的 native 链路（Pine_hook0，pine.cpp:186-302）

### 一、定位

| 层           | 位置                                                                   | 说明                                 |
| ------------ | ---------------------------------------------------------------------- | ------------------------------------ |
| Java 调用点  | Pierce.java:449                                                        | `PierceNative.hook0(...)`            |
| Java 声明    | PierceNative.java                                                      | `native Method hook0(...)`，只有签名 |
| C++ 实现     | **pine.cpp:186-302**                                                   | `Pine_hook0`，本课主角               |
| 关键调用实现 | trampoline_installer.cpp:168 / art_method.cpp:346 / art_method.cpp:448 | 装跳板 / BackupFrom / AfterHook      |

### 二、为什么要有 hook0（先答"为什么"）

Java 层到第五课为止，已备齐三样材料：`target`（要 hook 的方法）、`bridge`（新入口）、`backup`（原逻辑替身）。但"改入口"Java 干不了，三个原因：

1. **要裸改内存**：改 `entry_point_from_compiled_code_` 是往某个地址写 8 字节指针，inline 还要覆盖机器码——都是裸内存操作。
2. **要 stop-the-world**：ArtMethod 被几十个线程同时依赖，改它时其他线程不能正在读，否则读到改一半的状态崩溃。
3. **要调 ART 内部符号**：`SuspendVM`、`ArtMethod::CopyFrom` 这些只在 libart.so 里，只能 C++ 调。

### 三、完整源码（186-302）

```cpp
jobject Pine_hook0(JNIEnv *env, jclass,
                   jlong threadAddress, jclass declaring,
                   jobject hookRecord, jobject javaTarget,
                   jobject javaBridge, jobject javaBackup,
                   jboolean isInlineHook, jboolean isJni, jboolean isProxy) {

    // ① 记录目标方法描述（线程局部，供日志用）
    std::string targetMethodDesc = JNIHelper::methodToString(env, javaTarget);
    ThreadLocalVar::setTargetMethodDesc(targetMethodDesc);

    // ② 取 HookRecord 的 trampoline 字段 ID
    jfieldID HookRecord_trampoline = GetHookRecordTrampolineField(env, hookRecord);

    // ③ Java 对象 → native ArtMethod 指针
    auto thread  = reinterpret_cast<art::Thread*>(threadAddress);
    auto target  = art::ArtMethod::FromReflectedMethod(env, javaTarget);
    auto bridge  = art::ArtMethod::FromReflectedMethod(env, javaBridge);
    auto backup  = art::ArtMethod::FromReflectedMethod(env, javaBackup);

    // ④ 可选：先 JIT 编译 bridge（追求性能）
    if (PineConfig::jit_compilation_allowed && PineConfig::auto_compile_bridge) {
        bridge->Compile(thread);
    }

    bool is_inline_hook = JBOOL_TRUE(isInlineHook);
    const bool is_native = JBOOL_TRUE(isJni);
    const bool is_proxy  = JBOOL_TRUE(isProxy);
    // 注：当前分支已移除 is_native_or_proxy 合并变量，AfterHook/BackupFrom 直接传 is_native、is_proxy 两个分开的参数

    TrampolineInstaller *trampoline_installer = TrampolineInstaller::GetDefault();

    // ⑤ inline 降级：只支持 replacement / 目标没编译 → 回退 replacement
    if (is_inline_hook && (trampoline_installer->IsReplacementOnly() || !target->IsCompiled())) {
        is_inline_hook = false;
    }
    // ⑥ inline 安全性：不能安全内联 → 回退 replacement
    if (UNLIKELY(is_inline_hook && trampoline_installer->CannotSafeInlineHook(target))) {
        is_inline_hook = false;
    }

    bool skip_first_few_bytes = PineConfig::anti_checks
                                && is_inline_hook && trampoline_installer->CanSkipFirstFewBytes(target);

    void *new_entrypoint;
    char error_msg[512] = {0};
    {
        // ⑦ ★ 挂起 VM（stop-the-world）
        ScopedSuspendVM suspend_vm(thread);

        // ⑧ ★ 装跳板，返回原入口 call_origin
        void *call_origin = is_inline_hook
                ? trampoline_installer->InstallInlineTrampoline(target, bridge, skip_first_few_bytes)
                : trampoline_installer->InstallReplacementTrampoline(target, bridge);

        if (LIKELY(call_origin)) {
            // ⑨ ★ backup 复制 target 状态，成为"原方法替身"
            backup->BackupFrom(target, call_origin, is_inline_hook, is_native, is_proxy);
            // ⑩ ★ 改 target 的 access_flags，防 JIT/内联干扰
            target->AfterHook(is_inline_hook, is_native, is_proxy);
            new_entrypoint = target->GetEntryPointFromCompiledCode();
        } else {
            // 失败：拼错误信息
            snprintf(error_msg, sizeof(error_msg), "%s Failed to install %s trampoline ...");
            new_entrypoint = nullptr;
        }
    }  // ← 出作用域，ScopedSuspendVM 析构，恢复 VM

    if (LIKELY(new_entrypoint)) {
        // ⑪ 新入口写回 HookRecord.trampoline 字段
        env->SetLongField(hookRecord, HookRecord_trampoline, reinterpret_cast<jlong>(new_entrypoint));
        return javaBackup;
    } else {
        JNIHelper::Throw(env, ..., error_msg);
        return nullptr;
    }
}
```

### 四、逐步讲解

#### 步骤 ③：Java 对象 → native 指针（201-204）

```cpp
auto target = art::ArtMethod::FromReflectedMethod(env, javaTarget);
```

第二课讲的 `FromReflectedMethod`（art_method.cpp:99）：把 Java Method 换成 ArtMethod 指针。现在 target/bridge/backup 三个 Java 方法各自拿到 native 结构体地址，后面全围着这三个指针转。

#### 步骤 ⑦：ScopedSuspendVM 挂起 VM（257）【OS 原理重点】

```cpp
ScopedSuspendVM suspend_vm(thread);
```

**RAII**（C++ 靠构造/析构自动管资源）。定义在 android.h:163-175：

```cpp
class ScopedSuspendVM {
public:
    ScopedSuspendVM(void* self) { Android::SuspendVM(this, self, "pierce hook method"); }  // 构造：挂起
    ~ScopedSuspendVM() { Android::ResumeVM(this); }                                        // 析构：恢复
};
```

**作用：构造时挂起所有其他 Java 线程（stop-the-world），析构时恢复。** 中间那个 `{}` 块就是"手术区"。

为什么必须挂起：ArtMethod 是共享结构，几十个线程可能同时在读它。你改 entry_point 到一半，别的线程读到"改了一半"的指针（只写了 4 字节），跳到非法地址崩溃。

类比：给整栋楼配电箱换闸，先整楼断电。挂起 VM = 断电，改入口 = 换闸，恢复 = 重新供电。和 Android GC 的 stop-the-world 同一套机制。

#### 步骤 ⑧：装跳板（259-261）【REPLACEMENT 核心】

```cpp
void *call_origin = is_inline_hook
        ? InstallInlineTrampoline(target, bridge, skip_first_few_bytes)
        : InstallReplacementTrampoline(target, bridge);
```

主路径 `InstallReplacementTrampoline`（trampoline_installer.cpp:165-188）：

```cpp
void *TrampolineInstaller::InstallReplacementTrampoline(art::ArtMethod *target, art::ArtMethod *bridge) {
    void *target_origin_code_entry = target->GetEntryPointFromCompiledCode();  // ① 记下原入口
    void *method_jump_trampoline = CreateMethodJumpTrampoline(bridge);          // ② 造跳板
    if (!method_jump_trampoline) return nullptr;
    target->SetEntryPointFromCompiledCode(method_jump_trampoline);              // ③ 改入口为跳板
    return target_origin_code_entry;                                            // ④ 返回原入口
}
```

- **① 记下原入口** `target_origin_code_entry`：原方法真实机器码地址，hook 后不能丢，backup 要用。
- **② 造跳板** `CreateMethodJumpTrampoline(bridge)`：生成一段纯汇编跳板（详见本课"汇编在哪"）。
- **③ 改入口** `SetEntryPointFromCompiledCode(method_jump_trampoline)`：**整个 REPLACEMENT 模式的命门**，把 `entry_point_from_compiled_code_` 从"原机器码地址"改成"跳板地址"。本质是 `memcpy(instance+offset, &跳板地址, 8)`。
- **④ 返回原入口**：交给 backup 当"原逻辑入口"。

**为什么中间隔一层跳板，不直接指 bridge？** ART 调用方法有固定寄存器约定（this/参数在哪个寄存器、栈怎么摆），bridge 按"正常 Java 方法"编译，期望的约定和 ART 此刻状态对不上。跳板是手写汇编，专门做"ART 当前状态 → bridge 期望状态"的过渡。类比插头转换器。

#### 步骤 ⑨：BackupFrom —— backup 成为"原方法替身"（270）【三者串起来的关键】

```cpp
backup->BackupFrom(target, call_origin, is_inline_hook, is_native, is_proxy);
```

实现在 art_method.cpp:346-446。**作用：把 target 完整状态复制到 backup，让 backup 能独立执行原逻辑。** 关键：

```cpp
void ArtMethod::BackupFrom(ArtMethod* source, void* entry, ...) {
    // ① 把 source(=target) 整个结构体复制到 this(=backup)
    if (LIKELY(copy_from)) {
        copy_from(this, source, sizeof(void*));   // 优先 ART 的 CopyFrom 符号
    } else {
        memcpy(this, source, size);               // 兜底：直接内存拷贝
    }

    // ② 调整 access_flags
    uint32_t access_flags = source->GetAccessFlags();
    if (Android::version >= Android::kN) {
        if (Android::version >= Android::kR) access_flags &= ~kAccPreCompiled;
        access_flags |= kAccCompileDontBother;
    }
    if ((access_flags & AccessFlags::kStatic) == 0) {
        access_flags &= ~(AccessFlags::kPublic | AccessFlags::kProtected);
        access_flags |= AccessFlags::kPrivate;    // 非 static 改 private，当 direct method 调
    }
    access_flags &= ~AccessFlags::kConstructor;
    SetAccessFlags(access_flags);

    // ③ 处理 JIT 信息（防 GC 回收崩溃）—— 详见第十课第四节"JIT 信息迁移"

    // ④ 设 backup 自己的入口
    SetEntryPointFromCompiledCode(entry);   // entry = call_origin（原方法入口）
}
```

**核心是①和④：**

- **① `memcpy(this, source, size)`**：把 target 整个 ArtMethod（含方法体字节码位置 `data_`、`declaring_class`）复制给 backup，backup 就"长成"原方法的样子。
- **④ `SetEntryPointFromCompiledCode(entry)`**：把 backup 入口设成 `call_origin`（原方法真实机器码地址）。

这样"原逻辑"就保住了：hook 后 target 入口被改成跳板，但原机器码还在内存（地址=call_origin），backup 入口指向它，**调 backup = 执行原逻辑**。

类比：target 是一辆车，原厂发动机（原机器码）在 call_origin。hook 后 target 方向盘（入口）接到"导航仪"（跳板→bridge），先导航再决定走哪。backup 是"同款车壳 + 原厂发动机"，直接开 backup 就是开原车。

**② 为什么改 access_flags**：非 static 改 private、去掉构造器标志，让 backup 能作为"普通直接方法"被 ART 调用。`kAccCompileDontBother` 告诉 JIT"别优化 backup"，避免被 JIT 改动后和原方法行为不一致。

#### 步骤 ⑩：AfterHook —— 防止 hook 被 ART 干扰（271）

```cpp
target->AfterHook(is_inline_hook, is_native, is_proxy);
```

实现在 art_method.cpp:448-519。**作用：改 target 的 access_flags，防止 JIT/内联/解释器把 hook 绕过或还原。** 关键：

```cpp
void ArtMethod::AfterHook(bool is_inline_hook, bool is_native, bool is_proxy) {
    uint32_t access_flags = GetAccessFlags();

    // ≥N：禁止被 JIT 再编译
    if (Android::version >= Android::kN) {
        if (Android::version >= Android::kR) access_flags &= ~kAccPreCompiled;
        access_flags |= kAccCompileDontBother;
    }
    // ≥O 且非 inline：debug 模式强制解释器会忽略 entry_point，设 kNative 避免
    if (Android::version >= Android::kO && !is_inline_hook) {
        if (PineConfig::debuggable && !(is_native || is_proxy)) {
            access_flags |= AccessFlags::kNative;
        }
    }
    // ≥Q：清除 fast interpreter 缓存标志，刷新状态
    if (Android::version >= Android::kQ) {
        access_flags &= ~AccessFlags::kFastInterpreterToInterpreterInvoke;
    }
    // FastNative/CriticalNative 执行时 GC 被禁用，可能死锁，hook 方法不能带这两个标志（当前分支新增）
    bool hasNativeFlags = (access_flags & AccessFlags::kNative) != 0;
    if (hasNativeFlags) {
        access_flags &= ~AccessFlags::kFastNative;
        if (Android::version >= Android::kP) access_flags &= ~AccessFlags::kCriticalNative;
    }
    SetAccessFlags(access_flags);
    // 设解释器入口为 interpreter→compiled bridge
    if (art_interpreter_to_compiled_code_bridge)
        SetEntryPointFromInterpreter(art_interpreter_to_compiled_code_bridge);
}
```

为什么防 JIT：hook 后 target 入口被我们改了，但 JIT 不知道，过一会儿可能"好心"重编译 target、把入口又改回它认为对的值——hook 就被覆盖了。`kAccCompileDontBother`（"别费劲编译我"）就是给 JIT 打的"别动我"标记。

类比：root 后改了系统文件，系统"更新服务"过段时间会还原，你得打标记"这个文件跳过更新"。

#### 步骤 ⑪：写回 trampoline 字段 + 返回（287-297）

```cpp
env->SetLongField(hookRecord, HookRecord_trampoline, reinterpret_cast<jlong>(new_entrypoint));
return javaBackup;
```

把新入口存进 HookRecord 的 `trampoline` 字段（后面 invokeOriginalMethod 会用到），返回 `javaBackup`。

### 五、三者如何串起来（闭环图）

```
hook 前：
  target.entry_point ──→ 原机器码（执行原逻辑）

hook 后：
  target.entry_point ──→ 跳板 ──→ bridge ──→ handleCall（分发回调）
                                              └──→ invokeOriginalMethod ──→ backup.entry_point
                                                                              └──→ 原机器码（call_origin）

  backup.entry_point ──→ 原机器码（call_origin）   ← BackupFrom 里 SetEntryPointFromCompiledCode(entry) 设的
```

三句话记住分工：

1. **target**：入口被改，成为"被 hook 的方法"，所有调用被截走。
2. **bridge**：新入口（经跳板），接住调用、转 handleCall 分发回调。
3. **backup**：BackupFrom 复制了 target 状态 + 入口指向原机器码，成为"原方法替身"。

### 六、答疑

#### 问1：用 hook add 方法说明 target/bridge/backup 三者的作用（内存视角）

以 hook `Calculator.add(int a, int b) { return a + b; }` 为例。

**关键认知**：每个方法 = 一个 ArtMethod 结构体，里面有个 `entry_point_from_compiled_code_` 字段，指向"被调用时跳去执行的机器码地址"。方法跑起来本质就是"读 entry_point → 跳到那个地址执行"。

**hook 前**（内存里只有 target）：

```
[target: add]
   entry_point ──→ 机器码A（内容 = "a + b 然后返回"）
```

**hook 手术三步（改的就是 entry_point 指向）**：

```
手术①：改 target 入口 → 指向跳板 → 跳板拐到 bridge
手术②：BackupFrom → 让 backup 的 entry_point 指向机器码A（原逻辑没删，只是换 backup 来调）
手术③：AfterHook → 给 target 打"别让 JIT 动我"标记
```

**手术完成后完整状态**：

| 角色   | 是什么           | entry_point 指向  | 谁调它                    |
| ------ | ---------------- | ----------------- | ------------------------- |
| target | 你 hook 的原方法 | 跳板（被改掉）    | 业务代码照常调 add        |
| bridge | 新入口（拦截）   | 自己的 body       | 跳板拐进来的              |
| backup | 原方法替身       | 机器码A（原逻辑） | bridge 的 handleCall 里调 |

**真实调用走查 `calculator.add(1,2)`**：

```
1. calculator.add(1,2)
2. 读 target.entry_point → 是"跳板"（不是原逻辑了！）
3. 跳板 → 拐进 bridge
4. bridge body 执行：handleCall(artTargetMethod, this, [1, 2])
5. handleCall 里：
     a. 查 HookRecord（用 artTargetMethod 反查）→ 找到你的 MethodHook
     b. 调 beforeCall（可看/改参数 a=1, b=2）
     c. 调 invokeOriginalMethod → 实际调 backup
              ↓
        backup.entry_point → 机器码A → 执行 a+b → 返回 3  ← ★ 原逻辑在这里跑
     d. 调 afterCall（可改返回值）
6. handleCall 返回最终结果
7. calculator.add(1,2) 拿到结果
```

**为什么必须三个方法**：bridge 需要两件矛盾的事——先"拦下来"（插 before/after）+ 又得"调回原逻辑"。如果原逻辑还在 target 身上，bridge 一调 target 就又被拦截，无限循环。所以必须把原逻辑挪到第三个方法（backup）身上，bridge 调 backup 才不会被拦。

一句话：**target 门牌被换掉；bridge 新门房，先检查回调再叫原逻辑；backup 原逻辑本人，被 bridge 叫来干真正的活。**

#### 问2：replaceCall 和 beforeCall/afterCall 一样吗？

**机制同源，语义不同。** 底层都靠同一个开关 `returnEarly`。

CallFrame（Pierce.java:1300-1304）：

```java
public void setResult(Object result) {
    this.result = result;
    this.throwable = null;
    this.returnEarly = true;   // ← 置位"提前返回"
}
```

handleCall（Pierce.java:1047）靠这个开关决定是否调原方法：

```java
if (!callFrame.returnEarly) {
    callFrame.setResult(callFrame.invokeOriginalMethod());   // 调 backup
}
```

谁在 beforeCall 里调了 `setResult`，谁就阻止原方法（backup）执行。

**区别**：

| 维度       | MethodHook.beforeCall          | MethodReplacement.replaceCall                      |
| ---------- | ------------------------------ | -------------------------------------------------- |
| 原方法执行 | 默认执行（除非手动 setResult） | 必然不执行                                         |
| 定位       | 织入（前后插逻辑）             | 整体替换（顶掉原方法）                             |
| 实现       | 你自己写                       | final 的 beforeCall 内部调 replaceCall + setResult |
| 返回值作用 | 你设了才生效                   | 返回值就是方法最终返回值                           |
| 用途       | 记录参数、改参数、统计         | 完全改掉方法行为（DO_NOTHING、returnConstant）     |

MethodReplacement（MethodReplacement.java:36-45）故意把 beforeCall 设为 final、afterCall 空实现，逼你只用 replaceCall：

```java
@Override public final void beforeCall(CallFrame callFrame) {
    try {
        callFrame.setResult(replaceCall(callFrame));   // 返回值直接 setResult → returnEarly=true
    } catch (Throwable e) {
        callFrame.setThrowable(e);
    }
}
@Override public final void afterCall(CallFrame callFrame) {
}
```

一句话：**beforeCall 默认放行原方法、replaceCall 默认拦截原方法**——一个是"织入"，一个是"替换"。

#### 问3：entry_point 的 offset 适配为什么没在 hook0 里体现？

**因为适配发生在"初始化阶段"，hook0 运行时已经是适配好的结果。** 层层下钻：

```cpp
// 第 1 层：hook0 里（trampoline_installer.cpp:181）
target->SetEntryPointFromCompiledCode(method_jump_trampoline);

// 第 2 层：ArtMethod 方法（art_method.h:150-157）
void SetEntryPointFromCompiledCode(void* entry) {
    entry_point_from_compiled_code_.Set(this, entry);   // 调 Member::Set
}

// 第 3 层：Member::Set（member.h:43）
void Set(instance, value) {
    memcpy((char*)instance + offset, value, 8);   // offset 是适配好的
}
```

关键在第 3 层的 `offset`——它不是写死的，是在 **`InitMembers`（art_method.cpp:125，第二课讲的动态扫描）探测出来填进 Member 对象的**。hook0 执行时 offset 已经是"这个 Android 版本的正确偏移"。

时间线：

```
App 启动 → ensureInitialized → init0 → InitMembers（★ 这里适配，动态扫描填 offset）
        ↓ 之后 offset 固定
调 hook → hookNewMethod → hook0（★ 这里直接用填好的 offset，不再适配）
```

所以适配代码在 `InitMembers`，不在 hook0。

#### 问4：目前涉及汇编吗？

涉及，藏在"造跳板"那一步。`CreateMethodJumpTrampoline`（trampoline_installer.cpp:95-115）就是汇编跳板入口。真正的汇编（arm64.S:46-55）：

```asm
FUNCTION(pine_method_jump_trampoline)
    LDVAR(x0,  pine_method_jump_trampoline_dest_method)   // ① bridge 的 ArtMethod* 加载到 x0
    LDVAR(x17, pine_method_jump_trampoline_dest_entry)    // ② bridge 入口地址加载到 x17
    br  x17                                               // ③ 无条件跳转到 bridge 入口
```

整个 REPLACEMENT 跳板就这 3 条指令：

| 指令                    | 作用                            | 为什么                                        |
| ----------------------- | ------------------------------- | --------------------------------------------- |
| `ldr x0, [dest_method]` | bridge 的 ArtMethod 指针放进 x0 | ARM64 调用约定：x0 必须是被调方法的 ArtMethod |
| `ldr x17, [dest_entry]` | bridge 入口地址放进 x17         | x17 是临时寄存器，暂存跳转目标                |
| `br x17`                | 跳转到 bridge 入口              | 真正"拐进去"                                  |

`CreateMethodJumpTrampoline` 干的事：把汇编模板拷贝到可执行内存，再填两个数据槽（bridge 指针 + bridge 入口），最后 `Memory::FlushCache` 刷新指令缓存：

```cpp
void* TrampolineInstaller::CreateMethodJumpTrampoline(art::ArtMethod* dest) {
    void* mem = Memory::AllocUnprotected(kMethodJumpTrampolineSize);  // 分配可执行内存
    memcpy(mem, kMethodJumpTrampoline, kMethodJumpTrampolineSize);    // 拷贝汇编模板
    *dest_method_out = dest;                                          // 填①
    *origin_entry_out = dest->GetEntryPointFromCompiledCode();        // 填②
    Memory::FlushCache(mem, ...);                                     // 刷新指令缓存
    return mem;
}
```

#### 问5：hook0 是完整实现吗？

**hook0 是"编排者"，不是"演奏者"。** 它 11 步、200 行，是完整 hook 逻辑的执行入口，但每一句调用背后都藏着没展开的实现：

| hook0 里的调用                  | 背后藏着的"重活"                         | 藏在哪个文件                    |
| ------------------------------- | ---------------------------------------- | ------------------------------- |
| `FromReflectedMethod`           | offset 动态扫描 + R 版本反射适配         | art_method.cpp:99 / InitMembers |
| `SetEntryPointFromCompiledCode` | Member 的 offset 适配                    | art_method.h:150 → member.h     |
| `CreateMethodJumpTrampoline`    | 汇编跳板模板                             | arm64.S:46-55                   |
| `ScopedSuspendVM`               | suspend/resume 符号解析 + stop-the-world | android.h / android.cpp         |
| `BackupFrom`                    | CopyFrom 符号 + memcpy 整个结构          | art_method.cpp:250              |
| `AfterHook`                     | 一堆 access_flags 版本适配               | art_method.cpp:310              |
| `Memory::AllocUnprotected`      | 分配可执行内存（mmap 权限）              | memory.cpp                      |
| `Memory::FlushCache`            | 刷新 CPU 指令缓存（OS 原理）             | memory.cpp                      |

所以"很多东西没体现"的感觉是对的：hook0 这 200 行是"调度清单"，真正工程量散落在 art_method.cpp、trampoline_installer.cpp、arm64.S、android.cpp 里，加起来上千行。

**两句话收口**：hook0 是"完整 hook 逻辑的骨架"，主顺序已吃透；骨架里每根骨头（调用的函数）还包着实现细节，是后续往下钻的内容。

---

## 知识地图：汇编 / OS 原理在哪

之前感觉"没看到汇编/OS"，其实一直在，分布在三处：

| 目录                                                  | 对应知识                                                              |
| ----------------------------------------------------- | --------------------------------------------------------------------- |
| `trampoline/arch/*.S`（arm64.S / arm32.S / thumb2.S） | 手写汇编跳板（机器码肉身）                                            |
| `utils/memory.cpp` / `elf_image.cpp` / `memutil.h`    | OS 原理：mprotect 改页权限、ELF 解析、内存对齐、FlushCache 刷指令缓存 |
| `art/`                                                | ART 内部：改哪个字段、挂起哪个 VM、TLS 拿线程                         |
| `utils/scoped_memory_access_protection.*`             | 临时改代码页只读→可写的 RAII                                          |

三者关系一句话：汇编（.S）= 跳板机器码肉身；OS 原理（memory/elf）= 让跳板能写进内存、能被找到、能被 CPU 执行；ART 内部（art/）= 告诉改哪个字段、挂起哪个 VM。

---

## 第七课：从 hook() 逐行深入 —— 初始化与偏移扫描（C 核心）

> 本课开始，不再只讲骨架，而是从 `hook()` 方法（Pierce.java:342）逐行深入，遇到 C/汇编/OS 知识停下来单独讲透。Java 只讲核心点，重点深入 C。

### 一、hook() 方法签名（Java 简略，只讲核心点）

```java
public static MethodHook.Unhook hook(Member method, MethodHook callback, boolean canInitDeclaringClass)
```

核心点（一句话）：`Member` 是 `Method`/`Constructor`/`Field` 的公共接口，用它能同时 hook 方法和构造器；`Unhook` 返回值是"撤销句柄"，调 `unhook()` 能解除这次 hook（类比 `registerReceiver` 返回注销句柄）。

方法体校验段（344-362）排除的是"ART 里没有机器码可改"的方法：抽象方法无方法体、`<clinit>` 静态构造器不是普通方法。

### 二、ensureInitialized → init0（Java → C 的分界）

`ensureInitialized()`（364 行）双检锁，首次调 `initialize()` → `PierceNative.init0(...)`（Pierce.java:174）。**init0 就是 Java 踏入 C 的第一脚。**

### 三、C 核心：NativeMark 是什么（疑问的根）

**NativeMark.java:8**：

```java
/**
 * Internal ruler used for calculating ArtMethod size and members offset.
 */
final class NativeMark {
    private static native void m1(float f);   // 两个 native 方法
    private static native void m2();
    private interface I { void m(); }
}
```

**关键认知链**：

1. `m1`、`m2` 是两个 **native 方法**（Java 侧只有 `native` 声明，无方法体）。
2. ART 运行时，**每个 Java 方法（不管是否 native）在内存里都有一个 `ArtMethod` 结构体**。
3. 同一类的所有方法 ArtMethod 在内存里**连续排列**，所以 m1、m2 背后各有一个 ArtMethod 结构体，地址相减 = 单个 ArtMethod 结构体大小。

**"Ruler"** 是 `NativeMark` 类在 C++ 里的引用名（jni_bridge.cpp:34），作者把这个类叫"标尺/尺子"，专门量 ArtMethod 大小和偏移。

### 四、JNI 注册机制（为什么 m1 注册、m2 不注册）

#### native 方法的调用链

```
调用 m1()
  → ART 读 m1.entry_point_from_compiled_code_（native 方法指向 generic jni trampoline 公共桩）
  → 桩读 m1.entry_point_from_jni_（真正的 C 函数地址）
  → 跳转到 C 函数执行
```

所以 **`entry_point_from_jni_` = "native 方法的 C 实现地址"**。

#### 两种注册方式

**A. 静态注册**：函数名 `Java_包名_类名_方法名`，ART 用 dlsym 按符号名找。慢、名字长易错。

**B. 动态注册（RegisterNatives，本工程用）**：

```cpp
// ruler.cpp:13-19
static const JNINativeMethod gMethods[] = {
    {m1(), "(F)V", reinterpret_cast<void*>(Ruler_m1)}   // 显式绑定 m1 → Ruler_m1
};
bool register_Ruler(JNIEnv* env, jclass Ruler) {
    return env->RegisterNatives(Ruler, gMethods, 1) == JNI_OK;
}
```

`RegisterNatives` 把 Ruler*m1 地址写进 m1 的 `entry_point_from_jni*` 字段。

#### 为什么"获取 ArtMethod"还要注册

注册不是为了获取结构体，而是**往 m1 的 entry*point_from_jni* 字段"埋一个已知标记值（Ruler_m1 地址）"**，扫描时靠这个标记定位字段位置。

#### 为什么 m2 不注册

m2 只干一件事：提供第二个 ArtMethod 地址，和 m1 相减算 size。它从不参与扫描字段值，所以 entry*point_from_jni* 指向谁不重要，只需"存在一个结构体、有个地址"。

**一句话**：m1 是"带标记的尺子刻度"（要注册埋标记），m2 是"尺子另一端"（只要位置）。

### 五、Ruler_m1 地址为什么"编译时确定"

1. 编译：C 代码 → 机器码 → 放进 .so 的 `.text` 段。
2. 符号表：`Ruler_m1` 符号 → .text 段的相对位置。
3. 加载重定位：loadLibrary 时，动态链接器把 .so 映射进内存，相对地址 + 基址 = 真实地址。
4. C 里"函数名单独出现"就代表函数地址。

所以 Ruler_m1 是我们自己的函数，运行时地址稳定、可预测、已知。对比 `art_quick_to_interpreter_bridge` 是 libart.so 的符号，编译时不知道，得用 ELF 解析查。

### 六、InitMembers 逐行（art_method.cpp:128-158）—— 本课核心

```cpp
void ArtMethod::InitMembers(JNIEnv *env, ArtMethod *m1, ArtMethod *m2, ArtMethod *m3, uint32_t access_flags) {
    // ① 按版本初始化 access flag 常量值
    if (Android::version >= Android::kN) {
        kAccCompileDontBother = (Android::version >= Android::kOMr1)
                                ? AccessFlags::kCompileDontBother_O_MR1
                                : AccessFlags::kCompileDontBother_N;
    }
    // ② 算结构体大小：两个相邻方法地址相减
    size = Difference(reinterpret_cast<intptr_t>(m1), reinterpret_cast<intptr_t>(m2));
    // ③ declaring_class 偏移（M+ 是 0，更老版本是 sizeof(void*)）
    declaring_class.SetOffset(android_version >= Android::kM ? 0 : sizeof(void *));
    // ④ 探测 access_flags_ 偏移
    InitMemberAccessFlags(m1, access_flags);
    // ⑤ 探测 entry_point 系列偏移
    InitMemberEntryPoint(m1);
}
```

#### 行②：Difference（art_method.cpp:122）

```cpp
static inline size_t Difference(intptr_t a, intptr_t b) {
    intptr_t size = b - a;
    if (size < 0) size = -size;   // 取绝对值
    return size;
}
```

`reinterpret_cast<intptr_t>(m1)` 把指针强行解释成整数（`intptr_t` 是"足够存指针"的整数类型）。转整数后相减得纯字节差。

### 七、偏移扫描底层：FindOffset + safeReadValue（memory.h:30-49）

```cpp
template<typename T>
static int FindOffset(void *start, T value, size_t size, uint step) {
    for (uint32_t offset = 0; offset < size; offset += step) {
        T currentValue = safeReadValue<T>(start, offset);
        if (currentValue == value) return offset;   // 相等 → 字段偏移
    }
    return -1;
}

template<typename T>
static T safeReadValue(const void *start, uint32_t offset) {
    T value;
    memcpy(&value, (void*)((uintptr_t)start + offset), sizeof(T));
    return value;
}
```

`safeReadValue` 逐字：

1. `(uintptr_t)start + offset`：起始指针转整数 + 偏移 = 目标字段绝对地址。
2. `(void*)...`：整数转回指针。
3. `memcpy(&value, 源地址, sizeof(T))`：从源地址拷贝 sizeof(T) 字节。
4. 返回 value。

**偏移扫描 = 暴力枚举 + 已知值比对，没有魔法。**

### 八、探测 access*flags*（InitMemberAccessFlags，art_method.cpp:160-199）

思路：已知 m1 的 access*flags 值（Java 反射 getAccessFlags 拿到），扫描找"哪个位置的值 == 已知值"，就是 access_flags* 偏移。步长 2 因为字段通常 2 字节对齐。

### 九、探测 entry_point（InitMemberEntryPoint，art_method.cpp:201-251）

```cpp
// ① 扫 entry_point_from_jni_：找值 == Ruler_m1 的位置
for (uint32_t offset = 0; offset < size; offset += 2) {
    if (Memory::safeReadValue<void*>(m1, offset) == Ruler_m1) {
        entry_point_from_jni_.SetOffset(offset);
        break;
    }
}
// ② compiled_code = jni + 一个指针大小（O+ 要 AlignUp 对齐）
uint32_t compiled_code_entry_offset = entry_point_from_jni_.GetOffset() + entry_point_member_size;
if (Android::version >= Android::kO) {
    compiled_code_entry_offset = Memory::AlignUp(compiled_code_entry_offset, entry_point_member_size);
}
entry_point_from_compiled_code_.SetOffset(compiled_code_entry_offset);
```

entry*point_from_compiled_code* 不用单独扫，因为它紧跟 entry*point_from_jni* 后面（ART 布局相邻），+sizeof(void\*) 即可（O+ 要对齐，因为 PACKED(8) 可能有 padding）。

### 十、答疑（七个问题）

#### 问1+2：为什么 m2 不注册？为什么获取 ArtMethod 要注册？

见本课"四"节。m1 注册是为了埋已知标记 Ruler_m1 供扫描；m2 只提供地址算 size，不需注册。

#### 问3：Ruler_m1 地址为什么编译时确定？

见本课"五"节。

#### 问4：位标志 bit flags 详解

**核心思想**：一个 32 位整数存 32 个独立"是/否"开关，每一位（bit）代表一个属性。

| 位号 | 数值   | 含义（例） |
| ---- | ------ | ---------- |
| 0    | 1 = 2⁰ | public     |
| 1    | 2      | private    |
| 2    | 4      | protected  |
| 3    | 8      | static     |
| 4    | 16     | final      |
| 5    | 32     | native     |

举例："public static native" = 1 + 8 + 32 = 41 = 二进制 `00101001`。

三个操作：

- **设置**：`flags | 16`（按位或，任一为1结果1）
- **清除**：`flags & ~1`（取反 + 按位与）
- **检查**：`flags & 8`（非0说明该位是1）

为什么用位标志：省空间（1个int vs 32个boolean）、快（一条位运算指令）。

#### 问5：ArtMethod 字段布局固定吗？declaring_class 一定是第一个吗？

不固定，随 Android 版本变（字段增删、顺序调整、类型变化）。这是动态适配的根本原因。

declaring_class 不是恒定第一个：M+ 偏移 0，M- 偏移 sizeof(void\*)。所以不扫描而写死，因为规则简单（两种情况）。access_flags/entry_point 规则复杂，才扫描。

#### 问6：两种类型强转

```cpp
reinterpret_cast<intptr_t>(m1)   // C++ 风格
(uintptr_t)start                  // C 风格
```

功能一样：把指针值重新解释成整数。`reinterpret_cast` 是 C++ 明确的位重解释运算符；`(uintptr_t)` 是 C 遗留简洁写法。转整数是因为 C++ 指针不能直接相减得字节数。

#### 问7：内存对齐 alignment

**每种类型有对齐要求：起始地址必须是其大小的整数倍。**

| 类型       | 大小 | 对齐 |
| ---------- | ---- | ---- |
| char       | 1    | 1    |
| short      | 2    | 2    |
| int        | 4    | 4    |
| 指针(64位) | 8    | 8    |

原因：CPU 按固定块读内存，不对齐要读两次拼接（慢），某些 CPU 直接抛异常（崩）。

**struct padding**：

```c
struct S { char a; int b; };
// 实际布局：char a(偏移0) + 3字节padding + int b(偏移4)，sizeof=8（不是5）
```

因为 b 需要 4 字节对齐，不能放偏移 1，编译器插 3 字节 padding。

**对应工程**：Android O+ ArtMethod 指针字段组 PACKED(8)，jni 和 compiled_code 之间可能有 padding，所以 `AlignUp`：

```cpp
template <typename T>
static inline T AlignUp(T value, T align_with) {
    T alignment = value % align_with;
    if (alignment) value += (align_with - alignment);
    return value;
}
```

**为什么 safeReadValue 用 memcpy 不直接解引用**：扫描步长 2，读 8 字节指针时地址可能落在 2、6、10 这种不对齐位置，直接 `*(T*)` 会未对齐访问崩溃；memcpy 按字节拷贝，安全无视对齐。

### 十一、C 知识清单（本课掌握）

| #   | 知识点                        | 一句话                                          |
| --- | ----------------------------- | ----------------------------------------------- |
| 1   | 指针 = 地址 = 整数            | m1 的值就是内存地址                             |
| 2   | native 方法 + RegisterNatives | Java 声明 native，C 侧 RegisterNatives 绑定实现 |
| 3   | JNI_OnLoad                    | .so 加载时 ART 自动调用，完成注册               |
| 4   | struct 连续布局 + 偏移        | 偏移 = 字段离结构体起始的字节距离               |
| 5   | 起始地址 ≠ 偏移               | 起始地址是楼地址，偏移是房间号                  |
| 6   | memcpy 字节拷贝               | 从任意地址安全读数据，避对齐/可读性坑           |
| 7   | 内存对齐 + AlignUp            | O+ 字段有 padding，要向上对齐                   |
| 8   | 位标志 bit flags              | access_flags 用 32 位整数每个 bit 表示一个属性  |
| 9   | reinterpret_cast / intptr_t   | 指针↔整数互转，算字节差                         |

**一句话概括整课**：NativeMark 是作者造的"尺子"，m1/m2 提供"两个已知 ArtMethod 结构体 + 一个已知 C 函数地址"，InitMembers 用"暴力扫描 + 已知值比对"测出 ArtMethod 各字段偏移，存进 Member 对象，供 hook 改入口时使用。

---

## 第八课：Android::Init —— ELF 符号解析与符号地址计算（C 核心）

> 本课对应 `Pine_init0` 的第 3 处细节：`Android::Init`（android.cpp:35-113）。这是整条链 C/OS 知识最密集的一块。
> 核心解决一个问题：**hook 要调 libart.so 里的函数（如 SuspendVM），但编译时不知道它们的地址，必须在运行时从 libart.so 里"查符号"找出来。**

### 一、学习策略：C 代码要不要每行都背？

**结论：不用每行背。目标是"改 bug"，不是"重写 Pine"。**

判断标准一句话：**这段代码真出 bug 时，您要改的是它本身，还是它的调用方？**

| 层次 | 代码类型 | 要掌握的程度 | 本仓库对应 |
| --- | --- | --- | --- |
| 核心逻辑层 | hook 安装、backup/bridge 串联、ArtMethod 偏移探测 | 逐行懂，能改 | art_method.cpp 的 InitMembers/BackupFrom/AfterHook |
| 工具成熟层 | ELF 解析、xz 解压、符号表遍历、mprotect | 懂原理 + 能定位到行，不用背算法 | elf_image.cpp、memory.h |
| 架构汇编层 | 机器码跳板、arm64 汇编 | 懂"跳到哪"即可，最后啃 | trampoline/*.S |

**为什么 ELF 解析这类"工具成熟层"不用逐行懂**，三个理由：

1. 成熟代码，从 SandHook 继承，上万个 App 用了多年，`sh_offset`/`sh_entsize` 算法不会出 bug。
2. 它真出 bug 的形态几乎只有一种——"某 ROM 的符号查不到"，修复点在"该查哪个符号名"，不在解析算法本身。
3. ELF 是 Google 定死的标准格式，不会改，您改了反而坏事。

**现成证据**（android.cpp:259-266）：

```cpp
//这里处理和系统规定不一致的特殊情况
//honor magic 4 android 15 ,符号却是Android 14的版本
```

这证明：真正的 bug 来源是"哪个版本/厂商该查哪个符号名"，不是"符号表怎么遍历"。

**一句话记牢**：ELF 解析是"查字典"的工具，字典算法不用背，但要背"该查哪个词"。

### 二、为什么要有 Android::Init？

之前 `InitMembers` 用扫描探测了 ArtMethod 自己的字段偏移。但 hook 还需要另一批东西：**ART 运行时（libart.so）里的函数地址**，比如 `SuspendVM`/`ResumeVM`（挂起/恢复 VM）、`ArtMethod::CopyFrom`（复制方法）。

这些函数在 libart.so 里，是 Google 写的、版本各异的、编译时我们不知道地址的。`Android::Init` 的任务：**运行时从 libart.so 里把这些函数地址找出来，存进全局函数指针**，供 hook 调用。

### 三、两种"方法"，两套拿地址的机制（重要区分）

| | 是什么 | 例子 | 拿地址的方式 |
| --- | --- | --- | --- |
| ART 内部函数 | libart.so 里的 C++ 函数，Google 写的 | SuspendVM、ArtMethod::CopyFrom | ELF 符号解析（查符号表） |
| 用户 hook 的 Java 方法 | App 里（或别人 App 里）的 Java 方法 | add()、testTouTiaoJSON() | JNI 传 jmethodID → ArtMethod，不查符号表 |

**关键：hook 的业务方法走第二行，从头到尾不碰 ELF 符号表。**

因为 Java 方法有 jmethodID 这个"身份证"：

```
Java 反射 getDeclaredMethod("add") → jmethodID → native FromReflectedMethod → ArtMethod 指针 → 改入口
```

业务方法编译在 dex 里，没有 C++ 符号名，也不在 libart.so 符号表里，所以"符号解析"对业务方法根本不存在。

### 四、ELF 是什么（OS 核心概念）

**ELF（Executable and Linkable Format）** 是 Linux/Android 可执行文件和共享库的标准格式。Android 的 `.so`（类似 Windows 的 DLL）、可执行文件都是 ELF。`libart.so` 就是一个 ELF 文件。

一个 ELF 文件分成多个段（section），工程关心的几个：

| 段名 | 存什么 | 作用 |
| --- | --- | --- |
| .text | 机器码 | 函数编译后的指令 |
| .dynsym | 动态符号表 | 导出给外部用的符号 |
| .dynstr | 动态字符串表 | 符号的名字（字符串） |
| .symtab | 完整符号表 | 所有符号（含调试信息） |
| .strtab | 字符串表 | 完整符号名字 |

**符号表（symbol table）** = "符号名 → 地址偏移"的映射，像一个花名册。**符号就是函数或全局变量的名字**，符号表就是"名字 → 地址"的字典。

找函数地址 = 拿着名字去符号表查偏移，再算真实地址。这就是 `GetSymbolAddress` 的全部。

### 五、Android::Init 逐行（android.cpp:35-113）

#### 第 1 步：存版本 + 拿 JavaVM（36-41）

```cpp
Android::version = sdk_version;
if (env->GetJavaVM(&jvm_) != JNI_OK) abort();
```

**JavaVM vs JNIEnv**：JavaVM 是进程唯一、代表"虚拟机本身"、跨线程可用（全局）；JNIEnv 每线程一个、只当前线程用。`GetJavaVM` = 从当前线程 JNIEnv 拿全局 JavaVM，存进全局变量 `jvm_`，供后续其他线程用。

**类比**：JavaVM 是"公司总部"（全局一个），JNIEnv 是"工牌"（每人一个，只在自己工位有效）。

#### 第 2 步：打开 libart.so（43-62）

```cpp
ElfImage art_lib_handle("libart.so");
if (!art_lib_handle.IsOpened()) {
    // eng 调试版 ROM：libartd.so
    // 阿里 YunOS：libaoc.so
}
```

`ElfImage` 是本工程封装类（elf_image.h:43），负责"加载一个 ELF 文件、解析符号表"。构造时若打不开，依次兜底 `libartd.so`（eng 调试 ROM）、`libaoc.so`（YunOS AOC 运行时）——适配不同 ROM。

**ElfImage::Open 内部（elf_image.cpp:22-59）**：

```cpp
int fd = WrappedOpen(path, O_RDONLY | O_CLOEXEC);   // ① 打开文件
size_ = lseek(fd, 0, SEEK_END);                     // ② 求文件大小
base_ = GetModuleBase(path);                        // ③ 找加载基址（关键！）
header_ = mmap(nullptr, size_, PROT_READ, MAP_SHARED, fd, 0);  // ④ mmap 映射
ParseMemory(header_, false);                        // ⑤ 解析符号表
```

两个 OS 核心概念：

**概念 A：mmap** —— 系统调用，把文件内容映射到进程虚拟内存。映射后文件内容像"内存数组"一样，可直接指针访问，不用 read 逐个读。

**概念 B：GetModuleBase** —— 找加载基址（elf_image.cpp:235-260）。`libart.so` 被加载时映射到进程内存某基址（如 0x7f00000000），文件符号偏移 + 基址 = 运行时真实地址。基址从 **`/proc/self/maps`**（Linux 虚拟文件，记录进程所有内存映射）里解析出来。

#### 第 3 步：按版本解析符号（64-95）

```cpp
if (Android::version >= Android::kR) {
    suspend_all = ...GetSymbolAddress("_ZN3art16ScopedSuspendAllC1EPKcb");
    resume_all  = ...GetSymbolAddress("_ZN3art16ScopedSuspendAllD1Ev");
} else {
    suspend_vm = ...GetSymbolAddress("_ZN3art3Dbg9SuspendVMEv");
    resume_vm  = ...GetSymbolAddress("_ZN3art3Dbg8ResumeVMEv");
}
```

两个知识点：

**知识点 1：name mangling（名字捣碎）** —— 符号名是一堆乱码的原因。C++ 支持函数重载，同名函数参数不同，编译后必须有不同的符号名，否则链接器分不清。编译器把"函数名 + 参数类型"编码成唯一符号名。

解码 `_ZN3art16ScopedSuspendAllC1EPKcb`：

```
_ZN  = C++ 名字捣碎前缀
3art = 长度3 + "art"（命名空间）
16ScopedSuspendAll = 长度16 + "ScopedSuspendAll"（类名）
C1   = 构造函数
E    = 参数列表开始
PKc  = 参数 const char*（P=指针，K=const，c=char）
b    = 参数 bool
```

翻译：`art::ScopedSuspendAll::ScopedSuspendAll(const char*, bool)` —— 构造函数。

**知识点 2：函数指针** —— 存"函数机器码地址"的变量。

声明格式：`返回类型 (*指针名)(参数类型列表)`

```c
int add(int a, int b) { return a + b; }
int (*p)(int, int);   // 声明指针 p
p = add;              // 函数名本身是地址
int r = p(3, 4);      // 调用，等价 add(3,4)
```

`reinterpret_cast<void(*)(void*, const char*, bool)>` 把 `GetSymbolAddress` 返回的 `void*` 裸地址，强转成"能调用"的函数指针。函数指针本质 = 地址 + 签名约定，调用时就一条跳转指令。

#### 第 4 步：GetSymbolAddress（elf_image.cpp:219-229）

```cpp
void *ElfImage::GetSymbolAddress(const char *name, ...) const {
    if (!base_) return nullptr;
    Elf_Addr offset = LinearLookup(name, out_size);   // ① 查偏移
    if (!offset) return nullptr;
    return reinterpret_cast<void*>(reinterpret_cast<uintptr_t>(base_) + offset - bias_);  // ② 算真实地址
}
```

**真实地址 = 基址(base_) + 符号偏移(offset) - bias_**

`bias_` 是文件偏移和虚拟地址的差异修正量，通常很小或为 0。

#### 第 5 步：LinearLookup（elf_image.cpp:174-217）

线性查找：逐个遍历符号表，用 `strcmp` 比较符号名，匹配就返回 `st_value`（偏移）。

```cpp
Elf_Sym *sym = dynsym_;
for (Elf_Off i = 0; i < dynsym_count_; i++, sym++) {
    if (strcmp(dynstr_ + sym->st_name, name) == 0)   // 符号名匹配？
        return sym->st_value;                        // 返回偏移
}
```

复杂度 O(n)，查一个符号扫一遍。所以代码注释写 `// TODO: hash based lookup`——作者知道慢，理想用哈希表（系统 dlsym 就是哈希 O(1)），但 Pine 不能直接用 dlsym（libart 很多符号是 hidden，不在 .dynsym 导出表里），只能自己解析 .symtab 完整符号表。

### 六、DisableHiddenApiPolicy（android.cpp:115-165）

**目的**：绕过 Android 9+ 的 hidden API 限制（非系统 App 访问 @hide 内部 API 会被拒绝）。Pine 自己要反射访问 ART 内部类/字段，被限制挡了，必须绕过。

```cpp
static int FakeHandleHiddenApi() {   // 假函数，永远返回 0
    return 0;                        // 0 = false = "不拒绝访问"
}

trampoline_installer->NativeHookNoBackup(target, replace);  // inline hook 替换
```

**一句话**：ART 每次检查"该不该拒绝访问 hidden 成员"时，调用的其实是 Pine 塞进去的 `FakeHandleHiddenApi`，永远返回"不拒绝"，于是所有 hidden API 都能访问。

### 七、答疑（本课所有提问）

#### 问1：so 里的 add 方法，偏移量、内存地址怎么找？（结合 ELF + mmap）

先区分两个易混淆的"偏移"：

| 名称 | 含义 | 谁决定的 |
| --- | --- | --- |
| 文件偏移（sh_offset） | add 机器码在 .so 文件里第几个字节 | 编译/链接器按 ELF 段布局排的 |
| 虚拟地址偏移（st_value / sh_addr） | add 在内存里相对"加载基址"的偏移 | 链接器分配的"逻辑地址" |

这俩数字往往不相等（段之间有对齐填充）。

运行时真实地址公式：`add 真实地址 = 加载基址(base_) + 虚拟地址偏移(st_value) - bias_`

走一遍：系统 dlopen 加载 libfoo.so，mmap 映射到基址 0x7000000000；符号表查 "add" 得 st_value=0x1200；真实地址 = 0x7000000000 + 0x1200 = 0x7000001200。

**mmap 的作用**：文件是"死"的，mmap 把它投影到内存后，文件内容才能被 CPU 当内存访问/执行，base_ 就是投影起点。

**类比（Android）**：像 RecyclerView 的 LayoutManager。文件偏移 = 数据项在"数据源列表"的位置；虚拟地址偏移 = 该项在"屏幕可视区域"坐标；base_ = 屏幕起点。

#### 问2：按版本解析符号干嘛的？定义固定吗？

**干嘛的**：不同 Android 版本，ART 内部函数名字和签名都会变，要按版本查不同符号。Google 在 Android 11 把挂起 VM 从 `Dbg::SuspendVM()` 重构成 `ScopedSuspendAll` RAII 类，函数名参数全变，必须分开查。

**定义完全不固定**，这正是 hook 维护最痛苦处：

1. 版本会变（Android 大版本重构/改名/改签名）
2. 厂商会变（荣耀 Magic4 是 Android 15 却用 Android 14 的符号，android.cpp:259-266）
3. 架构会变（arm32 上某些函数被内联查不到，arm64 没有，android.cpp:148-151）

这些乱码字符串是维护者从 AOSP 源码反查出来的 mangled 名。维护时遇到"符号查不到"，要做的**去对应版本 AOSP 源码查新符号名，不是改解析算法**。

#### 问3：函数指针详细介绍

见本课"五·第3步·知识点2"。本质 = 地址 + 签名约定，底层就一条跳转指令。类比 Java 接口回调，但更底层——Java 存对象引用靠虚方法分发，C 存纯地址 CPU 直接 jump。

#### 问4：DisableHiddenApiPolicy 简要说明

见本课"六"节。

#### 问5：LinearLookup 举例说明（Android 代码类比）

```java
// Java 版：在联系人列表找名字匹配的人，返回电话
String findPhone(List<Contact> list, String name) {
    for (Contact c : list) {
        if (c.name.equals(name)) return c.phone;
    }
    return null;
}
```

对照 C 版（elf_image.cpp:174-185）：

- `dynsym_` = `List<Contact>`（符号表，一个符号 = 一个 Contact）
- `strcmp(...) == 0` = `c.name.equals(name)`（字符串比较，相等返回 0）
- `sym->st_value` = `c.phone`（符号地址偏移）
- 没找到返回 0 / null

#### 问6：是不是每 hook 一个方法都要做一次符号解析？

**不是。符号解析只发生在 Pine 初始化那一次，且只针对 libart.so 里 ART 自己的内部函数。业务方法走 jmethodID，不查符号表。**

ART 内部函数：初始化时一次性查符号（几十个固定函数），存进全局函数指针，以后反复用不再查。
业务方法：Java 反射 + JNI 传 jmethodID → ArtMethod 指针，不做任何符号解析。

"同名不同参"也不混淆：业务方法的 Java 重载在 dex 里就是两个独立 ArtMethod + 两个不同 jmethodID，反射拿哪个就是哪个；libart 的 C++ 重载 mangled 名已编码参数类型（add(int)→_Z3addi，add(long)→_Z3addl），是不同的字符串。

#### 问7：Pine hook 时要调用这些函数吗？算地址前重载/版本要做符号解析吗？

**（1）对**：hook 安装改 ArtMethod 时，不能有别的线程同时在跑 Java 代码，所以 Pine_hook0 先 `suspend_vm()` 挂起 VM（stop-the-world），改完再 `resume_vm()`。这就是为什么要解析 SuspendVM。

**（2）时间点纠正**：符号解析不是"用到时才做"，而是**初始化时一次性做完**，地址存进全局函数指针（Android::suspend_vm 等），以后 hook 只是"调用已算好的指针"，不再查符号表。

**（3）重载/版本差异**：不是"存在才触发解析"，而是符号解析天生就要处理这两件事——mangled 已编码参数（解决重载），if/else 查不同字符串（解决版本）。都是初始化那一次写好，不是运行时动态发生。

**类比**：符号解析 ≈ onCreate 里一次性 getSystemService() 存单例，以后直接单例调用，而不是每次 getSystemService。

#### 问8：怎么知道哪些函数需要解析、哪些有重载？

三部分来源：

| 来源 | 性质 | 说明 |
| --- | --- | --- |
| 需求驱动 | 设计期主动 | Pine 功能要调 libart 哪个内部函数，就解析哪个 |
| AOSP 源码 | 唯一事实来源 | 去 AOSP 查函数真实签名，算 mangled 名，对比版本差异 |
| 测试出 bug | 运营期补 | 某厂商/架构/云机符号查不到，补 fallback 分支 |

证据（测试踩坑后补的 fallback）：
- 荣耀 Magic4（android.cpp:259-266）
- arm32 内联（android.cpp:148-151）
- 云机小米14 Pro（android.cpp:214-215）

**维护工作流**（新增功能或修 bug 时）：去 AOSP 查签名 → 算 mangled 名（或用 c++filt 反查）→ 写 GetSymbolAddress("乱码串") + 按版本写 if/else → 真机各版本测试，查不到补 fallback。这一步不需要等 bug 才做。

#### 问9：这部分代码/原理必须吃透吗？

**分层**：

| 子块 | 是否必须吃透 | 原因 |
| --- | --- | --- |
| 整体机制（符号名→地址流程） | 必须懂 | 理解 hook 地基 |
| 版本分支逻辑（if/else 查新符号旧符号） | 必须懂，且要会改 | 真实 bug 高发区 |
| 解析算法细节（sh_offset/sh_entsize/bias_/LinearLookup 逐行） | 不用吃透，懂原理即可 | 成熟代码不会出 bug |

**类比**：RecyclerView 的 LayoutManager 内部回收算法懂"复用 View"即可，不用懂双向链表怎么维护；但"何时用 Linear 还是 Grid"必须懂。

**收口**：机制要懂，版本分支要会改，算法细节放过。bias_、sh_entsize 这些知道"有这个东西、作用是修正偏移"即可，不用背公式。

### 八、C 知识清单（本课掌握）

| # | 知识点 | 一句话 |
| --- | --- | --- |
| 1 | ELF 格式 | Linux/Android 可执行文件标准，分 .text/.dynsym/.dynstr 等段 |
| 2 | 符号表 | "符号名 → 地址偏移"的字典 |
| 3 | JavaVM vs JNIEnv | JavaVM 全局唯一，JNIEnv 每线程一个 |
| 4 | mmap | 系统调用，把文件映射到内存，指针直接访问 |
| 5 | /proc/self/maps | Linux 虚拟文件，记录进程内存映射，找加载基址 |
| 6 | name mangling | C++ 名字捣碎，解决重载；符号名是乱码的原因 |
| 7 | 函数指针 | 存函数地址的变量，`返回值(*)(参数)` |
| 8 | strcmp | C 字符串比较，相等返回 0 |
| 9 | LinearLookup | 线性遍历符号表比对名字，O(n) |
| 10 | 真实地址 = 基址 + 偏移 | 符号偏移 + 加载基址 = 运行时地址 |

**一句话概括整课**：`Android::Init` 用 `ElfImage` 打开 libart.so，`mmap` 映射进内存，遍历符号表用 `strcmp` 比对（乱码符号名是 C++ name mangling 的结果），找到函数偏移后加基址算真实地址，存进全局函数指针，供 hook 调 ART 内部函数。

---

## 第九课：JNI 引用 RAII（ScopedLocalRef）与 jmethodID→ArtMethod（Require）

> 本课对应 `Pine_init0` 第 4 处细节。为什么讲：第七课讲了 `InitMembers` 需要 m1/m2 两个 ArtMethod 指针，但没讲这两个指针怎么从"方法名"变出来。本课补上——就是 pine.cpp:114-117 三行。
> 代码位置：pine.cpp:114-117，内部方法 `Require` 在 art_method.cpp:109-120，RAII 类在 scoped_local_ref.h。

### 一、这三行在干嘛（一句话）

```cpp
// pine.cpp:114-117
ScopedLocalClassRef Ruler(env, com___m4399___pierce___core___NativeMark);   // ① 拿 NativeMark 类
auto m1 = art::ArtMethod::Require(env, Ruler.Get(), m1(), "(F)V", true);    // ② 拿 m1 的 ArtMethod
auto m2 = art::ArtMethod::Require(env, Ruler.Get(), m2(), "()V", true);     // ③ 拿 m2 的 ArtMethod
```

**把"方法名 m1/m2"换成"它们的 ArtMethod 指针"**，交给后面的 `InitMembers` 用。

### 二、进入内部方法 Require（art_method.cpp:109-120）

```cpp
ArtMethod* ArtMethod::Require(JNIEnv* env, jclass c, const char* name, const char* signature, bool is_static) {
    jmethodID m = is_static ? env->GetStaticMethodID(c, name, signature)   // ① 按名字拿方法句柄
                            : env->GetMethodID(c, name, signature);
    if (Android::version >= Android::kR) {
        if (reinterpret_cast<uintptr_t>(m) & 1) {                          // ② 判断最低位
            ScopedLocalRef javaMethod(env, env->ToReflectedMethod(c, m, ...));
            return GetArtMethodForR(env, javaMethod.Get());                // ③ 绕道读字段
        }
    }
    return reinterpret_cast<ArtMethod*>(m);                                // ④ 直接强转
}
```

C++ 语法点出（简单一句）：
- **三目运算符** `条件 ? 值1 : 值2`：`is_static` 为真用 `GetStaticMethodID`，否则用 `GetMethodID`。
- **`reinterpret_cast<uintptr_t>(m)`**：把 jmethodID 这个指针值重新解释成整数，才能做 `& 1` 位运算（指针不能直接位与）。

**唯一要记的坑：jmethodID 在 Android R 前后含义变了。**

- **Android < R**：`jmethodID` 的值**就是 ArtMethod 结构体地址**，直接 `reinterpret_cast` 强转（第④步）。
- **Android >= R**：Google 改了实现，`jmethodID` 不再是直接指针。判断方式 `& 1` 看最低位——ArtMethod 地址是 8 字节对齐的，最低位本该是 0，R 版本拿它当标志位。是 1 就绕道：`ToReflectedMethod` 拿回 Java 的 `Method` 对象，再进 `GetArtMethodForR`（art_method.h:35-42）读它的 `artMethod` 字段（这个字段仍存真实指针）。

**类比（Android）**：R 之前 jmethodID 是"员工本人"，R 之后变成"工牌号"，要拿工牌去查档案才能找到真人。

### 三、RAII 类 ScopedLocalClassRef（scoped_local_ref.h）

`Ruler` 就是 `ScopedLocalClassRef`，它管一个 JNI 的坑：

**`FindClass` 返回的 jclass 是 "local ref"，有数量上限（约 512/低内存 16），用完不 `DeleteLocalRef` 会堆满崩溃。**

`ScopedLocalClassRef` 用 **RAII**（C++ 版 try-with-resources）解决：构造时拿引用，对象离开作用域时析构自动 `DeleteLocalRef`。你只管用，不用记着删。

C++ 语法点出：`ScopedLocalClassRef` 是 `ScopedLocalRef<jclass>` 的子类（继承），`<jclass>` 是**模板**（一份代码适配多种类型，类比 Java 泛型）。析构函数 `~ScopedLocalRef()` 里调 `Reset()` 自动删引用——这就是 RAII 的落点。

**类比（Android）**：就像 Bitmap 要手动 recycle，这个类等于帮你自动 recycle 的包装。

### 四、本课要点清单

| 要点 | 一句话 |
| --- | --- |
| jmethodID 版本坑 | < R 是 ArtMethod 指针；>= R 变了，靠 `& 1` 判断是否绕道 |
| local ref 泄漏 | FindClass 返回的 jclass 有上限，用完要删，RAII 自动删 |
| 调用链 | `Require` → `GetStaticMethodID` →（R 版本）`ToReflectedMethod` → `GetArtMethodForR` |

**一句话概括整课**：`ScopedLocalClassRef` 用 RAII 自动管理 JNI local ref（防泄漏/防超限崩溃），`Require` 负责把"方法名"转成"ArtMethod 指针"——`< R` 直接强转 jmethodID，`>= R` 绕道 Method 对象的 `artMethod` 字段。

---

## 第十课：REPLACEMENT 模式总收口 —— 用 add(a,b) 走完整条 hook 链路

> 本课是 REPLACEMENT 模式（Android O+ 默认）的**最终收口**：把前面第一课到第九课散落的点，用 `int add(int a, int b)` 一个例子串成一条完整链路，并对 BackupFrom 的 JIT/GC 迁移做**图解法**详解。
> 注意：本课基于**当前分支**的最新代码（`AfterHook` 签名已从旧的 `is_native_or_proxy` 拆成 `is_native, is_proxy`，并新增 kFastNative/kCriticalNative 清除）。

### 一、两个贯穿全局的前置概念（先钉死，后面全用它）

#### 概念 1：偏移（offset）是"刻度"，入口值（value）是"读数"

init0 扫描算出来的是**偏移 offset**（"这个字段在 ArtMethod 的第几个字节"），全局一份、所有方法通用。hook 时才是拿这个刻度去某个具体方法（add）的对象上**读写读数**。

看 `Member::Set`（member.h:36-41）就一目了然：

```cpp
void Set(IType* instance, MType value) {
    memcpy(reinterpret_cast<void*>((uintptr_t) instance + offset), &value, sizeof(MType));
}
```

公式：**字段实际地址 = ArtMethod 实例基址(instance) + 偏移(offset)**。

| 概念 | 谁 | 什么时候 | 含义 |
|------|-----|---------|------|
| 偏移 offset（刻度） | init0 扫描得到 | 初始化时，一次性 | "入口字段在第 40 字节" |
| 入口值 value（读数） | 每个方法各自 | hook 时用 offset 读 | "add 这个字段存的是 0x7000" |

类比 Android：init0 给一张"户型图"（每房间在第几格），hook 才拿图去某一户（add）找卧室看里面放了啥、再换掉它。

#### 概念 2：ArtMethod 只是"户口本"，执行逻辑（机器码/字节码）在别处

一个方法的"执行逻辑"在内存里有两个形态，**都不在 ArtMethod 结构体里**：

| 形态 | 存哪 | 谁用 | 什么时候有 |
|------|------|------|-----------|
| 字节码 bytecode | dex 文件的 code_item | 解释器 | App 安装就有 |
| 机器码 compiled code | JIT 编译出的可执行内存 | CPU 直接跑 | 方法被 JIT 编译后 |

ArtMethod 结构体本身只是"户口本"（存名字、access_flags、所属类、**入口指针**），不含执行逻辑。它最关键的字段 `entry_point_from_compiled_code_` 是个**指针**，指向机器码入口。

**关键结论（贯穿 hook 本质）**：`BackupFrom` 里的 `copy_from` 只拷贝 ArtMethod 这个"户口本"（几十字节的元数据），**不拷贝机器码、也不拷贝字节码**。机器码（0x7000）和字节码（dex 里）从头到尾都只有一份，backup 是"复用"而非"拷贝"。

---

### 二、REPLACEMENT 完整链路一张图

```
【Java 层 —— 装前准备】
Pierce.hook(add, callback)
  └─ ensureInitialized() → initialize()
       └─ 加载 libpierce.so + init0（扫描 ArtMethod 字段偏移，量刻度）
  └─ getArtMethod(add)             // Java Method → add 的 ArtMethod* 指针
  └─ 登记 HookRecord（一次 ArtMethod 一次）
  └─ hookNewMethod()
       ├─ 定模式：O+ → REPLACEMENT（isInlineHook=false）
       ├─ 收集参数类型 paramTypes = [int, int]
       ├─ genereateDexAndLoad()     // 生成 bridge/backup dex 并加载
       └─ hook0(...)  ←———— Java/native 边界

【C 层 Pine_hook0（pine.cpp:203-330）—— 核心大代码】
  Seg1  拿三个 ArtMethod*：target(add) / bridge / backup
  Seg2  bridge->Compile() 编译 bridge，拿最佳入口
  Seg3  ScopedSuspendVM 挂起全线程（stop-the-world）
        ├─ ① InstallReplacementTrampoline(target, bridge)  改 add 入口 → 跳板
        ├─ ② backup->BackupFrom(target, call_origin, ...)   把 add 状态拷进 backup
        └─ ③ target->AfterHook(...)                         处理 add 的 flags 防 JIT
  Seg4  SetLongField 写 debug 字段 → 返回 javaBackup

【运行时 —— 一次 add(1,2) 调用】
  add(1,2) → 入口=跳板 → bridge → handleCall(回调分发) → before → backup(原逻辑) → after
```

---

### 三、三个核心动作的"状态变化表"

先约定地址（假设值，方便跟踪）：

| 符号 | 地址 | 含义 |
|------|------|------|
| `0x7000` | add 原始机器码入口 | add 编译后的真实代码 |
| `0x8000` | bridge 编译后入口 | bridge 方法代码 |
| `0x9000` | 跳板 | CreateMethodJumpTrampoline 分配的可执行内存 |

#### ① `InstallReplacementTrampoline(target, bridge)`（trampoline_installer.cpp:168-201）—— 只动 add 入口指针

```cpp
void *target_origin_code_entry = target->GetEntryPointFromCompiledCode(); // 记下 0x7000
void *method_jump_trampoline = CreateMethodJumpTrampoline(bridge);        // 造跳板，写死 bridge 入口
target->SetEntryPointFromCompiledCode(method_jump_trampoline);            // add 入口改成 0x9000
return target_origin_code_entry;                                          // 返回 0x7000（=call_origin）
```

| 对象 | hook 前入口 | hook 后入口 |
|------|------------|------------|
| target (add) | `0x7000` | `0x9000`（跳板） |
| bridge | `0x8000` | 不变 |
| backup | 未设置 | 未设置 |

**本质**：把 add 的 `entry_point_from_compiled_code_` 从真代码改成跳板。跳板里写死 bridge 入口，之后谁调 add 都先跑跳板再拐 bridge。

返回值 `0x7000` 是 add 的**原始入口**——replacement 唯一需要"备份"的东西就是这个地址（机器码没被动过，还在 0x7000 躺着）。这正是 replacement 比 inline 简单之处：**inline 要备份"被覆盖的机器码"，replacement 只备份"一个地址"**。

> 代码里有一段注释掉的 `call_origin_trampoline`（trampoline_installer.cpp:182-186）：原版 Pine 会用 `CallOriginTrampoline` 把 r0 寄存器设成原方法再跳原入口，但实测**异常回溯栈时会变僵尸线程**，故弃用，直接返回裸地址。

#### ② `backup->BackupFrom(target, 0x7000, false, false, false)`（art_method.cpp:346-446）—— 重点，见第四节

结果：

| 属性 | add (target) | backup |
|------|-------------|--------|
| 方法体逻辑 | 原 add 逻辑 | 复制了 add 的"户口本"（元数据） |
| 入口 | `0x9000`（跳板） | `0x7000`（add 原代码） |
| access_flags | 加 kAccCompileDontBother | 改成 private + kAccCompileDontBother |

#### ③ `target->AfterHook(false, false, false)`（art_method.cpp:448-519）

给 target 加 `kAccCompileDontBother`（禁止 JIT 再编译它，防止 hook 后入口被 JIT 改回），清 fast-interpreter 缓存标志，设解释器入口。详见第五节。

---

### 四、BackupFrom 详解（art_method.cpp:346-446）—— 最容易出 bug 的方法

它做四件事：

#### 第 1 件：复制"户口本"（347-358）

```cpp
if (LIKELY(copy_from)) {
    copy_from(this, source, sizeof(void*));   // 调 ART 官方 ArtMethod::CopyFrom
} else {
    memcpy(this, source, size);               // 老 Android 兜底：裸 memcpy
}
```

**为什么优先 `copy_from` 而非 `memcpy`**：原版 Pine 用 memcpy 整块拷贝，但 ArtMethod 里有 `GcRoot<Class> declaring_class_` 字段——GC 移动类对象时只更新"原始 ArtMethod"里的地址，不会更新 memcpy 出来的那份，导致 backup 存过期类指针、访问崩溃。`copy_from`（ART 官方 CopyFrom）知道这些字段存在，会正确修正 GcRoot，还顺带处理 JIT 入口、Nterp 入口、hotness counter。**注意：copy_from 拷贝的是"户口本"结构体，不是机器码/字节码。**

#### 第 2 件：修 access_flags（360-375）

```cpp
uint32_t access_flags = source->GetAccessFlags();
if (Android::version >= Android::kN) {
    if (Android::version >= Android::kR) access_flags &= ~kAccPreCompiled;
    access_flags |= kAccCompileDontBother;         // 禁止 JIT 编译 backup
}
if ((access_flags & AccessFlags::kStatic) == 0) {   // 非静态方法（add 就是）
    access_flags &= ~(AccessFlags::kPublic | AccessFlags::kProtected);
    access_flags |= AccessFlags::kPrivate;          // 改成 private，当 direct method 调
}
access_flags &= ~AccessFlags::kConstructor;
SetAccessFlags(access_flags);
```

三个目的：`kAccCompileDontBother` 别让 JIT 编译 backup；非静态改 `kPrivate` 让 backup 能当直接方法被调用；清 `kConstructor`（backup 不是构造器）。

#### 第 3 件：JIT 信息迁移（387-431）—— 核心，野指针的根

**为什么要处理**：hook 把 add 入口从 0x7000 改成 0x9000 后，ART 的 GC 里有一张表 `method_code_map_`（JitCodeCache 维护）记录"哪段机器码属于哪个方法"。此时表里还写着"0x7000 属于 add"，但 add 入口已不再指向 0x7000，GC 判定"0x7000 没人用了"→ **回收**。可 backup 入口刚被我们指向 0x7000 → backup 指向被释放的内存 → 野指针（UAF）随机崩溃。

```cpp
bool clear_jit_info_ref = Android::version >= Android::kN && !is_proxy;  // 390
if (LIKELY(clear_jit_info_ref)) {
    enterMoveJitInfo = Android::MoveJitInfo(source, this);   // 398 先尝试"迁移"
    clear_jit_info_ref = !enterMoveJitInfo;
    if (UNLIKELY(clear_jit_info_ref)) {
        clear_jit_info_ref = !is_inline_hook && !is_native && art_quick_to_interpreter_bridge;  // 405
    }
}
if (UNLIKELY(clear_jit_info_ref)) {
    // 迁移失败 → 清引用：backup 走解释器，别碰可能被回收的机器码
    SetEntryPointFromCompiledCode(art_quick_to_interpreter_bridge);  // 411
    if (Android::version < Android::kS) {
        entry_point_from_jni_.Set(this, nullptr);                     // 421
    }
} else {
    // 迁移成功 → backup 入口 = add 原入口 0x7000
    SetEntryPointFromCompiledCode(entry);                             // 424
    if (UNLIKELY((is_native || is_proxy) && Android::version >= Android::kO)) {
        SetEntryPointFromJni(source->GetEntryPointFromJni());        // 429 恢复 proxy/native 的 jni 入口
    }
}
```

**用内存图理解（这是重点）**：

```
【hook 前，一切正常】
add 的 ArtMethod                          JIT 机器码
┌──────────────────────┐                 ┌──────────────────────┐
│ entry_point = 0x7000 │──指向──────────►│ 0x7000: add 的机器码 │
└──────────────────────┘                 └──────────────────────┘
method_code_map_ 表：{ 0x7000 → add }    ← GC 靠它判断，add 在用 → 不回收

【hook 后（改了 add 入口），危机出现】
add 的 ArtMethod                          JIT 机器码
┌──────────────────────┐                 ┌──────────────────────┐
│ entry_point = 0x9000 │──►跳板           │ 0x7000: add 的机器码 │
└──────────────────────┘                  └──────────────────────┘
                                              ▲
backup 的 ArtMethod                          │
┌──────────────────────┐                     │
│ entry_point = 0x7000 │────────────────────┘  ← backup 还指着它
└──────────────────────┘
method_code_map_ 表：{ 0x7000 → add }   ← 表还写"属于 add"，但 add 不要了 → GC 会回收 → backup 野指针！

【迁移成功（MoveJitInfo 把表改掉）】
method_code_map_ 表：{ 0x7000 → backup }  ← 过户给 backup
backup.entry_point = 0x7000               ← 和表一致 → GC 不回收 → 安全

【迁移失败（主动放弃用机器码）】
backup.entry_point = 解释器入口           ← 不再指向 0x7000，改走解释器读字节码
backup.entry_point_from_jni = nullptr     ← 清空，防止还藏旧机器码引用
0x7000 机器码：没人引用了，GC 爱收不收，backup 不用了。
```

**关键区分（迁移失败清空 vs GC 清理）**：

| 方式 | 谁动手 | 时机 | 结果 |
|------|--------|------|------|
| GC 回收（被动） | GC | 不可控 | backup 还在用 0x7000 就被删 → 野指针崩溃 |
| 迁移失败清空（主动） | 我们的代码 | hook 时先下手 | 先让 backup 不依赖 0x7000，GC 后面爱收不收 → 安全 |

一句话：**迁移成功 = 把机器码在 `method_code_map_` 表里的归属从 add 过户给 backup；迁移失败 = 让 backup 改走解释器，主动放弃机器码，宁可慢也不碰会变野指针的内存。**

> 补充：405 行的二次条件 `!is_inline_hook && !is_native`——inline 模式（机器码被覆盖了、走 backup 的 trampoline）或 native 方法（入口是 jni 函数地址，不是机器码，GC 不回收）即使迁移失败也不清空，直接走 else 设入口。

> 关于"字节码会不会被 GC 破坏"：字节码归"类"管（backup 的 declaring_class 还引用着类，类不卸载字节码就在），机器码 GC（JitCodeCache）只回收"JIT 编译出来的机器码"，**管不着 dex 里的字节码**。所以迁移失败走解释器是安全的。

#### 第 4 件：打日志（433-445）

纯调试，跳过。

---

### 五、AfterHook 详解（art_method.cpp:448-519）

```cpp
void ArtMethod::AfterHook(bool is_inline_hook, bool is_native, bool is_proxy) {
    uint32_t access_flags = GetAccessFlags();

    if (Android::version >= Android::kN) {                 // ≥N 禁止被 JIT 再编译
        if (Android::version >= Android::kR) access_flags &= ~kAccPreCompiled;
        access_flags |= kAccCompileDontBother;
    }
    if (Android::version >= Android::kO && !is_inline_hook) {
        if (UNLIKELY(PineConfig::debuggable && !(is_native || is_proxy))) {
            access_flags |= AccessFlags::kNative;          // debug 模式防解释器忽略 entry_point
        }
    }
    if (Android::version >= Android::kQ) {
        access_flags &= ~AccessFlags::kFastInterpreterToInterpreterInvoke;  // 清 fast 解释器缓存标志
    }

    bool hasNativeFlags = (access_flags & AccessFlags::kNative) != 0;
    if (UNLIKELY(hasNativeFlags)) {
        // FastNative/CriticalNative 执行时 GC 被禁用，可能死锁，hook 方法不能带这两个标志
        access_flags &= ~AccessFlags::kFastNative;
        if (Android::version >= Android::kP) access_flags &= ~AccessFlags::kCriticalNative;
    }

    SetAccessFlags(access_flags);

    if (art_interpreter_to_compiled_code_bridge) {
        SetEntryPointFromInterpreter(art_interpreter_to_compiled_code_bridge);  // 设解释器入口
    }
}
```

**核心作用**：防止 ART 的 JIT/内联/解释器把 hook 绕过或还原。

- `kAccCompileDontBother`：给 JIT 打"别动我"标记，防它重编译 target 把入口改回。
- `kFastNative`/`kCriticalNative` 清除（**当前分支新增**）：这两个标志会让方法在执行时禁用 GC，hook 方法不能带（否则可能死锁）。
- 设解释器入口为 `interpreter→compiled bridge`，保证走解释器路径时也能正确进 compiled 代码。

---

### 六、完整走查：`calculator.add(1, 2)` 一次调用

hook 完成后，三个方法的状态：

| 角色 | 是什么 | entry_point 指向 | 谁调它 |
|------|--------|-----------------|--------|
| target (add) | 你 hook 的原方法 | `0x9000`（跳板） | 业务代码照常调 add |
| bridge | 新入口（拦截） | 自己的 body | 跳板拐进来的 |
| backup | 原方法替身 | `0x7000`（原代码） | bridge 的 handleCall 里调 |

```
1. calculator.add(1,2)
2. 读 target.entry_point → 是跳板（不是原逻辑了）
3. 跳板 → 拐进 bridge
4. bridge body 执行：handleCall(artTargetMethod, this, [1, 2])
5. handleCall 里：
     a. 查 HookRecord（artTargetMethod 反查）→ 找到你的 MethodHook
     b. 调 beforeCall（可改参数，比如 a 改成 10）
     c. 调 invokeOriginalMethod → 实际调 backup
            ↓
        backup.entry_point → 0x7000 原机器码 → 执行 a+b → 返回（10+2=12）  ← 原逻辑在这跑
     d. 调 afterCall（可改返回值）
6. handleCall 返回最终结果
7. calculator.add(1,2) 拿到结果
```

---

### 七、收口记忆点

| # | 记忆点 | 一句话 |
|---|--------|--------|
| 1 | hook 本质 | 改指针，不改代码——机器码/字节码都是原方法那一份，backup 靠指针复用 |
| 2 | 偏移 vs 值 | init0 量"刻度"（偏移），hook 用刻度读写"读数"（某方法的入口值） |
| 3 | 三个方法分工 | target 被截走、bridge 拦截分发、backup 保存原逻辑 |
| 4 | 三核心动作 | InstallReplacementTrampoline 改入口、BackupFrom 复制+迁 JIT、AfterHook 防 JIT |
| 5 | JIT 迁移 | 机器码在 method_code_map_ 表里的归属从 add 过户给 backup；失败就走解释器 |
| 6 | copy_from 拷什么 | 只拷 ArtMethod 结构体（户口本），不拷机器码/字节码 |

**一句话概括整课**：REPLACEMENT hook 就是"改 add 的入口指针指向跳板（InstallReplacementTrampoline）→ 把 add 的户口本复制给 backup 并迁移 JIT 归属（BackupFrom）→ 给 add 打防 JIT 标记（AfterHook）"，原逻辑从头到尾没动过，backup 靠指针指向原机器码执行。

---
