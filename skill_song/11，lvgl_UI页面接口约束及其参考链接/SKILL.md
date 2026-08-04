---
name: "lvgl-约束及其参考"
description: "LVGL UI开发辅助工具，提供代码规范、公共接口参考和文档查询。Invoke when writing LVGL UI code, creating pages, or when unsure about LVGL API usage."
---

# LVGL Helper

LVGL UI 开发辅助技能，提供统一的代码规范、公共接口参考和官方文档查询支持。

## 核心规则

### 1. 页面切换与局部更新（强制）

1. 每次设计或修改页面切换前，**必须先询问用户**旧页面对象处理策略，只能由用户在“立即删除、延后删除、不删除常驻”三选一；用户未明确时不得自行决定。
2. 同一页面刷新时，禁止通过 `lv_obj_clean` 或重建整页实现；必须持久保存已创建控件句柄，仅更新实际变化对象。例如 A/B/C 三项中只有 B 变化时，只允许操作 B。
3. 所有局部更新仍必须沿用 `app_arch → app_ui → page draw` 链路，并在既有 `app_ui_render_current_page()` 所持 LVGL lock 内执行；禁止其他任务直接操作 LVGL 对象。
4. 本项目用户已选择“**立即删除**”：切入新页面时清除旧页面对象；页面再次进入时必须在读取任何旧句柄前清零该页句柄并完整重建，避免悬空指针。

### 2. 版本识别（强制）

**在开始任何 LVGL 开发任务前，必须明确当前使用的 LVGL 版本：**

- 如果项目中有明确的版本标识（如 `lv_conf.h` 中的版本号、CMakeLists.txt 配置等），自动识别版本
- 如果无法确定版本，**必须询问用户**当前使用的是 LVGL 8.3 还是 LVGL 9.5
- 不同版本的 API 差异较大，必须使用对应版本的正确语法

**版本判断方法：**
```c
// 查看 lv_conf.h 或 lv_version.h
#include "lvgl.h"
#if LV_VERSION_MAJOR == 9
    // LVGL 9.x 语法
#elif LV_VERSION_MAJOR == 8
    // LVGL 8.x 语法
#endif
```

### 2. 官方文档参考（强制）

**当遇到以下情况时，必须参考官方文档：**
- 不确定某个 API 的正确用法
- 不知道某个功能的实现边界
- 需要了解特定控件的详细参数
- 遇到版本差异问题

**参考链接：**
- LVGL 9.5 文档：https://lvgl.io/docs/open/9.5/
- LVGL 8.3 文档：https://lvgl.io/docs/open/8.3/

**使用方式：**
1. 优先查阅对应版本的官方文档
2. 确认 API 的参数、返回值、使用限制
3. 参考官方示例代码

### 3. 项目公共接口参考（强制）

**编写 LVGL 代码时，必须参考项目中的现有公共接口，保持代码风格统一：**

#### 3.1 必须参考的公共模块

- **UI 公共接口文件**：查找项目中的 `ui_*.c/h` 或 `gui_*.c/h` 文件
- **按钮事件处理**：参考现有的按钮点击、长按、按住等事件处理函数
- **触摸注册接口**：查找触摸回调注册方式
- **页面数据传输**：参考页面间数据传递的公共方法
- **页面统一入口**：查找页面初始化、加载、卸载的统一接口
- **刷新函数**：参考 UI 刷新、数据更新的公共函数
- **数据传递值**：查找全局数据、页面参数的传递方式

#### 3.2 代码风格要求

**所有页面必须保持统一风格：**

```c
// ✅ 正确示例：统一的页面结构
static void page_xxx_create(lv_obj_t *parent)
{
    // 1. 创建容器
    lv_obj_t *container = lv_obj_create(parent);
    lv_obj_set_size(container, LV_PCT(100), LV_PCT(100));
    
    // 2. 创建控件（参考公共接口）
    lv_obj_t *btn = lv_btn_create(container);
    lv_obj_add_event_cb(btn, btn_event_handler, LV_EVENT_CLICKED, NULL);
    
    // 3. 设置样式（使用统一样式）
    lv_obj_add_style(btn, &style_btn_primary, 0);
}

// ✅ 正确示例：统一的事件处理
static void btn_event_handler(lv_event_t *e)
{
    lv_event_code_t code = lv_event_get_code(e);
    if (code == LV_EVENT_CLICKED) {
        // 使用公共数据传递接口
        page_data_send(PAGE_ID_XXX, data);
    }
}
```

