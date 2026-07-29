---
name: "lvgl-约束及其参考"
description: "LVGL UI开发辅助工具，提供代码规范、公共接口参考和文档查询。Invoke when writing LVGL UI code, creating pages, or when unsure about LVGL API usage."
---

# LVGL Helper

LVGL UI 开发辅助技能，提供统一的代码规范、公共接口参考和官方文档查询支持。

## 核心规则

### 1. 版本识别（强制）

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

#### 强制调用规则

1. **业务任务只包含并调用 `app_arch.h`，不得直接调用 `app_ui.h` 的数据更新、切页、刷新接口。** `app_ui.h` 是 UI 内部框架接口；`app_arch.h` 才是业务层线程安全入口。
2. **禁止业务任务直接调用页面 `draw()` 函数、`lv_label_set_text()`、`lv_obj_create()` 或其他 LVGL API。** LVGL 控件只能由 `app_ui` 的统一渲染路径在 `lvgl_port_lock()` 保护下操作。
3. 同页面数据更新优先调用 `app_arch_patch_ui_page_data(APP_ARCH_UI_SRC_USER_API, &patch)`；只有需要切换目标页面并携带基础数据时，才调用 `app_arch_switch_ui_page_with_data()`。
4. 外部数据更新默认**不应强制切页**。如果温湿度页面当前未显示，只更新缓存数据；用户进入该页后，由页面 `draw()` 显示最新值。仅在明确业务要求“收到数据必须跳转页面”时才切页。
5. `app_ui_page_data_t.user_data` 当前只保存指针、不复制通用业务结构体。传入的数据必须是静态变量、全局变量或其他生命周期足够长的存储，**禁止传递任务局部变量、函数栈变量或即将释放的动态内存地址**。
6. `app_ui_update_label_preview()` 是已存在的特殊安全接口：它会复制标签预览数据到 UI 内部缓存。其他通用 `user_data` 场景不能假设有此复制行为。
7. **禁止在 ISR/中断回调中调用任何 `app_arch_*ui*` 接口。** ISR 只能使用 `xQueueSendFromISR()` 或任务通知把数据交给普通业务任务，再由该任务调用 `app_arch` 接口。
8. 高频传感器采样不能每次都立即刷新 UI。应按有效数据变化或 200~500 ms 等业务周期合并刷新，避免频繁重绘和占用 LVGL 锁。

#### 温湿度传输约定

温度、湿度属于关联数据，优先使用一个长期有效的业务结构体经 `user_data` 传递；整数建议用放大 10 倍的定点值，例如 `253` 代表 `25.3℃`，`615` 代表 `61.5%RH`。页面 `draw()` 从 `ctx->page_data.user_data` 读取该结构体并显示，不能由传感器任务直接修改 label。

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