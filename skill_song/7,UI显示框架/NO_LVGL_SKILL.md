---
name: "ui-state-machine-framework-non-lvgl"
description: "Designs C UI state-machine frameworks for non-LVGL displays. Invoke when building, refactoring, or reviewing embedded UI state routing."
---

# UI State Machine Framework

## 适用场景

当用户要求设计、补充、重构或审查嵌入式 UI 状态机框架时使用本 skill。适用于：

- 非 LVGL 的 TFT/LCD/OLED 页面状态机
- 裸机轮询 UI 框架
- 按键、旋钮、触摸输入驱动的页面切换
- 基于 `ui_context_t` 的页面数据传递
- 基于 `ui_update_display()` 或类似函数的页面渲染路由

## 项目参考来源

本 skill 的默认框架思路参考以下 LBOL 工程文件中的现有模式：

- 推荐的 UI 分层、状态机、页面管理、渲染接口结构
- 比如 `ui_update_display(ui_context_t *ctx)` 的 switch-case 页面路由方式
- 比如 `ui_context_t`、`ui_state_t`、页面数据结构定义

处理其他工程时，应先读取目标工程已有 UI 头文件、渲染入口、事件入口和页面文件，不能直接套模板覆盖原有架构。

## 核心设计原则

1. 状态机只管理状态和事件，不直接写复杂绘图逻辑。
2. 页面渲染只读取 `ui_context_t` 或等价上下文，不直接决定复杂业务流程。
3. 输入事件先转换为统一 UI 事件，再交给状态机处理。
4. 页面跳转集中管理，避免页面之间互相随意调用。
5. 最小改动优先，保留工程已有命名、目录、初始化流程和主循环方式。
6. 禁止使用 `goto`。
7. 能复用现有函数时必须复用，不为一次性逻辑创建多余抽象。

## 推荐分层

推荐拆成 5 层，但可以按目标工程规模裁剪：

1. 输入层
   - 按键、编码器、触摸、串口命令等硬件输入
   - 只负责采集原始输入，不直接切页面

2. UI 事件层
   - 将硬件输入转换为统一事件，例如 `UI_EVT_UP`、`UI_EVT_DOWN`、`UI_EVT_OK`、`UI_EVT_BACK`
   - 可用队列、环形缓冲或简单变量实现

3. 状态机层
   - 根据当前页面状态和事件决定是否切页、移动焦点、刷新数据
   - 典型接口：`ui_state_machine_dispatch(ctx, event, param)`

4. 页面管理层
   - 管理当前页、上一页、父页面、页面栈
   - 典型接口：`ui_page_switch()`、`ui_page_push()`、`ui_page_pop()`

5. 渲染层
   - 根据 `ctx->current_state` 调用对应页面绘制函数
   - 调用 LCD/TFT/OLED 绘制 API

## 通用数据结构建议

优先沿用工程已有结构。如果需要补充，可按以下方向扩展：

```c
typedef enum {
    UI_EVT_NONE = 0,
    UI_EVT_UP,
    UI_EVT_DOWN,
    UI_EVT_LEFT,
    UI_EVT_RIGHT,
    UI_EVT_OK,
    UI_EVT_BACK,
    UI_EVT_REFRESH,
    UI_EVT_TIMEOUT,
    UI_EVT_DATA_CHANGED,
    UI_EVT_MAX
} ui_event_t;
```

```c
#define UI_PAGE_STACK_DEPTH 8

typedef struct {
    uint8_t indicate;
    uint8_t sub_indicate;
    uint8_t selected_item;
    uint16_t selected_sub_item;

    ui_state_t current_state;
    ui_state_t previous_state;
    ui_state_t parent_state;

    ui_state_t page_stack[UI_PAGE_STACK_DEPTH];
    uint8_t stack_top;

    uint8_t need_refresh;
} ui_context_base_t;
```

如目标工程已经有 `ui_context_t`，不要重新定义一个平行上下文，优先在原结构中小范围增加字段。

## 函数注释规范

每个新增或修改的函数都必须有中文注释，至少说明：

- 函数名称
- 参数含义
- 返回值含义
- 函数作用

推荐格式：

```c
/**
 * @brief UI状态机事件分发函数
 * @param ctx UI上下文指针，用于读取当前状态并更新页面状态
 * @param event UI事件类型，例如确认、返回、上下左右等
 * @param param 事件附加参数，无参数时填0
 * @return true表示事件已处理，false表示参数无效或事件未处理
 * @note 作用：根据当前页面状态和输入事件执行页面跳转、焦点移动或刷新请求。
 */
bool ui_state_machine_dispatch(ui_context_t *ctx, ui_event_t event, uint32_t param);
```

## 日志规范

新增日志必须使用英文，便于串口调试和自动化过滤。

推荐示例：

```c
printf("UI event: state=%d event=%d param=%lu\r\n", ctx->current_state, event, param);
printf("UI switch: from=%d to=%d\r\n", old_state, new_state);
printf("UI render: state=%d indicate=%u sub=%u selected=%u\r\n",
       ctx->current_state, ctx->indicate, ctx->sub_indicate, ctx->selected_item);
```

日志不要过密。优先放在事件入口、页面切换、异常参数、渲染入口这些关键位置。

## 渲染路由规范

对于已有 `ui_update_display(ui_context_t *ctx)` 的工程，优先保留 switch-case 路由方式。建议每个 case 直接返回页面绘制结果：