### 3.3 页面对象生命周期、局部刷新和安全接口（强制执行）

**以下规则为强制规则，编写、修改、审查任何 LVGL 页面时必须执行，不能省略。**

#### 3.3.1 页面切换前必须确认旧页面处理策略

每次新增或修改页面切换逻辑前，必须先询问用户旧页面对象的处理方式；用户没有明确选择时，禁止自行决定：

1. **立即删除**：进入新页面时删除旧页面全部对象，返回旧页面时重新创建。
2. **延后删除**：切页后暂时保留旧页面对象，必须明确删除触发时机和对象有效期。
3. **常驻不删除**：页面对象一直保留，通过隐藏和显示切换；必须评估 RAM 占用和后台定时器。

本项目当前已经确认的策略是：**真正切入新页面时，立即删除旧页面对象；重新进入旧页面时重新创建该页面对象。**

#### 3.3.2 页面创建、同页局部更新和重新进入规则

```text
第一次进入页面：
    创建 A、B、C、D、F 等全部页面对象。
    页面 .c 文件静态保存每个需要动态更新对象的句柄。

同一个页面更新：
    判断实际变化的是哪一个对象。
    只操作对应对象句柄。
    例如 A、B、C、D、F 中 D 的文字、数值、颜色、位置或显隐变化时，只更新 D。

按钮选择状态切换：
    只更新原高亮按钮为非高亮，以及新选择按钮为高亮。
    不更新其他未变化按钮和静态对象。

收到新数据更新文本框：
    只更新对应文本框。
    不删除、不重建页面内其他对象。

切换到其他页面：
    按用户确认策略处理旧页面；本项目当前为立即删除旧页面对象。
    删除后立即清空该页面 .c 文件静态保存的对象句柄，防止悬空指针。

重新回到页面：
    重新创建 A、B、C、D、F 等页面对象。
    保存新的对象句柄。
```

**同一页面严禁**使用 `lv_obj_clean()`、`lv_obj_del()` 后重建整页来处理按键高亮、文本更新、数值更新或样式更新。页面中没有变化的对象不能删除、不能重新创建、也不应重复设置。

#### 3.3.3 页面对象句柄和外部接口规则

1. `lv_obj_t *` 及公共控件句柄必须只在对应页面 `.c` 文件中用 `static` 保存和管理。
2. **严禁**在页面 `.h` 文件中使用 `extern lv_obj_t *` 或 `extern` 公共控件句柄把页面对象暴露给外部模块。
3. 页面 `.h` 文件只允许声明必要的页面操作接口函数，例如“请求更新 D 文本”“请求删除 D 对象”“请求更新按钮选中状态”。
4. 外部模块不能直接调用页面对象更新或删除函数，更不能直接调用 `lv_label_set_text()`、`lv_obj_del()` 等 LVGL API；业务模块必须通过 `app_arch → app_ui → page draw` 链路发起请求。
5. 页面内部在 `draw()` 或受统一渲染入口调用的局部更新函数中，根据上下文执行对象创建、更新、隐藏、显示或删除；全部 LVGL API 必须处于已有 `lvgl_port_lock()` 保护范围。
6. 页面对象删除后，页面必须立即将对应静态句柄置为 `NULL`；任何局部更新前必须确保当前页面仍有效且对象句柄有效。

大白话：外部可以请求“更新 D”或“删除 D”，但外部不能拿到 D 的 `lv_obj_t *` 自己操作。页面自己保管对象，统一渲染链路负责拿锁后执行，这样不会因为跨任务、切页后旧指针失效而崩溃。

### 4. 强制性参考页面（自定义模板）

**开发者需要在 skill 目录内维护参考模板文件，AI 必须参考这些模板：**

#### 4.1 模板文件位置

模板文件位于本 skill 目录下的 `templates/` 文件夹：

