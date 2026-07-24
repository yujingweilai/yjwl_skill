---
name: "lvgl-helper"
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
.trae/skills/lvgl-helper/templates/
├── ui_common_interface.md      # UI 公共接口模板
├── button_event_template.md    # 按钮事件模板
├── image_render_template.md    # 图片渲染模板
├── font_usage_template.md      # 字体使用模板
├── state_machine_template.md   # 状态机模板
├── page_navigation_template.md # 页面导航模板
└── custom_module_template.md   # 自定义模块模板
```

**注意：** 开发者需要根据实际项目自行创建和维护这些模板文件，AI 在编写代码时会引用这些模板。

#### 4.2 模板内容示例

**开发者应该在模板文件中提供：**

1. **UI 公共接口模板** (`ui_common_interface.md`)
   - 页面创建函数签名
   - 页面销毁函数签名
   - 页面刷新函数签名
   - 数据传递接口

2. **按钮事件模板** (`button_event_template.md`)
   - 按钮创建标准代码
   - 事件回调函数模板
   - 样式应用方式

3. **图片渲染模板** (`image_render_template.md`)
   - 图片资源引用方式
   - 图片控件创建方法
   - 图片更新接口

4. **字体使用模板** (`font_usage_template.md`)
   - 字体声明方式
   - 字体应用方法
   - 多语言字体处理

5. **状态机模板** (`state_machine_template.md`)
   - 页面状态定义
   - 状态切换逻辑
   - 状态处理函数

6. **自定义模块模板** (`custom_module_template.md`)
   - 项目特有的公共模块
   - 自定义控件封装
   - 业务逻辑接口

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