```c
bool ui_update_display(ui_context_t *ctx)
{
    if (ctx == NULL) {
        printf("UI render failed: ctx is NULL\r\n");
        return false;
    }

    printf("UI render: state=%d indicate=%u sub=%u selected=%u\r\n",
           ctx->current_state, ctx->indicate, ctx->sub_indicate, ctx->selected_item);

    switch (ctx->current_state) {
        case UI_STATE_MAIN_MENU:
            return ui_draw_main_menu(ctx);

        case UI_STATE_LABEL_PREVIEW:
            return ui_draw_label_preview(ctx);

        default:
            printf("UI render skipped: unknown state=%d\r\n", ctx->current_state);
            return false;
    }
}
```

若现有页面绘制函数返回 `void`，不要为了模板强行大范围改签名；可以先保持原函数签名，让 `ui_update_display()` 在调用后返回 `true`。

## 适配方式

本 skill 适用于非 LVGL 的普通 TFT/LCD/OLED 裸机显示框架。

页面函数内部可以：

- 清屏或局部擦除
- 绘制文本、图标、框线、位图
- 根据 `ctx->indicate` 和 `ctx->selected_item` 做全量或局部刷新
- 调用已有 LCD/TFT/OLED 绘制函数
- 使用 `ui_update_display()` 的 switch-case 作为页面路由

框架约束：

- 页面绘制函数应复用现有 LCD/TFT/OLED 底层函数。
- 不要把按键判断写进页面绘制函数，按键应先转为统一 UI 事件。
- 局部刷新要避免引入未初始化坐标、未知缓存或不存在的绘图函数。
- 如果现有页面函数返回 `void`，不要为了套模板强行改成 `bool`，除非用户明确要求统一接口。
- 普通框架优先使用 `enum + switch`，只有页面数量很多或跳转关系复杂时再考虑页面表或函数指针表。

## 状态机接口骨架

可根据工程需要放入 `ui_state_machine.h/.c`，也可以合并到已有 `ui_event.c` 或 `ui_interface.c`，以最小改动为准。

```c
/**
 * @brief UI状态机初始化函数
 * @param ctx UI上下文指针，用于初始化当前状态和选择项
 * @return true表示初始化成功，false表示参数无效
 * @note 作用：设置默认页面、清空选择状态，并请求首次刷新。
 */
bool ui_state_machine_init(ui_context_t *ctx);

/**
 * @brief UI状态机事件分发函数
 * @param ctx UI上下文指针，用于读取当前状态和写入新状态
 * @param event UI事件类型
 * @param param 事件附加参数
 * @return true表示事件已处理，false表示参数无效或事件未处理
 * @note 作用：统一处理页面切换、焦点移动和刷新请求。
 */
bool ui_state_machine_dispatch(ui_context_t *ctx, ui_event_t event, uint32_t param);

/**
 * @brief UI页面切换函数
 * @param ctx UI上下文指针，用于保存旧状态并设置新状态
 * @param new_state 目标页面状态
 * @return true表示切换成功，false表示参数无效
 * @note 作用：集中完成页面状态切换和刷新标志设置。
 */
bool ui_state_machine_switch(ui_context_t *ctx, ui_state_t new_state);
```

## 页面跳转处理建议

页面跳转应优先集中在状态机中。例如：

```c
switch (ctx->current_state) {
    case UI_STATE_MAIN_MENU:
        if (event == UI_EVT_OK) {
            return ui_state_machine_switch(ctx, UI_STATE_LABEL_PREVIEW);
        }
        break;

    case UI_STATE_LABEL_PREVIEW:
        if (event == UI_EVT_BACK) {
            return ui_state_machine_switch(ctx, UI_STATE_MAIN_MENU);
        }
        break;

    default:
        break;
}
```

复杂页面可以拆出页面专属事件处理函数，例如：

```c
static bool ui_state_handle_main_menu(ui_context_t *ctx, ui_event_t event, uint32_t param);
static bool ui_state_handle_custom(ui_context_t *ctx, ui_event_t event, uint32_t param);
```

但只有在单个 dispatch 函数明显过长时才拆分。

## 页面命名与拼写

处理已有工程时不要贸然全局改名。比如工程中已有 `UI_STATE_SETING`、`ui_draw_seting()`、`ui_seting_data_t`，除非用户明确要求统一拼写，否则继续沿用，避免引入大范围编译错误。

新工程可使用标准拼写：

- `UI_STATE_SETTING`
- `ui_draw_setting()`
- `ui_setting_data_t`

## UI 公共接口规范（强制要求）

**所有 UI 模块必须通过公共接口与外界交互，其他组件不能直接调用页面绘制函数。**

### 公共接口设计原则

1. **单一入口**：所有显示更新都通过 `ui_update_display()` 一个函数进入
2. **上下文传递**：所有数据都通过 `ui_context_t` 结构体传递，不依赖全局变量
3. **状态驱动**：根据 `current_state` 决定显示哪个页面，外部不需要知道具体页面函数
4. **数据载荷**：每个页面的数据独立封装在 `ui_context_t` 中，页面间数据隔离

### 公共接口头文件示例