```
├── D:\code\LBOL\code\new_git\lbol_esp32\LBOL\components\app_arch\app_arch.c     # 程序架构模板，外部传输数据给UI的入口点
├── D:\code\LBOL\code\new_git\lbol_esp32\LBOL\components\app_arch\include\app_arch_config.h   # 程序架构配置模板
├── D:\code\LBOL\code\new_git\lbol_esp32\LBOL\components\ui_common\imge_common.c     # 图片显示及其一些形状框 公共模块模板
├── D:\code\LBOL\code\new_git\lbol_esp32\LBOL\components\ui_common\font_common.c     # 字体使用公共模块模板
├── D:\code\LBOL\code\new_git\lbol_esp32\LBOL\components\ui_common\ui_common.c       # UI公共模块模板
├── D:\code\LBOL\code\new_git\lbol_esp32\LBOL\components\app_ui\include\app_ui.h      # 应用UI公共接口模板
├── D:\code\LBOL\code\new_git\lbol_esp32\LBOL\components\app_ui\app_ui.c              # 应用UI公共接口模板
├── D:\code\LBOL\code\new_git\lbol_esp32\LBOL\components\app_arch\include\app_arch.h   # 应用架构公共接口模板
└── D:\code\LBOL\code\new_git\lbol_esp32\LBOL\components\roboto_font\roboto_bold_18.c     # Roboto 字体加粗18号公共模块模板
└── D:\code\LBOL\code\new_git\lbol_esp32\LBOL\components\roboto_font\roboto_bold_16.c     # Roboto 字体加粗16号公共模块模板
└── D:\code\LBOL\code\new_git\lbol_esp32\LBOL\components\roboto_font\roboto_regular_16.c  # Roboto 字体普通16号公共模块模板
└── D:\code\LBOL\code\new_git\lbol_esp32\LBOL\components\roboto_font\roboto_regular_14.c  # Roboto 字体普通14号公共模块模板
└── D:\code\LBOL\code\new_git\lbol_esp32\LBOL\components\roboto_font\roboto_bold_14.c     # Roboto 字体加粗14号公共模块模板
└── D:\code\LBOL\code\new_git\lbol_esp32\LBOL\components\roboto_font\roboto_italic_13.c  # Roboto 字体斜体13号公共模块模板
└── D:\code\LBOL\code\new_git\lbol_esp32\LBOL\components\roboto_font                       # Roboto 字体公共模块模板
└── D:\code\LBOL\code\new_git\lbol_esp32\LBOL\components\ui_imge                              # 图片显示
└── D:\code\LBOL\code\new_git\lbol_esp32\LBOL\components\app_ui\UI_FRAMEWORK_GUIDE.md        # UI框架指南模板
└── D:\code\LBOL\code\new_git\lbol_esp32\LBOL\docs\ARCH_GUIDE.md                            # 应用架构指南模板
└── D:\code\LBOL\code\new_git\lbol_esp32\LBOL\components\app_ui\ui_even_common.c            # 按钮事件公共接口公共模块模板
└── D:\code\LBOL\code\new_git\lbol_esp32\LBOL\components\ui_common\include\color_common.h          # 颜色定义模板
└── D:\code\LBOL\code\new_git\lbol_esp32\LBOL\components\ui_common\label_common.c             # 带边框的文本标签公共模块模板
└── D:\code\LBOL\code\new_git\lbol_esp32\LBOL\components\app_ui\include\app_ui_test.h          # 应用UI测试公共接口模板
└── D:\code\LBOL\code\new_git\lbol_esp32\LBOL\components\app_ui\app_ui_test.c              # 应用UI测试公共接口模板



### 4.2 项目固定外部数据传输链路（强制）

本项目已经具备 `app_arch → app_ui → page_ui` 的数据传递框架。后续凡是传感器、RTC、电池、通信或其他业务任务向 UI 更新数据，**必须优先复用以下链路，不允许新建平行 UI 更新通道**：

```text
业务任务（例如温湿度传感器）
    ↓ 只调用 app_arch.h 的公共接口
app_arch_patch_ui_page_data() / app_arch_switch_ui_page_with_data()
    ↓ app_arch.c 获取 s_ui_api_mutex，串行保护 UI 上下文访问
app_ui_patch_page_data() / app_ui_switch_page_with_data()
    ↓ 更新 app_ui_context_t.page_data，设置 need_refresh
app_ui_update_display() → app_ui_render_current_page()
    ↓ 获取 lvgl_port_lock() 后调用当前页面 draw()