```c
/******************* UI 接口头文件 *******************/
#ifndef __ui_interface_H__
#define __ui_interface_H__

#include <stdint.h>
#include <stdbool.h>

/* ========== 页面数据载荷结构体 ========== */
/* 大白话：每个页面需要显示的数据都封装在这里，页面之间互不干扰 */

/* 预览页面数据 */
typedef struct {
    uint8_t position;            /* 显示位置位掩码：bit0-第1行，bit1-第2行... */
    char l_type[25];             /* 标签类型/日期（示例："COOKED 01 JUN 2025"） */
    char l_product_name[25];     /* 商品名称（示例："Chicken"），空字符串表示未设置 */
    char l_details[25];          /* 详细信息（示例："Spicy, contains nuts"） */
    char l_usage_tip[25];        /* 使用提示（示例："Use within 3 days"） */
    uint8_t l_print_page;        /* 打印纸张数，最小1张最多10张 */
} ui_preview_data_t;

/* 设置页面数据 */
typedef struct {
    uint8_t position;            /* 显示位置位掩码 */
    char *l_date_set_day;        /* 显示设置天 */
    char *l_date_set_month;      /* 显示设置月 */
    char *l_date_set_year;       /* 显示设置年 */
    char *l_time_set_hour;       /* 显示设置时 */
    char *l_time_set_minute;     /* 显示设置分 */
    char *l_time_set_ampm;       /* 显示设置AMPM */
    char *l_bat_data;            /* 电池电量显示 */
} ui_seting_data_t;

/* ========== UI 状态定义 ========== */
/* 大白话：每个值代表一个页面，状态机靠这个知道当前在哪个页面 */
typedef enum {
    UI_STATE_IDLE = 0,           /* 空闲状态 */
    UI_STATE_MAIN_MENU,          /* 主菜单 */
    UI_STATE_LABEL_PREVIEW,      /* 标签预览渲染 */
    UI_STATE_DATE,               /* 日期设置 */
    UI_STATE_SETING,             /* 设置页面 */
    UI_STATE_MAX                 /* 页面总数，用于数组大小定义 */
} ui_state_t;

/* ========== UI 全局上下文结构体 ========== */
/* 大白话：这是整个 UI 框架的"记忆中枢"，所有页面状态和数据都记在这里 */
typedef struct {
    /* 页面状态数据 */
    uint8_t indicate;            /* 页面显示状态（0=不显示，1=初始化，2=选择/动态） */
    uint8_t sub_indicate;        /* 子页面显示状态 */
    uint8_t selected_item;       /* 选中的元素：页面上哪个 UI 区块需要更新 */
    uint16_t selected_sub_item;  /* 选中的子元素：指向哪个子页面进行渲染 */
    ui_state_t current_state;    /* 当前 UI 状态：指向要渲染的页面 */
    
    /* 各页面数据载荷 */
    ui_preview_data_t preview_data;      /* 传递到预览页面的数据 */
    ui_seting_data_t seting_data;        /* 传递到设置页面的数据 */
    
    uint8_t need_refresh;        /* 刷新标志：1=需要重新画画面，0=不用画 */
} ui_context_t;

/* ========== 公共接口函数 ========== */
/**
 * @brief 根据当前状态刷新屏幕
 * @param ctx UI 上下文结构体指针
 * @return 成功返回 true；ctx 为 NULL 返回 false
 * @note 作用：这是 UI 模块的唯一入口，其他组件只需要调用这个函数
 *       内部会根据 current_state 自动调用对应页面的绘制函数
 */
bool ui_update_display(ui_context_t *ctx);

#endif /* __ui_interface_H__ */
```

### 公共接口实现示例

```c
/* ========== 显示更新函数实现 ========== */
/**
 * @brief 根据当前状态刷新屏幕
 * @param ctx UI 上下文结构体指针
 * @return 成功返回 true；ctx 为 NULL 返回 false
 * @note 作用：这是 UI 模块的唯一入口
 *       其他组件（比如按键处理、业务逻辑）只需要：
 *       1. 修改 ui_context_t 里的数据
 *       2. 设置 need_refresh = 1
 *       3. 调用 ui_update_display()
 *       不需要知道具体页面函数叫什么
 */
bool ui_update_display(ui_context_t *ctx)
{
    if (ctx == NULL) {
        printf("UI render failed: ctx is NULL\r\n");
        return false;
    }

    /* 不需要刷新就直接返回 */
    if (ctx->need_refresh == 0) {
        return true;
    }

    printf("UI render: state=%d indicate=%u sub=%u selected=%u\r\n",
           ctx->current_state, ctx->indicate, ctx->sub_indicate, ctx->selected_item);

    /* 根据当前状态调用对应页面绘制函数 */
    switch (ctx->current_state) {
        case UI_STATE_MAIN_MENU:
            ui_draw_main_menu(ctx);
            break;
        case UI_STATE_LABEL_PREVIEW:
            ui_draw_label_preview(ctx);
            break;
        case UI_STATE_DATE:
            ui_draw_date(ctx);
            break;
        case UI_STATE_SETING:
            ui_draw_seting(ctx);
            break;
        default:
            printf("UI render skipped: unknown state=%d\r\n", ctx->current_state);
            break;
    }

    /* 画完了，清除刷新标志 */
    ctx->need_refresh = 0;
    return true;
}
```

### 其他组件如何调用 UI

```c
/* 大白话：其他组件（比如按键处理、传感器数据更新）只需要这样调用 UI： */

/* 步骤 1：修改数据 */
ui_ctx.preview_data.l_print_page = 5;
strcpy(ui_ctx.preview_data.l_type, "COOKED 01 JUN 2025");

/* 步骤 2：标记需要刷新 */
ui_ctx.need_refresh = 1;

/* 步骤 3：调用公共接口 */
ui_update_display(&ui_ctx);

/* 不需要知道当前在哪个页面，也不需要直接调用 ui_draw_xxx() */
/* ui_update_display() 内部会自动根据 current_state 调用正确的页面函数 */
```

### 公共接口的好处

1. **解耦**：其他组件不需要知道 UI 内部有哪些页面、页面函数叫什么
2. **安全**：所有数据都通过 `ui_context_t` 传递，不会出现野指针
3. **统一**：所有显示更新都走同一个入口，方便调试和追踪
4. **易扩展**：新增页面只需要在 `ui_update_display()` 里加一个 case，外部调用代码不用改

### 公共接口约束

- **禁止**其他组件直接调用 `ui_draw_xxx()` 页面绘制函数
- **禁止**绕过 `ui_update_display()` 直接操作 LCD/TFT
- **必须**通过 `ui_context_t` 传递数据，不要依赖全局变量
- **必须**在修改数据后设置 `need_refresh = 1`，否则画面不会更新

---

## UI 框架使用指南（大白话版）

### 一、框架到底是干什么的？

**一句话：让嵌入式 UI 代码不乱。**

没有框架时，按键处理、页面切换、画面绘制全混在一起，页面一多就改不动。
有了框架后：
- **按键** → 翻译成统一事件
- **状态机** → 决定下一步干什么
- **页面函数** → 只管画画面
- 各司其职，互不干扰

### 二、框架组件一览

| 组件 | 干什么的 | 你需要做什么 |
|------|---------|-------------|
| `ui_event_t` 枚举 | 定义所有可能的 UI 事件 | 根据硬件输入增减事件类型 |
| `ui_state_t` 枚举 | 定义所有页面 | 每新增一个页面就加一项 |
| `ui_context_t` 结构体 | 存所有 UI 状态和数据 | 一般不用改，除非需要存新数据 |
| `ui_state_machine_init()` | 初始化 UI 状态 | 开机时调用一次 |
| `ui_state_machine_dispatch()` | 处理 UI 事件 | 每次有按键事件时调用 |
| `ui_state_machine_switch()` | 切换页面 | 在 dispatch 里调用，不要直接改状态 |
| `ui_update_display()` | 刷新画面 | 主循环里每次都调用 |
| `ui_draw_xxx()` | 画具体页面 | 每个页面写一个，在 update_display 里注册 |

### 三、页面如何注册到框架中

**大白话：就是告诉框架"多了一个页面"，让框架能找到它、画它、切到它。**

注册一个新页面需要 4 步，缺一不可：

**第 1 步：在 `ui_state_t` 枚举里加一个新状态**

```c
typedef enum {
    UI_STATE_IDLE = 0,
    UI_STATE_MAIN_MENU,
    UI_STATE_LABEL_PREVIEW,
    UI_STATE_NEW_PAGE,     /* ← 新增：你的新页面 */
    UI_STATE_MAX           /* 这个永远放最后，不要动 */
} ui_state_t;
```

> 大白话：枚举里每一项就是一个页面。加一项 = 多一个页面。`UI_STATE_MAX` 是计数器，永远放最后。

**第 2 步：写一个页面绘制函数**

```c
/**
 * @brief 画新页面
 * @param ctx UI 上下文指针，里面存着当前页面状态和要显示的数据
 * @return true 画好了，false 画失败了
 * @note 作用：在屏幕上画出新页面的内容
 */
bool ui_draw_new_page(ui_context_t *ctx)
{
    /* 清屏或局部擦除 */
    LCD_Clear(WHITE);

    /* 画文本、图标等 */
    LCD_ShowString(10, 10, "New Page", BLACK);

    /* 根据 ctx 里的数据画动态内容 */
    LCD_ShowString(10, 30, ctx->preview_data.l_type, BLACK);

    return true;
}
```

> 大白话：这个函数就是你的页面"长什么样"。框架会在需要时调用它来画画面。

**第 3 步：在 `ui_update_display()` 的 switch 里加一个 case**

```c
bool ui_update_display(ui_context_t *ctx)
{
    switch (ctx->current_state) {
        case UI_STATE_MAIN_MENU:
            return ui_draw_main_menu(ctx);
        case UI_STATE_LABEL_PREVIEW:
            return ui_draw_label_preview(ctx);
        case UI_STATE_NEW_PAGE:        /* ← 新增这一行 */
            return ui_draw_new_page(ctx);  /* ← 新增这一行 */
        default:
            return false;
    }
}
```

> 大白话：这步是告诉框架"当状态是 NEW_PAGE 时，去调用 ui_draw_new_page() 来画"。不加这步，框架不知道你的页面函数在哪。

**第 4 步：在 `ui_state_machine_dispatch()` 里加跳转逻辑**

```c
/* 在 dispatch 的 switch 里加 */
case UI_STATE_MAIN_MENU:
    if (event == UI_EVT_OK && ctx->selected_item == 2) {
        return ui_state_machine_switch(ctx, UI_STATE_NEW_PAGE);
    }
    break;

case UI_STATE_NEW_PAGE:
    if (event == UI_EVT_BACK) {
        return ui_state_machine_switch(ctx, UI_STATE_MAIN_MENU);
    }
    break;
```

> 大白话：这步是告诉框架"什么时候能进入这个页面、什么时候能从这个页面出来"。不加这步，你的页面永远进不去。

**4 步做完，新页面就注册好了！**

### 四、如何设定起始页面（开机第一个页面）

**大白话：开机后用户看到的第一个页面，由 `ui_state_machine_init()` 决定。**