page_ui 绘制函数
    ↓ 只从 const app_ui_context_t *ctx 读取页面数据并更新 LVGL 控件
LVGL / 显示驱动
```

#### 按键事件与传感器数据的固定处理流程（强制）

**外部按键事件必须遵循：**

```text
按键按下
    ↓
app_arch 负责串行保护和事件投递
    ↓
app_ui 判断当前页面和事件
    ↓
当前页面 draw() 仅更新实际变化的控件
```

按键事件不得由驱动、业务任务或中断直接操作页面对象和 LVGL API。按键仅用于改变 UI 事件、页面状态或选择值；同一页面内只允许更新旧选中对象、新选中对象及高亮框等实际变化控件，禁止删除或重建整页。

**传感器及其他外部数据必须遵循：**

```text
传感器任务
    ↓
app_arch_patch_ui_page_data()
    ↓
app_ui 更新当前页面数据并请求刷新
    ↓
当前页面 draw()
    ↓
仅更新对应的 b 对象或其他实际变化控件
```

传感器任务只负责提交数据，页面 `draw()` 负责读取 `ctx` 并更新控件。传感器任务不得直接调用 `lv_label_set_text()`、页面 `draw()` 或其他 LVGL API；不需要切页时不得为了显示新数据强制切页。

#### 强制调用规则

1. **业务任务只包含并调用 `app_arch.h`，不得直接调用 `app_ui.h` 的数据更新、切页、刷新接口。** `app_ui.h` 是 UI 内部框架接口；`app_arch.h` 才是业务层线程安全入口。
2. **禁止业务任务直接调用页面 `draw()` 函数、`lv_label_set_text()`、`lv_obj_create()` 或其他 LVGL API。** LVGL 控件只能由 `app_ui` 的统一渲染路径在 `lvgl_port_lock()` 保护下操作。
3. 同页面数据更新优先调用 `app_arch_patch_ui_page_data(APP_ARCH_UI_SRC_USER_API, &patch)`；只有需要切换目标页面并携带基础数据时，才调用 `app_arch_switch_ui_page_with_data()`。
4. 外部数据更新默认**不应强制切页**。如果温湿度页面当前未显示，只更新缓存数据；用户进入该页后，由页面 `draw()` 显示最新值。仅在明确业务要求“收到数据必须跳转页面”时才切页。
5. `app_ui_page_data_t.user_data` 当前只保存指针、不复制通用业务结构体。传入的数据必须是静态变量、全局变量或其他生命周期足够长的存储，**禁止传递任务局部变量、函数栈变量或即将释放的动态内存地址**。
6. `app_ui_update_label_preview()` 是已存在的特殊安全接口：它会复制标签预览数据到 UI 内部缓存。其他通用 `user_data` 场景不能假设有此复制行为。
7. **禁止在 ISR/中断回调中调用任何 `app_arch_*ui*` 接口。** ISR 只能使用 `xQueueSendFromISR()` 或任务通知把数据交给普通业务任务，再由该任务调用 `app_arch` 接口。
8. 高频传感器采样不能每次都立即刷新 UI。应按有效数据变化或 200~500 ms 等业务周期合并刷新，避免频繁重绘和占用 LVGL 锁。

#### 长期数据、页面交互数据与临时 `user_data` 的选择规则（强制）

1. **长期使用数据必须走 UI 公共接口。** 标签类型、RTC 日期、元素名字、详情、业务日期、用户配置等需要跨页面保留或被多个组件使用的数据，应使用 `app_ui_shared_data_t` 等明确类型，并通过 `app_arch` 公共接口提交给 `app_ui` 内部长期缓存。
2. **页面交互数据一般走 UI 公共接口。** 一个页面填写、选择或修改后还要被其他页面读取的数据，不能依赖旧页面对象或临时 `user_data`；必须复制到 `app_ui.c` 的静态业务缓存中，页面删除后数据仍然有效。
3. **临时数据才使用 `app_ui_page_data_t.user_data`。** 仅在一次临时页面交换中使用、允许随页面退出而失效、且不需要被其他页面或外围组件长期读取的数据，可以通过 `user_data` 传递。
4. `user_data` 只复制指针，不复制指针指向的内容。即使是临时数据，其有效期也必须覆盖目标页面完整使用期间；禁止传入函数局部变量、任务栈变量、临时字符串缓冲区或即将释放的动态内存地址。
5. 目标页面退出后若数据允许消失，应停止访问该 `user_data`，并在页面上下文复位时清除对应指针；禁止页面删除后继续保存或读取旧指针。
6. 如果无法确定数据是否会跨页面、是否需要长期保存，默认按长期数据处理并走公共接口，不能为了省一个接口而冒悬空指针风险。
7. 外围业务任务仍只能调用 `app_arch.h`；公共数据由 `app_arch` 使用既有 `s_ui_api_mutex` 串行提交，`app_ui` 按值复制，页面通过 `const` 指针只读使用。

```text
需要长期保存 / 多页面共用 / 页面之间继续交互
    → app_arch 公共接口
    → app_ui 完整复制到静态长期缓存
    → 页面通过 const 数据只读显示