```c
bool ui_state_machine_init(ui_context_t *ctx)
{
    /* ↓↓↓ 这一行决定开机停在哪个页面 ↓↓↓ */
    ctx->current_state = UI_STATE_MAIN_MENU;  /* 开机默认在主菜单 */

    ctx->previous_state = UI_STATE_MAIN_MENU;
    ctx->parent_state = UI_STATE_MAIN_MENU;

    ctx->indicate = 0;
    ctx->sub_indicate = 0;
    ctx->selected_item = 0;
    ctx->selected_sub_item = 0;
    ctx->stack_top = 0;

    /* 第一次需要画画面 */
    ctx->need_refresh = 1;

    return true;
}
```

> 大白话：想把开机页面改成别的，只需要把 `UI_STATE_MAIN_MENU` 改成你想要的页面状态。比如改成 `UI_STATE_LABEL_PREVIEW`，开机就直接进预览页。

**如果想让某个页面成为起始页面：**
1. 确保该页面已在 `ui_state_t` 枚举中定义
2. 确保该页面的绘制函数已在 `ui_update_display()` 中注册
3. 把 `ui_state_machine_init()` 里的 `ctx->current_state` 改成该页面状态

### 五、如何注册多个页面

**大白话：每个页面都走同样的 4 步流程，注册多少次都行。**

假设你有 5 个页面：

```c
typedef enum {
    UI_STATE_IDLE = 0,        /* 空闲 */
    UI_STATE_MAIN_MENU,       /* 主菜单 */
    UI_STATE_PREVIEW,         /* 预览 */
    UI_STATE_SETTING,         /* 设置 */
    UI_STATE_DATE,            /* 日期 */
    UI_STATE_ABOUT,           /* 关于 */
    UI_STATE_MAX              /* 总数 = 6 */
} ui_state_t;
```

每个页面都要做：

| 步骤 | 做什么 | 举例 |
|------|--------|------|
| 1. 枚举加状态 | 在 `ui_state_t` 里加一项 | `UI_STATE_ABOUT` |
| 2. 写绘制函数 | `bool ui_draw_about(ctx)` | 画"关于"页面内容 |
| 3. 渲染路由加 case | `ui_update_display()` 里加 case | `case UI_STATE_ABOUT: return ui_draw_about(ctx);` |
| 4. 状态机加跳转 | `dispatch()` 里加跳转逻辑 | 从主菜单按 OK 进入"关于" |

> 大白话：就是"重复 4 步"。框架不限制页面数量，你加多少个都行。

### 六、页面内部如何创建才能接入框架

**大白话：页面绘制函数有固定格式，照着写就能接入。**

```c
/**
 * @brief 画 XXX 页面
 * @param ctx UI 上下文指针
 * @return true 画好了
 * @note 作用：画出 XXX 页面的内容
 */
bool ui_draw_xxx(ui_context_t *ctx)
{
    /* ===== 第 1 部分：清屏或局部擦除 ===== */
    LCD_Clear(WHITE);  /* 或者只擦局部区域 */

    /* ===== 第 2 部分：画固定内容（标题、框线等） ===== */
    LCD_ShowString(10, 10, "Title", BLACK);
    LCD_DrawLine(0, 25, 240, 25, BLACK);

    /* ===== 第 3 部分：画动态内容（从 ctx 读数据） ===== */
    LCD_ShowString(10, 40, ctx->xxx_data.name, BLACK);

    /* ===== 第 4 部分：画选中高亮（根据 selected_item） ===== */
    if (ctx->selected_item == 0) {
        LCD_ShowString(5, 40, ">", BLACK);  /* 选中项前面画个箭头 */
    }

    return true;
}
```

**关键规则：**
- 页面函数只负责"画"，不要在里面写按键判断
- 数据从 `ctx` 里读，不要直接读全局变量
- 选中状态看 `ctx->selected_item`，框架会帮你更新这个值
- 返回 `true` 表示画好了，返回 `false` 表示画失败了

### 七、按键和状态机如何接入

**大白话：按键 → 翻译成事件 → 交给状态机处理。**

#### 7.1 按键接入流程

```
用户按了一个键
    ↓
按键驱动检测到按键（GPIO 中断 / 轮询扫描）
    ↓
把按键码投递到消息队列（或直接调用翻译函数）
    ↓
翻译成统一事件（UI_EVT_UP / UI_EVT_DOWN / UI_EVT_OK 等）
    ↓
调用 ui_state_machine_dispatch(ctx, event, param)
    ↓
状态机根据当前页面 + 事件，决定下一步干什么
```

#### 7.2 按键翻译示例

```c
/**
 * @brief 把硬件按键码翻译成 UI 事件
 * @param key_code 硬件按键码（比如 GPIO 读到的值）
 * @return 对应的 UI 事件
 * @note 作用：把各种硬件按键统一翻译成状态机能认识的事件
 */
static ui_event_t ui_key_to_event(uint16_t key_code)
{
    switch (key_code) {
        case KEY_UP:    return UI_EVT_UP;     /* 上键 → 向上事件 */
        case KEY_DOWN:  return UI_EVT_DOWN;   /* 下键 → 向下事件 */
        case KEY_OK:    return UI_EVT_OK;     /* 确认键 → 确认事件 */
        case KEY_BACK:  return UI_EVT_BACK;   /* 返回键 → 返回事件 */
        default:        return UI_EVT_NONE;   /* 不认识的键 → 无事件 */
    }
}
```

#### 7.3 主循环里怎么接

```c
/* 裸机主循环 */
int main(void)
{
    Hardware_Init();
    ui_state_machine_init(&ui_ctx);

    while (1) {
        /* 第 1 步：扫描按键 */
        uint16_t key = Key_Scan();
        if (key != KEY_NONE) {
            ui_event_t evt = ui_key_to_event(key);
            ui_state_machine_dispatch(&ui_ctx, evt, 0);
        }

        /* 第 2 步：刷新画面 */
        if (ui_ctx.need_refresh) {
            ui_update_display(&ui_ctx);
        }
    }
}
```

#### 7.4 状态机怎么处理事件

状态机收到事件后，根据"当前在哪个页面"和"什么事件"两个条件来决定干什么：

```c
bool ui_state_machine_dispatch(ui_context_t *ctx, ui_event_t event, uint32_t param)
{
    switch (ctx->current_state) {

        case UI_STATE_MAIN_MENU:
            /* 在主菜单页面时： */
            if (event == UI_EVT_UP) {
                /* 按了上键 → 焦点上移 */
                if (ctx->selected_item > 0) {
                    ctx->selected_item--;
                    ctx->need_refresh = 1;  /* 标记需要重画 */
                }
            } else if (event == UI_EVT_DOWN) {
                /* 按了下键 → 焦点下移 */
                if (ctx->selected_item < 2) {
                    ctx->selected_item++;
                    ctx->need_refresh = 1;
                }
            } else if (event == UI_EVT_OK) {
                /* 按了确认 → 进入选中项对应的页面 */
                switch (ctx->selected_item) {
                    case 0: return ui_state_machine_switch(ctx, UI_STATE_PREVIEW);
                    case 1: return ui_state_machine_switch(ctx, UI_STATE_SETTING);
                }
            }
            break;

        case UI_STATE_PREVIEW:
            /* 在预览页面时： */
            if (event == UI_EVT_BACK) {
                /* 按了返回 → 回到主菜单 */
                return ui_state_machine_switch(ctx, UI_STATE_MAIN_MENU);
            }
            break;
    }
    return true;
}
```

### 八、数据如何传递

**大白话：所有数据都放在 `ui_context_t` 里，页面从 ctx 里读数据来画。**

#### 8.1 定义页面数据

```c
/* 每个页面的数据单独定义一个结构体 */
typedef struct {
    char name[25];        /* 商品名称 */
    char date[25];        /* 日期 */
    uint8_t count;        /* 数量 */
} ui_preview_data_t;

typedef struct {
    uint8_t hour;         /* 小时 */
    uint8_t minute;       /* 分钟 */
    uint8_t brightness;   /* 亮度 */
} ui_setting_data_t;

/* 所有页面数据都放在 ui_context_t 里 */
typedef struct {
    uint8_t selected_item;
    ui_state_t current_state;
    uint8_t need_refresh;

    ui_preview_data_t preview_data;   /* 预览页的数据 */
    ui_setting_data_t setting_data;   /* 设置页的数据 */
} ui_context_t;
```

#### 8.2 往页面传数据

```c
/* 其他组件（比如传感器任务、按键任务）往 ctx 里写数据 */
strcpy(ui_ctx.preview_data.name, "Chicken");
strcpy(ui_ctx.preview_data.date, "01 JUN 2025");
ui_ctx.preview_data.count = 5;

/* 写完数据后，标记需要刷新 */
ui_ctx.need_refresh = 1;
```

#### 8.3 页面里读数据

```c
bool ui_draw_preview(ui_context_t *ctx)
{
    LCD_Clear(WHITE);
    LCD_ShowString(10, 10, ctx->preview_data.name, BLACK);   /* 读名称 */
    LCD_ShowString(10, 30, ctx->preview_data.date, BLACK);   /* 读日期 */

    char buf[10];
    sprintf(buf, "Count: %d", ctx->preview_data.count);
    LCD_ShowString(10, 50, buf, BLACK);                       /* 读数量 */

    return true;
}
```

> 大白话：数据传递就三步——写数据到 ctx、设 need_refresh=1、框架自动调用绘制函数来画。

### 九、页面如何刷新

**大白话：框架不会自动刷新，必须有人"喊一声"才会画。**

#### 9.1 刷新触发方式

| 触发方式 | 代码 | 场景 |
|---------|------|------|
| 状态机切页面 | `ui_state_machine_switch()` 内部自动设 `need_refresh = 1` | 切页面时自动刷新 |
| 移动焦点 | 在 dispatch 里手动设 `ctx->need_refresh = 1` | 上下移动选中项时刷新 |
| 数据更新 | 其他任务写完数据后设 `ctx->need_refresh = 1` | 传感器数据变了要刷新显示 |
| 定时器 | 定时器回调里设 `ctx->need_refresh = 1` | 定时刷新时钟等 |

#### 9.2 刷新流程

```
某处设置了 ctx->need_refresh = 1
    ↓
主循环检测到 need_refresh == 1
    ↓
调用 ui_update_display(&ui_ctx)
    ↓
根据 ctx->current_state 找到对应的 ui_draw_xxx()
    ↓
ui_draw_xxx() 画画面
    ↓
画完后 ctx->need_refresh = 0（框架自动清除）
```

#### 9.3 关键代码

```c
/* 主循环里 */
while (1) {
    /* ... 按键处理 ... */

    /* 只有 need_refresh == 1 时才画，省 CPU 时间 */
    if (ui_ctx.need_refresh) {
        ui_update_display(&ui_ctx);
    }
}
```