只在当前临时页面使用 / 页面退出后允许消失
    → app_ui_page_data_t.user_data
    → 保证数据至少活到页面停止使用
    → 页面退出后清除引用，不再读取
```

大白话：以后还要用的数据，必须先存进 UI 的“公共保险箱”；只用这一次、页面关掉就可以不要的数据，才放进 `user_data` 临时传递。`user_data` 只是地址，不会帮忙复制内容，所以不能指向马上失效的局部变量。

#### 温湿度传输约定

温度、湿度属于关联数据，建议使用一个公共业务结构体统一保存；整数建议用放大 10 倍的定点值，例如 `253` 代表 `25.3℃`，`615` 代表 `61.5%RH`。如果数据需要持续刷新、跨页面保留或被多个组件读取，必须通过 `app_arch` 公共接口复制到 `app_ui` 长期缓存；只有一次临时页面显示且退出后允许丢失时，才可以通过 `user_data` 传递。页面 `draw()` 只读数据并更新控件，传感器任务不能直接修改 label。

大白话：`app_arch` 不是多余中转，它负责把多个业务任务的 UI 请求排队串行化；`app_ui` 负责保存页面数据和统一拿 LVGL 锁画界面。数据不是被重复复制两次，而是按职责逐层交给对应模块处理。

**注意：** 开发者需要根据实际项目自行创建和维护这些模板文件，AI 在编写代码时会引用这些模板。


#### 4.3 AI 参考规则

**AI 在编写 LVGL 代码时，必须：**

1. 先检查项目中是否存在 `lvgl_templates/` 目录
2. 如果存在，**强制阅读**相关模板文件
3. 严格按照模板中的代码风格和接口规范编写代码
4. 如果模板中没有覆盖的场景，参考官方文档和项目现有代码

### 5. 代码检查清单

**每次编写 LVGL 代码后，检查：**

- [ ] 是否确认了 LVGL 版本（8.3 或 9.5）
- [ ] 是否参考了项目中的公共接口
- [ ] 是否遵循了统一的代码风格
- [ ] 是否查看了强制性参考模板（如果存在）
- [ ] 不确定的 API 是否查阅了官方文档
- [ ] 事件处理是否使用了统一的回调格式
- [ ] 数据传递是否使用了公共接口
- [ ] 样式应用是否使用了统一的样式定义

## 使用示例

### 场景 1：创建新页面

```markdown
用户：帮我创建一个设置页面

AI 应该：
1. 确认 LVGL 版本（询问或自动识别）
2. 查找项目中的公共接口文件
3. 检查是否存在 lvgl_templates/ 目录
4. 参考现有页面的代码风格
5. 查阅官方文档确认 API 用法
6. 按照统一风格创建页面
```

### 场景 2：不确定的 API 用法

```markdown
用户：如何创建一个下拉列表

AI 应该：
1. 确认 LVGL 版本
2. 查阅对应版本的官方文档
3. 查看项目中是否有类似实现
4. 参考公共接口和模板
5. 提供正确的代码示例
```

## 注意事项

1. **版本差异**：LVGL 8.x 和 9.x 的 API 有显著差异，必须使用对应版本的语法
2. **性能考虑**：避免在循环中频繁创建/销毁控件，使用对象池或复用
3. **内存管理**：及时释放不使用的控件和样式
4. **代码复用**：优先使用项目中的公共接口，避免重复造轮子
5. **文档优先**：不确定的地方先查文档，不要猜测 API 用法