> 大白话：`need_refresh` 就像一个 flag，有人改了数据或切了页面就把它设成 1，主循环看到 1 就去画，画完自动清零。没设 1 就不画，省时间。

### 十、关键参数说明

#### 10.1 `ui_context_t` 关键字段

| 字段 | 类型 | 大白话解释 |
|------|------|-----------|
| `current_state` | `ui_state_t` | 当前在哪个页面，框架靠这个决定画什么 |
| `previous_state` | `ui_state_t` | 上一个页面，用于"返回"功能 |
| `parent_state` | `ui_state_t` | 父页面，用于层级返回 |
| `selected_item` | `uint8_t` | 当前选中第几项，页面函数靠这个画高亮 |
| `selected_sub_item` | `uint16_t` | 当前选中子项，用于子菜单 |
| `indicate` | `uint8_t` | 页面显示状态（0=不显示，1=初始化，2=选择/动态） |
| `sub_indicate` | `uint8_t` | 子页面显示状态 |
| `need_refresh` | `uint8_t` | 刷新标志（1=要画，0=不用画） |
| `page_stack[]` | `ui_state_t[]` | 页面栈，记录访问过的页面，支持多级返回 |
| `stack_top` | `uint8_t` | 页面栈的栈顶，表示栈里有多少个页面 |

#### 10.2 `ui_event_t` 事件类型

| 事件 | 大白话解释 | 典型触发 |
|------|-----------|---------|
| `UI_EVT_NONE` | 没有事件 | 占位用 |
| `UI_EVT_UP` | 向上 | 按了上键 / 旋钮往上转 |
| `UI_EVT_DOWN` | 向下 | 按了下键 / 旋钮往下转 |
| `UI_EVT_LEFT` | 向左 | 按了左键 |
| `UI_EVT_RIGHT` | 向右 | 按了右键 |
| `UI_EVT_OK` | 确认 | 按了确认键 / 回车键 |
| `UI_EVT_BACK` | 返回 | 按了返回键 / 退出键 |
| `UI_EVT_REFRESH` | 刷新 | 需要重新画画面 |
| `UI_EVT_TIMEOUT` | 超时 | 长时间没操作 |
| `UI_EVT_DATA_CHANGED` | 数据变了 | 业务数据更新 |

#### 10.3 核心函数参数

```c
/* 状态机初始化：开机时调用一次 */
bool ui_state_machine_init(ui_context_t *ctx);
/* ctx：UI 上下文指针，一般传 &ui_ctx */

/* 事件分发：每次有按键事件时调用 */
bool ui_state_machine_dispatch(ui_context_t *ctx, ui_event_t event, uint32_t param);
/* ctx：UI 上下文指针 */
/* event：翻译后的 UI 事件（比如 UI_EVT_OK） */
/* param：附加参数（比如旋钮转了多少格），没有就填 0 */

/* 页面切换：在 dispatch 里调用 */
bool ui_state_machine_switch(ui_context_t *ctx, ui_state_t new_state);
/* ctx：UI 上下文指针 */
/* new_state：要切到的目标页面状态 */

/* 画面刷新：主循环里调用 */
bool ui_update_display(ui_context_t *ctx);
/* ctx：UI 上下文指针 */
```

### 十一、扩展接口函数使用

#### 11.1 页面栈操作（用于多级返回）

```c
/**
 * @brief 页面入栈（进入子页面时调用）
 * @param ctx UI 上下文指针
 * @param state 要入栈的页面状态
 * @return true 入栈成功，false 栈满了
 * @note 作用：记住当前页面，方便以后返回
 */
bool ui_page_push(ui_context_t *ctx, ui_state_t state)
{
    if (ctx->stack_top >= UI_PAGE_STACK_DEPTH) {
        printf("UI page stack overflow\r\n");
        return false;
    }
    ctx->page_stack[ctx->stack_top] = state;
    ctx->stack_top++;
    return true;
}

/**
 * @brief 页面出栈（返回上一级时调用）
 * @param ctx UI 上下文指针
 * @return 出栈后的页面状态，栈空时返回 UI_STATE_IDLE
 * @note 作用：取出上一个页面，切回去
 */
ui_state_t ui_page_pop(ui_context_t *ctx)
{
    if (ctx->stack_top == 0) {
        return UI_STATE_IDLE;
    }
    ctx->stack_top--;
    return ctx->page_stack[ctx->stack_top];
}
```

**使用示例：**

```c
/* 从主菜单进入设置页面 */
case UI_STATE_MAIN_MENU:
    if (event == UI_EVT_OK) {
        ui_page_push(ctx, UI_STATE_MAIN_MENU);  /* 记住主菜单 */
        return ui_state_machine_switch(ctx, UI_STATE_SETTING);
    }
    break;

/* 从设置页面返回 */
case UI_STATE_SETTING:
    if (event == UI_EVT_BACK) {
        ui_state_t prev = ui_page_pop(ctx);     /* 取出上一级 */
        return ui_state_machine_switch(ctx, prev);
    }
    break;
```

#### 11.2 自定义事件处理函数

当单个页面的事件处理逻辑很复杂时，可以拆出独立函数：

```c
/**
 * @brief 主菜单页面事件处理
 * @param ctx UI 上下文指针
 * @param event UI 事件
 * @param param 事件附加参数
 * @return true 事件已处理
 * @note 作用：把主菜单的事件处理逻辑单独拿出来，让 dispatch 更清晰
 */
static bool ui_handle_main_menu(ui_context_t *ctx, ui_event_t event, uint32_t param)
{
    if (event == UI_EVT_UP && ctx->selected_item > 0) {
        ctx->selected_item--;
        ctx->need_refresh = 1;
    } else if (event == UI_EVT_DOWN && ctx->selected_item < 2) {
        ctx->selected_item++;
        ctx->need_refresh = 1;
    } else if (event == UI_EVT_OK) {
        switch (ctx->selected_item) {
            case 0: return ui_state_machine_switch(ctx, UI_STATE_PREVIEW);
            case 1: return ui_state_machine_switch(ctx, UI_STATE_SETTING);
        }
    }
    return true;
}

/* dispatch 里调用 */
case UI_STATE_MAIN_MENU:
    return ui_handle_main_menu(ctx, event, param);
```

#### 11.3 数据更新接口

```c
/**
 * @brief 更新预览页面数据
 * @param ctx UI 上下文指针
 * @param name 商品名称
 * @param date 日期
 * @param count 数量
 * @return true 更新成功
 * @note 作用：其他组件通过这个函数更新预览页数据，不要直接操作 ctx
 */
bool ui_update_preview_data(ui_context_t *ctx, const char *name,
                            const char *date, uint8_t count)
{
    if (ctx == NULL || name == NULL) {
        return false;
    }
    strncpy(ctx->preview_data.name, name, sizeof(ctx->preview_data.name) - 1);
    strncpy(ctx->preview_data.date, date, sizeof(ctx->preview_data.date) - 1);
    ctx->preview_data.count = count;
    ctx->need_refresh = 1;  /* 数据变了，标记刷新 */
    return true;
}
```

### 十二、完整接入流程图

```
┌──────────┐    按键/旋钮     ┌──────────┐
│  硬件输入 │ ──────────────→ │ 消息队列  │
└──────────┘                 └────┬─────┘
                                  │
                                  ↓
                          ┌──────────────┐
                          │   主循环      │
                          └──────┬───────┘
                                 │
                    ┌────────────┼────────────┐
                    ↓            ↓            ↓
              扫描按键       翻译事件       调用状态机
                    │            │            │
                    └────────────┼────────────┘
                                 ↓
                    ┌─────────────────────────┐
                    │ ui_state_machine_dispatch │
                    │  决定：切页/移焦点/刷新    │
                    └────────────┬────────────┘
                                 ↓
                    ┌─────────────────────────┐
                    │ ui_state_machine_switch   │
                    │  更新 current_state       │
                    │  设 need_refresh = 1      │
                    └────────────┬────────────┘
                                 ↓
                    ┌─────────────────────────┐
                    │ ui_update_display()       │
                    │  根据 current_state       │
                    │  调用 ui_draw_xxx()       │
                    └────────────┬────────────┘
                                 ↓
                    ┌─────────────────────────┐
                    │ LCD/TFT/OLED 刷新完成     │
                    │ need_refresh = 0          │
                    └─────────────────────────┘
```

### 十三、常见操作速查表

| 我想... | 怎么做 |
|---------|--------|
| 新增一个页面 | 枚举加状态 → 写绘制函数 → update_display 加 case → dispatch 加跳转 |
| 改开机起始页 | 改 `ui_state_machine_init()` 里的 `ctx->current_state` |
| 从 A 页跳到 B 页 | 在 dispatch 里 A 的 case 下调用 `ui_state_machine_switch(ctx, UI_STATE_B)` |
| 加一个新按键 | `ui_key_to_event()` 加 case → dispatch 里加处理逻辑 |
| 往页面传数据 | 写数据到 `ctx->xxx_data` → 设 `need_refresh = 1` |
| 刷新当前页面 | 设 `ctx->need_refresh = 1`，主循环会自动画 |
| 返回上一级 | dispatch 里调 `ui_state_machine_switch(ctx, ctx->previous_state)` 或用页面栈 |
| 移动选中项 | dispatch 里改 `ctx->selected_item` + 设 `need_refresh = 1` |

---

## 修改前检查清单

动手改代码前必须确认：

- 已读取当前 UI 头文件，确认 `ui_state_t` 和 `ui_context_t` 的真实字段
- 已读取当前渲染入口，确认页面绘制函数名称和返回值类型
- 已确认主循环或事件入口在哪里
- 已确认已有日志风格和串口输出方式

## 修改后复核清单

完成修改后必须复核：

- 没有引入未声明变量
- 没有调用不存在的函数
- 没有破坏已有枚举名称
- 没有把 `SETING` 和 `SETTING` 混用导致编译错误
- 没有改变用户未要求改变的程序框架
- 没有使用 `goto`
- 新增函数都有中文注释
- 新增日志为英文
- `NULL` 参数路径有明确返回值
- 页面绘制函数返回值与调用方式匹配

## 输出给用户的总结要求

完成任务后，用大白话总结：

1. 改了哪些文件
2. 每个文件主要改了什么
3. 为什么这样改
4. 有没有引入新变量或新函数
5. 复核时重点看了什么
6. 是否运行了编译或检查命令，结果如何
7. 有没有引入新的库或头文件
8. 生成一个UI界面使用指南及其流程框架的.md文件，**必须包含以上"UI 框架使用指南（大白话版）"中的所有核心章节**（页面注册4步流程、起始页设定、多页面注册、按键接入、数据传递、刷新机制、关键参数说明、扩展接口、流程图、速查表等），不能省略

总结要直接、具体，不要只说“已优化”。
