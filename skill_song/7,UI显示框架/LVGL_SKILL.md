---
name: "ui-state-machine-framework-lvgl"
description: "Designs C UI state-machine frameworks for LVGL displays. Invoke when building, refactoring, or reviewing embedded LVGL UI state routing."
---

# LVGL UI 状态机框架 Skill

## 适用场景

当用户要求设计、补充、重构或审查 **LVGL** UI 状态机框架时使用本 skill。适用于：

- LVGL 页面状态机
- RTOS 任务内 LVGL UI 事件分发框架
- 按键、旋钮、触摸输入驱动的 LVGL 页面切换
- 基于 `ui_context_t` 的 LVGL 页面数据传递
- 基于 `ui_update_display()` 或类似函数的 LVGL 页面渲染路由
- LVGL 触摸屏输入设备注册与事件处理
- LVGL 页面生命周期管理（create/enter/refresh/exit/destroy）
- ESP32 + LVGL 工程实践（线程安全、内存管理、双核分工）

**注意**：本 skill 仅适用于 LVGL 架构。如果用户使用非 LVGL 的裸机 TFT/LCD/OLED，请使用 `ui-state-machine-framework-non-lvgl` skill。

## 构建前询问流程（强制要求）

**开始构建 LVGL UI 框架之前，必须先向用户确认以下问题，不能直接套模板。**

### 第 1 问：输入方式是什么？

向用户确认：

> "你的输入设备是什么类型？触摸屏、物理按键、旋钮编码器、还是混合输入？"

- **触摸屏** → 需要进一步问第 2 问
- **物理按键 / 旋钮** → 默认走状态机方式（消息队列 → dispatch）
- **混合输入**（触摸屏 + 按键）→ 需要进一步问第 2 问

### 第 2 问（仅触摸屏时需要问）：是否注册到 LVGL？

向用户确认：

> "触摸屏输入是注册到 LVGL 中让 LVGL 直接处理触摸事件，还是不注册到 LVGL、由你自己的代码采集触摸坐标后走状态机处理？"

**方案 A：注册到 LVGL（LVGL 原生方式）**

- 把触摸屏驱动注册为 LVGL 输入设备（`lv_indev_t`）
- LVGL 自己处理触摸坐标、点击检测、控件交互
- 适合：大量 LVGL 控件交互（按钮点击、滑块拖动、列表滚动）
- 用户点击 LVGL 按钮 → LVGL 触发 `lv_event_cb` → 在回调里处理业务

**方案 B：不注册到 LVGL（状态机方式）**

- 触摸屏驱动由自己的代码管理（SPI/I2C 读取坐标）
- 自己判断触摸区域，翻译成 UI 事件
- 交给 `ui_state_machine_dispatch()` 统一处理
- 适合：自定义 UI 逻辑、不想依赖 LVGL 事件系统、需要统一按键+触摸逻辑

**方案 C：混合方式**

- 物理按键走状态机（消息队列 → dispatch）
- 触摸屏注册到 LVGL，LVGL 控件回调处理触摸交互
- LVGL 回调里也可以投递 UI 事件给状态机
- 适合：既有物理按键又有触摸屏的工程

**必须等用户回答后再开始写代码，不要替用户做决定。**

### 第 3 问：UI 框架内部是否已有接口函数？（强制询问）

向用户确认：

> "你的 UI 框架内部是否已经有接口函数？比如 `ui_partial_update()`（局部数据更新）、`ui_cross_page_update()`（跨页面数据更新）等？如果有的话，请把接口函数的声明或页面截图发给我，我会沿用你现有的接口设计。"

- **有接口函数** → 用户提供接口函数声明或截图后，必须沿用现有接口命名、参数风格和调用方式，不能重新造一套
- **没有接口函数** → 使用本 skill 提供的默认接口函数（见"局部数据更新接口"和"跨页面数据更新接口"章节）

**必须等用户回答后再开始写代码。**

### 第 5 问：外部传感器数据类型有哪些？（强制询问）

向用户确认：

> "你有哪些外部传感器或数据源需要更新到 UI 上？比如温度、湿度、电池电量、时间等？它们的数据类型是什么（整型、浮点、字符串）？"

- 根据用户回答，为每种传感器数据在 `ui_context_t` 中增加对应的数据字段
- 为每种传感器数据生成对应的局部更新函数

**必须等用户回答后再开始写代码。**

---

## 核心设计原则

1. 状态机只管理状态和事件，不直接写复杂绘图逻辑。
2. 页面渲染只读取 `ui_context_t` 或等价上下文，不直接决定复杂业务流程。
3. 输入事件先转换为统一 UI 事件，再交给状态机处理。
4. 页面跳转集中管理，避免页面之间互相随意调用。
5. 最小改动优先，保留工程已有命名、目录、初始化流程和主循环方式。
6. **禁止使用 `goto`。**
7. 能复用现有函数时必须复用，不为一次性逻辑创建多余抽象。
8. **所有函数、事件、状态、枚举、结构体都必须用中文大白话写注释，让人一看就懂。**
9. **函数调用层级、流转关系必须用中文大白话说明白。**
10. **RTOS 环境下，所有 LVGL 操作必须加锁，保证线程安全。**

---

## 推荐分层（5 层）

推荐拆成 5 层，但可以按目标工程规模裁剪：

```
┌─────────────────────────────────────────────┐
│  第 1 层：输入层                              │
│  按键、编码器、触摸、串口命令等硬件输入         │
│  只负责采集原始输入，不直接切页面               │
├─────────────────────────────────────────────┤
│  第 2 层：UI 事件层                           │
│  把硬件输入翻译成统一事件                      │
│  比如 UI_EVT_UP、UI_EVT_DOWN、UI_EVT_OK      │
│  可以用队列、环形缓冲或简单变量实现             │
├─────────────────────────────────────────────┤
│  第 3 层：状态机层                            │
│  根据当前页面状态 + 事件，决定                 │
│  要不要切页、移动焦点、刷新数据                │
│  核心函数：ui_state_machine_dispatch()        │
├─────────────────────────────────────────────┤
│  第 4 层：页面管理层                           │
│  管理当前页、上一页、父页面、页面栈             │
│  核心函数：ui_page_switch()                   │
├─────────────────────────────────────────────┤
│  第 5 层：渲染层                              │
│  根据 ctx->current_state 调用对应绘制函数      │
│  创建/更新 LVGL 控件                          │
└─────────────────────────────────────────────┘
```

---

## 函数调用流转关系（大白话说明）

下面用大白话讲清楚整个 UI 框架是怎么转起来的：

### 整体流程

```
用户按了一个键
    ↓
【第 1 层】按键驱动检测到按键，把原始数据丢进消息队列
    ↓
【第 2 层】UI 任务从队列里取出消息，翻译成统一事件（比如 UI_EVT_OK）
    ↓
【第 3 层】把事件交给状态机 ui_state_machine_dispatch()
    ↓
    状态机看看当前在哪个页面，根据事件决定：
    - 要不要切到别的页面？ → 调用 ui_state_machine_switch()
    - 要不要移动焦点？ → 改 ctx->selected_item
    - 要不要刷新显示？ → 设 ctx->need_refresh = 1
    ↓
【第 4 层】ui_state_machine_switch() 更新 ctx->current_state
    ↓
【第 5 层】主循环调用 ui_update_display()，根据 current_state
    调用对应页面的绘制函数，把画面画出来
```

### 函数调用层级图

```
主循环 / UI 任务
│
├── xQueueReceive() 或按键扫描
│   └── 取到消息后调用 ↓
│
├── ui_state_machine_dispatch(ctx, event, param)
│   │   作用：根据当前页面和事件，决定下一步干什么
│   │
│   ├── ui_state_machine_switch(ctx, new_state)
│   │       作用：切换页面状态，设置刷新标志
│   │
│   └── 也可能直接改 ctx->selected_item（移动焦点）
│       或设 ctx->need_refresh = 1（请求刷新）
│
├── ui_update_display(ctx)
│   │   作用：根据 current_state 调用对应页面绘制函数
│   │
│   ├── ui_draw_main_menu(ctx)
│   ├── ui_draw_label_preview(ctx)
│   ├── ui_draw_setting(ctx)
│   └── ... 其他页面绘制函数
│
└── lv_task_handler()（LVGL 工程）
        作用：让 LVGL 刷新屏幕、处理动画
```

### 页面切换流转示例

```
假设当前在【主菜单】页面，用户按了【确认】键：

1. 按键驱动 → 投递消息 UI_MSG_KEY_EVENT(KEY_OK)
2. UI 任务取出消息 → 翻译成 UI_EVT_OK
3. 调用 ui_state_machine_dispatch(ctx, UI_EVT_OK, 0)
4. 状态机发现当前是 UI_STATE_MAIN_MENU，收到 UI_EVT_OK
   → 调用 ui_state_machine_switch(ctx, UI_STATE_LABEL_PREVIEW)
5. switch 函数：
   - 保存旧状态 ctx->previous_state = UI_STATE_MAIN_MENU
   - 设置新状态 ctx->current_state = UI_STATE_LABEL_PREVIEW
   - 设置刷新标志 ctx->need_refresh = 1
6. 主循环调用 ui_update_display(ctx)
7. switch(ctx->current_state) 走到 UI_STATE_LABEL_PREVIEW
   → 调用 ui_draw_label_preview(ctx)
8. 画面更新完成
```

---

## 通用数据结构

优先沿用工程已有结构。如果需要补充，可按以下方向扩展：

```c
/*
 * UI 事件类型枚举
 * 大白话：定义所有可能的 UI 操作事件
 * 按键、旋钮、触摸等操作都会被翻译成这些事件
 */
typedef enum {
    UI_EVT_NONE = 0,       /* 没有事件，占位用 */
    UI_EVT_UP,             /* 向上：按了上键 / 旋钮往上转 */
    UI_EVT_DOWN,           /* 向下：按了下键 / 旋钮往下转 */
    UI_EVT_LEFT,           /* 向左：按了左键 */
    UI_EVT_RIGHT,          /* 向右：按了右键 */
    UI_EVT_OK,             /* 确认：按了确认键 / 回车键 */
    UI_EVT_BACK,           /* 返回：按了返回键 / 退出键 */
    UI_EVT_REFRESH,        /* 刷新：需要重新画画面 */
    UI_EVT_TIMEOUT,        /* 超时：长时间没操作，自动触发 */
    UI_EVT_DATA_CHANGED,   /* 数据变了：业务数据更新，需要刷新显示 */
    UI_EVT_MAX             /* 事件数量上限，用于数组大小定义 */
} ui_event_t;
```

```c
/*
 * UI 页面状态枚举
 * 大白话：每个值代表一个页面，状态机靠这个知道当前在哪个页面
 * 根据实际工程增减页面
 */
typedef enum {
    UI_STATE_MAIN_MENU = 0,     /* 主菜单页面 */
    UI_STATE_LABEL_PREVIEW,     /* 标签预览页面 */
    UI_STATE_SETTING,           /* 设置页面 */
    UI_STATE_MAX                /* 页面数量上限，用于数组大小定义 */
} ui_state_t;
```

```c
/*
 * UI 页面栈深度
 * 大白话：最多能记住几层页面，用于"返回上一级"功能
 */
#define UI_PAGE_STACK_DEPTH 8

/*
 * UI 上下文结构体
 * 大白话：这是整个 UI 框架的"记忆中枢"
 * 记录了当前在哪个页面、选了哪个选项、要不要刷新等所有状态信息
 * 所有页面绘制函数都从这里读数据来画画面
 */
typedef struct {
    uint8_t indicate;           /* 当前大分类索引，比如主菜单里选了第几项 */
    uint8_t sub_indicate;       /* 当前小分类索引，比如子菜单里选了第几项 */
    uint8_t selected_item;      /* 当前选中项，用于高亮显示 */
    uint16_t selected_sub_item; /* 当前选中子项 */

    ui_state_t current_state;   /* 当前在哪个页面 */
    ui_state_t previous_state;  /* 上一个页面，用于返回功能 */
    ui_state_t parent_state;    /* 父页面，用于层级返回 */

    /* 页面栈：记录访问过的页面，支持多级返回 */
    ui_state_t page_stack[UI_PAGE_STACK_DEPTH];
    uint8_t stack_top;          /* 栈顶指针，指向当前栈里有多少个页面 */

    uint8_t need_refresh;       /* 刷新标志：1=需要重新画画面，0=不用画 */
} ui_context_base_t;
```

如目标工程已经有 `ui_context_t`，不要重新定义一个平行上下文，优先在原结构中小范围增加字段。

---

## 注释规范（强制要求）

**所有函数、事件、状态、枚举值、结构体字段都必须用中文大白话写注释。**

注释要让人一看就懂，不要写废话。

### 函数注释格式

```c
/**
 * @brief UI 状态机事件分发函数
 * @param ctx   UI 上下文指针，里面存着当前在哪个页面、选了哪项等信息
 * @param event UI 事件类型，比如确认、返回、上下左右
 * @param param 事件附加参数，没有就填 0
 * @return true 表示事件处理完了，false 表示参数有问题或事件没处理
 * @note 作用：根据当前页面和按键事件，决定是切页面、移焦点还是刷新画面
 */
bool ui_state_machine_dispatch(ui_context_t *ctx, ui_event_t event, uint32_t param);
```

### 枚举注释格式

```c
typedef enum {
    UI_STATE_MAIN_MENU = 0,  /* 主菜单页面：开机后看到的第一个页面 */
    UI_STATE_LABEL_PREVIEW,  /* 标签预览页面：预览要打印的内容 */
    UI_STATE_SETTING,        /* 设置页面：调浓度、速度等参数 */
    UI_STATE_MAX             /* 页面总数，用来定义数组大小 */
} ui_state_t;
```

### 结构体注释格式

```c
/*
 * UI 上下文结构体
 * 大白话：整个 UI 的"大脑"，所有页面状态都记在这里
 */
typedef struct {
    uint8_t selected_item;    /* 当前选了第几项，用来高亮显示 */
    ui_state_t current_state; /* 当前在哪个页面 */
    uint8_t need_refresh;     /* 1=该刷新画面了，0=画面是最新的不用画 */
} ui_context_t;
```

---

## 日志规范

新增日志必须使用英文，便于串口调试和自动化过滤。

```c
printf("UI event: state=%d event=%d param=%lu\r\n", ctx->current_state, event, param);
printf("UI switch: from=%d to=%d\r\n", old_state, new_state);
printf("UI render: state=%d indicate=%u sub=%u selected=%u\r\n",
       ctx->current_state, ctx->indicate, ctx->sub_indicate, ctx->selected_item);
```

日志不要过密。优先放在事件入口、页面切换、异常参数、渲染入口这些关键位置。

---

## 状态机核心代码（带完整注释）

```c
/**
 * @brief UI 状态机初始化
 * @param ctx UI 上下文指针
 * @return true 初始化成功，false 参数有问题
 * @note 作用：开机时调用一次，把页面设成主菜单，清空选择状态
 */
bool ui_state_machine_init(ui_context_t *ctx)
{
    if (ctx == NULL) {
        printf("UI init failed: ctx is NULL\r\n");
        return false;
    }

    /* 开机默认停在主菜单 */
    ctx->current_state = UI_STATE_MAIN_MENU;
    ctx->previous_state = UI_STATE_MAIN_MENU;
    ctx->parent_state = UI_STATE_MAIN_MENU;

    /* 选中第一项 */
    ctx->indicate = 0;
    ctx->sub_indicate = 0;
    ctx->selected_item = 0;
    ctx->selected_sub_item = 0;

    /* 清空页面栈 */
    ctx->stack_top = 0;

    /* 第一次需要画画面 */
    ctx->need_refresh = 1;

    printf("UI init: state=%d\r\n", ctx->current_state);
    return true;
}

/**
 * @brief UI 页面切换函数
 * @param ctx       UI 上下文指针
 * @param new_state 要切到哪个页面
 * @return true 切换成功，false 参数有问题
 * @note 作用：把当前页面状态改成新的，并标记需要刷新画面
 *       所有页面跳转都必须走这个函数，不要直接改 ctx->current_state
 */
bool ui_state_machine_switch(ui_context_t *ctx, ui_state_t new_state)
{
    if (ctx == NULL) {
        printf("UI switch failed: ctx is NULL\r\n");
        return false;
    }

    if (new_state >= UI_STATE_MAX) {
        printf("UI switch failed: invalid state=%d\r\n", new_state);
        return false;
    }

    /* 记住旧状态，方便调试 */
    ui_state_t old_state = ctx->current_state;

    /* 保存旧页面到栈里（用于返回功能） */
    if (ctx->stack_top < UI_PAGE_STACK_DEPTH) {
        ctx->page_stack[ctx->stack_top] = ctx->current_state;
        ctx->stack_top++;
    }

    /* 更新状态 */
    ctx->previous_state = ctx->current_state;
    ctx->current_state = new_state;
    ctx->need_refresh = 1;

    /* 切页面时重置选择项到第一项 */
    ctx->selected_item = 0;
    ctx->selected_sub_item = 0;

    printf("UI switch: from=%d to=%d\r\n", old_state, new_state);
    return true;
}

/**
 * @brief UI 状态机事件分发（核心函数）
 * @param ctx   UI 上下文指针
 * @param event 用户操作翻译成的 UI 事件
 * @param param 事件附加参数（比如旋钮转了多少格）
 * @return true 事件处理完了，false 没处理
 * @note 作用：这是整个 UI 的"大脑"
 *       收到事件后，看看当前在哪个页面，决定下一步干什么
 *       - 可能要切到别的页面
 *       - 可能要移动选中项
 *       - 可能要刷新画面
 */
bool ui_state_machine_dispatch(ui_context_t *ctx, ui_event_t event, uint32_t param)
{
    if (ctx == NULL) {
        printf("UI dispatch failed: ctx is NULL\r\n");
        return false;
    }

    if (event <= UI_EVT_NONE || event >= UI_EVT_MAX) {
        printf("UI dispatch failed: invalid event=%d\r\n", event);
        return false;
    }

    printf("UI dispatch: state=%d event=%d param=%lu\r\n",
           ctx->current_state, event, param);

    switch (ctx->current_state) {

        case UI_STATE_MAIN_MENU:
            /* 主菜单页面：处理上下移动和确认进入 */
            if (event == UI_EVT_UP && ctx->selected_item > 0) {
                ctx->selected_item--;
                ctx->need_refresh = 1;
            } else if (event == UI_EVT_DOWN && ctx->selected_item < 2) {
                ctx->selected_item++;
                ctx->need_refresh = 1;
            } else if (event == UI_EVT_OK) {
                /* 按了确认，根据选中项进入不同页面 */
                switch (ctx->selected_item) {
                    case 0:
                        return ui_state_machine_switch(ctx, UI_STATE_LABEL_PREVIEW);
                    case 1:
                        return ui_state_machine_switch(ctx, UI_STATE_SETTING);
                    default:
                        break;
                }
            }
            break;

        case UI_STATE_LABEL_PREVIEW:
            /* 标签预览页面：按返回键回到主菜单 */
            if (event == UI_EVT_BACK) {
                return ui_state_machine_switch(ctx, UI_STATE_MAIN_MENU);
            }
            break;

        case UI_STATE_SETTING:
            /* 设置页面：按返回键回到主菜单 */
            if (event == UI_EVT_BACK) {
                return ui_state_machine_switch(ctx, UI_STATE_MAIN_MENU);
            }
            /* 上下键调整设置项 */
            if (event == UI_EVT_UP && ctx->selected_item > 0) {
                ctx->selected_item--;
                ctx->need_refresh = 1;
            } else if (event == UI_EVT_DOWN && ctx->selected_item < 3) {
                ctx->selected_item++;
                ctx->need_refresh = 1;
            }
            break;

        default:
            printf("UI dispatch: unknown state=%d\r\n", ctx->current_state);
            break;
    }

    return true;
}
```

---

## 渲染路由（带完整注释）

```c
/**
 * @brief UI 画面刷新总入口
 * @param ctx UI 上下文指针
 * @return true 画好了，false 参数有问题或状态不认识
 * @note 作用：根据当前页面状态，调用对应的绘制函数
 *       主循环里每次都调它，但只有 need_refresh=1 时才真正画
 */
bool ui_update_display(ui_context_t *ctx)
{
    if (ctx == NULL) {
        printf("UI render failed: ctx is NULL\r\n");
        return false;
    }

    /* 不需要刷新就直接返回，省时间 */
    if (ctx->need_refresh == 0) {
        return true;
    }

    printf("UI render: state=%d selected=%u\r\n",
           ctx->current_state, ctx->selected_item);

    bool result = false;

    switch (ctx->current_state) {
        case UI_STATE_MAIN_MENU:
            result = ui_draw_main_menu(ctx);
            break;

        case UI_STATE_LABEL_PREVIEW:
            result = ui_draw_label_preview(ctx);
            break;

        case UI_STATE_SETTING:
            result = ui_draw_setting(ctx);
            break;

        default:
            printf("UI render skipped: unknown state=%d\r\n", ctx->current_state);
            result = false;
            break;
    }

    /* 画完了，清除刷新标志 */
    if (result) {
        ctx->need_refresh = 0;
    }

    return result;
}
```

---

## 数据更新接口（强制要求）

**以下两个接口函数是强制性要求，所有 LVGL UI 框架必须实现。**

### 局部数据更新接口（同页面数据刷新）

**大白话：当外部传感器数据变化时，只需要更新当前页面上的某一块 LVGL 控件，不需要整个页面重画。**

```c
/**
 * @brief 局部数据更新函数（同页面刷新）
 * @param ctx UI 上下文指针
 * @param item_id 要更新的 UI 区块编号（比如 0=温度标签，1=湿度标签）
 * @param data 新数据指针（根据数据类型可以是整型、浮点、字符串等）
 * @param data_len 数据长度（字符串类型需要，整型/浮点可填 0）
 * @return true 更新成功，false 参数无效或当前页面不支持该区块
 * @note 作用：只刷新当前页面上指定的 LVGL 控件文本，避免整个页面重绘
 *       适用于传感器数据频繁变化的场景，比如温度、湿度、电量等
 *       调用后会自动设置 need_refresh = 1
 *       RTOS 环境下会自动加锁保护 LVGL 操作
 */
bool ui_partial_update(ui_context_t *ctx, uint8_t item_id, const void *data, uint16_t data_len);
```

**使用示例：**

```c
/* 示例 1：更新当前页面的温度标签（整型） */
int16_t temperature = 25;
ui_partial_update(&ui_ctx, 0, &temperature, 0);

/* 示例 2：更新当前页面的湿度标签（浮点） */
float humidity = 65.5f;
ui_partial_update(&ui_ctx, 1, &humidity, 0);

/* 示例 3：更新当前页面的商品名称标签（字符串） */
char name[] = "Chicken";
ui_partial_update(&ui_ctx, 2, name, sizeof(name));
```

**实现示例（LVGL 版本）：**

```c
/**
 * @brief 局部数据更新函数实现（LVGL 版本）
 * @param ctx UI 上下文指针
 * @param item_id 要更新的 UI 区块编号
 * @param data 新数据指针
 * @param data_len 数据长度
 * @return true 更新成功
 * @note 作用：根据当前页面状态和 item_id，只更新指定 LVGL 控件的文本
 */
bool ui_partial_update(ui_context_t *ctx, uint8_t item_id, const void *data, uint16_t data_len)
{
    if (ctx == NULL || data == NULL) {
        printf("UI partial update failed: invalid param\r\n");
        return false;
    }

    /* RTOS 环境：加锁保护 LVGL 操作 */
    xSemaphoreTakeRecursive(lvgl_mutex, portMAX_DELAY);

    /* 根据当前页面状态，更新对应的数据字段和 LVGL 控件 */
    switch (ctx->current_state) {
        case UI_STATE_LABEL_PREVIEW:
            /* 预览页面的局部更新 */
            switch (item_id) {
                case 0:  /* 温度标签 */
                    ctx->preview_data.temperature = *(int16_t *)data;
                    if (g_temp_label != NULL) {
                        char buf[16];
                        sprintf(buf, "%d C", ctx->preview_data.temperature);
                        lv_label_set_text(g_temp_label, buf);
                    }
                    break;
                case 1:  /* 湿度标签 */
                    ctx->preview_data.humidity = *(float *)data;
                    if (g_humidity_label != NULL) {
                        char buf[16];
                        sprintf(buf, "%.1f%%", ctx->preview_data.humidity);
                        lv_label_set_text(g_humidity_label, buf);
                    }
                    break;
                case 2:  /* 商品名称标签 */
                    strncpy(ctx->preview_data.l_product_name, (char *)data,
                            sizeof(ctx->preview_data.l_product_name) - 1);
                    if (g_name_label != NULL) {
                        lv_label_set_text(g_name_label, ctx->preview_data.l_product_name);
                    }
                    break;
                default:
                    printf("UI partial update: unknown item_id=%d\r\n", item_id);
                    xSemaphoreGiveRecursive(lvgl_mutex);
                    return false;
            }
            break;

        case UI_STATE_SETTING:
            /* 设置页面的局部更新 */
            switch (item_id) {
                case 0:  /* 电池电量标签 */
                    ctx->seting_data.battery_level = *(uint8_t *)data;
                    if (g_battery_label != NULL) {
                        char buf[16];
                        sprintf(buf, "%d%%", ctx->seting_data.battery_level);
                        lv_label_set_text(g_battery_label, buf);
                    }
                    break;
                default:
                    printf("UI partial update: unknown item_id=%d\r\n", item_id);
                    xSemaphoreGiveRecursive(lvgl_mutex);
                    return false;
            }
            break;

        default:
            printf("UI partial update: unsupported state=%d\r\n", ctx->current_state);
            xSemaphoreGiveRecursive(lvgl_mutex);
            return false;
    }

    xSemaphoreGiveRecursive(lvgl_mutex);

    /* 标记需要刷新 */
    ctx->need_refresh = 1;
    ctx->partial_update_flag = 1;
    ctx->partial_item_id = item_id;

    printf("UI partial update: state=%d item=%d\r\n", ctx->current_state, item_id);
    return true;
}
```

### 跨页面数据更新接口（跨页面数据刷新）

**大白话：当数据变化时，如果目标数据不在当前页面，需要先切换到目标页面，再用局部更新的方式刷新 LVGL 控件。**

```c
/**
 * @brief 跨页面数据更新函数
 * @param ctx UI 上下文指针
 * @param target_state 目标页面状态（数据要更新到哪个页面）
 * @param item_id 目标页面上的 UI 区块编号
 * @param data 新数据指针
 * @param data_len 数据长度（字符串类型需要，整型/浮点可填 0）
 * @return true 更新成功，false 参数无效
 * @note 作用：先切换到目标页面，再用局部更新的方式刷新 LVGL 控件
 *       适用于传感器数据需要显示在非当前页面的场景
 *       调用流程：1.切换页面 → 2.局部更新数据 → 3.刷新显示
 */
bool ui_cross_page_update(ui_context_t *ctx, ui_state_t target_state,
                          uint8_t item_id, const void *data, uint16_t data_len);
```

**使用示例：**

```c
/* 示例：当前在主菜单，但需要更新预览页面的温度数据 */
int16_t temperature = 25;
ui_cross_page_update(&ui_ctx, UI_STATE_LABEL_PREVIEW, 0, &temperature, 0);

/* 执行流程：
 * 1. 先调用 ui_state_machine_switch() 切换到 UI_STATE_LABEL_PREVIEW
 * 2. 再调用 ui_partial_update() 更新 item_id=0 的温度标签
 * 3. 自动设置 need_refresh = 1，lv_task_handler() 会刷新显示
 */
```

**实现示例（LVGL 版本）：**

```c
/**
 * @brief 跨页面数据更新函数实现（LVGL 版本）
 * @param ctx UI 上下文指针
 * @param target_state 目标页面状态
 * @param item_id 目标页面上的 UI 区块编号
 * @param data 新数据指针
 * @param data_len 数据长度
 * @return true 更新成功
 * @note 作用：先切换到目标页面，再用局部更新刷新 LVGL 控件
 */
bool ui_cross_page_update(ui_context_t *ctx, ui_state_t target_state,
                          uint8_t item_id, const void *data, uint16_t data_len)
{
    if (ctx == NULL || data == NULL) {
        printf("UI cross update failed: invalid param\r\n");
        return false;
    }

    if (target_state >= UI_STATE_MAX) {
        printf("UI cross update failed: invalid state=%d\r\n", target_state);
        return false;
    }

    /* 第 1 步：如果当前不在目标页面，先切换过去 */
    if (ctx->current_state != target_state) {
        if (!ui_state_machine_switch(ctx, target_state)) {
            printf("UI cross update: switch to state=%d failed\r\n", target_state);
            return false;
        }
        printf("UI cross update: switched from %d to %d\r\n",
               ctx->previous_state, target_state);
    }

    /* 第 2 步：在目标页面上执行局部更新 */
    if (!ui_partial_update(ctx, item_id, data, data_len)) {
        printf("UI cross update: partial update failed at state=%d item=%d\r\n",
               target_state, item_id);
        return false;
    }

    printf("UI cross update: state=%d item=%d success\r\n", target_state, item_id);
    return true;
}
```

### 局部更新与全页刷新的区别（LVGL 版本）

| 更新方式 | 函数 | 适用场景 | 性能 |
|---------|------|---------|------|
| 局部更新 | `ui_partial_update()` | 传感器数据频繁变化，只更新某个 LVGL 控件文本 | 高（只刷新局部） |
| 跨页面更新 | `ui_cross_page_update()` | 数据需要显示在非当前页面 | 中（先切页再局部刷新） |
| 全页刷新 | `ui_update_display()` | 页面切换、焦点移动、整体布局变化 | 低（整个页面重绘） |

### 在 ui_context_t 中增加局部更新相关字段（LVGL 版本）

```c
typedef struct {
    /* ... 原有字段 ... */

    uint8_t need_refresh;        /* 刷新标志：1=需要重新画画面，0=不用画 */

    /* 局部更新相关字段（强制要求） */
    uint8_t partial_update_flag; /* 局部更新标志：1=只刷新局部区域，0=全页刷新 */
    uint8_t partial_item_id;     /* 局部更新的区块编号：0=温度，1=湿度，2=名称... */

    /* LVGL 特有：页面切换动画 */
    lv_scr_load_anim_t anim_type;
    uint32_t anim_time;
} ui_context_t;
```

### LVGL 页面绘制函数如何支持局部更新

```c
/* 静态变量保存 LVGL 对象 */
static lv_obj_t *preview_screen = NULL;
static lv_obj_t *g_temp_label = NULL;
static lv_obj_t *g_humidity_label = NULL;
static lv_obj_t *g_name_label = NULL;

/**
 * @brief 预览页面绘制函数（LVGL 版本，支持局部更新）
 * @param ctx UI 上下文指针
 * @return true 画好了
 * @note 作用：根据 partial_update_flag 决定是全页重绘还是只更新局部 LVGL 控件
 */
bool ui_draw_label_preview(ui_context_t *ctx)
{
    /* 第一次进入时创建 LVGL 控件 */
    if (preview_screen == NULL) {
        preview_screen = lv_obj_create(NULL);

        lv_obj_t *title = lv_label_create(preview_screen);
        lv_label_set_text(title, "Label Preview");
        lv_obj_align(title, LV_ALIGN_TOP_MID, 0, 10);

        g_temp_label = lv_label_create(preview_screen);
        lv_obj_align(g_temp_label, LV_ALIGN_TOP_LEFT, 10, 40);

        g_humidity_label = lv_label_create(preview_screen);
        lv_obj_align(g_humidity_label, LV_ALIGN_TOP_LEFT, 10, 60);

        g_name_label = lv_label_create(preview_screen);
        lv_obj_align(g_name_label, LV_ALIGN_TOP_LEFT, 10, 80);
    }

    /* 如果是局部更新，只更新指定控件的文本 */
    if (ctx->partial_update_flag == 1) {
        switch (ctx->partial_item_id) {
            case 0:  /* 只更新温度标签 */
                if (g_temp_label != NULL) {
                    char buf[16];
                    sprintf(buf, "%d C", ctx->preview_data.temperature);
                    lv_label_set_text(g_temp_label, buf);
                }
                break;
            case 1:  /* 只更新湿度标签 */
                if (g_humidity_label != NULL) {
                    char buf[16];
                    sprintf(buf, "%.1f%%", ctx->preview_data.humidity);
                    lv_label_set_text(g_humidity_label, buf);
                }
                break;
            case 2:  /* 只更新商品名称标签 */
                if (g_name_label != NULL) {
                    lv_label_set_text(g_name_label, ctx->preview_data.l_product_name);
                }
                break;
        }

        /* 清除局部更新标志 */
        ctx->partial_update_flag = 0;
        ctx->partial_item_id = 0;

        /* 切换到这个页面 */
        lv_scr_load_anim(preview_screen, ctx->anim_type, ctx->anim_time, 0, false);
        return true;
    }

    /* 全页刷新：更新所有控件文本 */
    char buf[16];
    sprintf(buf, "%d C", ctx->preview_data.temperature);
    lv_label_set_text(g_temp_label, buf);

    sprintf(buf, "%.1f%%", ctx->preview_data.humidity);
    lv_label_set_text(g_humidity_label, buf);

    lv_label_set_text(g_name_label, ctx->preview_data.l_product_name);

    lv_scr_load_anim(preview_screen, ctx->anim_type, ctx->anim_time, 0, false);
    return true;
}
```

---

## LVGL 架构（RTOS 任务模式）

### 适用场景

工程中使用了 LVGL，有 `lvgl.h`、`lv_timer_handler()`、`lv_obj_t` 等。

### RTOS UI 任务示例

```c
/*
 * LVGL UI 任务（跑在 RTOS 里）
 * 大白话：这是一个专门负责 UI 的线程
 * 从消息队列里取按键消息，交给状态机处理
 * 然后调用 lv_task_handler() 让 LVGL 刷新屏幕
 */
void Task_LvglUI(void *arg)
{
    ui_msg_t msg;

    /* 初始化 UI 状态机 */
    ui_state_machine_init(&ui_ctx);

    while (1) {
        /* 第 1 步：从消息队列里取消息（不阻塞，取不到就跳过） */
        if (xQueueReceive(ui_msg_queue, &msg, 0) == pdPASS) {

            switch (msg.type) {
                case UI_MSG_KEY_EVENT:
                    /* 按键消息：翻译成 UI 事件，交给状态机 */
                    {
                        ui_event_t evt = ui_key_to_event(msg.key_code);
                        ui_state_machine_dispatch(&ui_ctx, evt, msg.param);
                    }
                    break;

                case UI_MSG_RTC_TIME:
                    /* 时间消息：更新时间数据，标记需要刷新 */
                    ui_update_time(&ui_ctx, msg.param);
                    ui_ctx.need_refresh = 1;
                    break;

                default:
                    break;
            }
        }

        /* 第 2 步：如果标记了需要刷新，调用画面刷新 */
        if (ui_ctx.need_refresh) {
            ui_update_display(&ui_ctx);
        }

        /* 第 3 步：让 LVGL 处理内部事务（动画、控件刷新等） */
        lv_task_handler();

        /* 每 5ms 循环一次 */
        vTaskDelay(pdMS_TO_TICKS(5));
    }
}

/**
 * @brief 把按键码翻译成 UI 事件
 * @param key_code 硬件按键码
 * @return 对应的 UI 事件类型
 * @note 作用：把各种硬件按键统一翻译成状态机能认识的事件
 */
static ui_event_t ui_key_to_event(uint16_t key_code)
{
    switch (key_code) {
        case KEY_UP:    return UI_EVT_UP;
        case KEY_DOWN:  return UI_EVT_DOWN;
        case KEY_LEFT:  return UI_EVT_LEFT;
        case KEY_RIGHT: return UI_EVT_RIGHT;
        case KEY_OK:    return UI_EVT_OK;
        case KEY_BACK:  return UI_EVT_BACK;
        default:        return UI_EVT_NONE;
    }
}
```

### LVGL 页面绘制函数示例

```c
/**
 * @brief 画主菜单页面（LVGL 版本）
 * @param ctx UI 上下文指针
 * @return true 画好了，false 画失败了
 * @note 作用：创建或更新主菜单的 LVGL 控件
 *       第一次进入时创建控件，之后只更新选中样式
 */
bool ui_draw_main_menu(ui_context_t *ctx)
{
    /* 第一次进入这个页面时创建控件 */
    if (main_menu_screen == NULL) {
        main_menu_screen = lv_obj_create(NULL);

        /* 创建标题 */
        lv_obj_t *title = lv_label_create(main_menu_screen);
        lv_label_set_text(title, "Main Menu");

        /* 创建菜单列表 */
        for (int i = 0; i < 3; i++) {
            menu_labels[i] = lv_label_create(main_menu_screen);
            lv_label_set_text(menu_labels[i], menu_items[i]);
            lv_obj_align(menu_labels[i], LV_ALIGN_TOP_LEFT, 10, 30 + i * 30);
        }
    }

    /* 更新选中样式：选中的项加粗，其他正常 */
    for (int i = 0; i < 3; i++) {
        if (i == ctx->selected_item) {
            lv_obj_set_style_text_font(menu_labels[i], &lv_font_montserrat_16, 0);
        } else {
            lv_obj_set_style_text_font(menu_labels[i], &lv_font_montserrat_14, 0);
        }
    }

    /* 切换到这个页面 */
    lv_scr_load(main_menu_screen);

    return true;
}
```

### LVGL 按键走状态机（不注册到 LVGL 控件上）

**完全可行，而且推荐这么做。**

具体做法：

```
按键硬件中断 / GPIO 扫描
    ↓
把按键码投递到 RTOS 消息队列（xQueueSend）
    ↓
UI 任务从队列取出消息
    ↓
翻译成 ui_event_t（比如 UI_EVT_OK）
    ↓
交给 ui_state_machine_dispatch() 处理
    ↓
状态机决定切页面 / 移焦点
    ↓
ui_update_display() 更新 LVGL 控件样式
    ↓
lv_task_handler() 刷新屏幕
```

**好处：**
- 按键逻辑和 LVGL 控件完全解耦
- 所有按键处理集中在状态机里，不散落在各个控件回调中
- 方便调试，所有事件都经过同一个入口
- 换输入设备（比如从按键换成旋钮）只改翻译层，状态机不用动

**对比：LVGL 控件事件回调方式**

```c
/* 这种方式也可以，但按键逻辑会散落在各个回调里 */
static void btn_event_cb(lv_event_t *e)
{
    lv_event_code_t code = lv_event_get_code(e);
    if (code == LV_EVENT_CLICKED) {
        /* 直接在这里处理业务逻辑 */
        do_something();
    }
}
```

**两种方式可以混用：**
- 物理按键（上、下、确认、返回）→ 走状态机
- LVGL 控件交互（按钮点击、滑块拖动）→ 走 LVGL 回调
- LVGL 回调里也可以投递 UI 事件给状态机处理

### LVGL 架构约束

- 状态机不要直接堆大量 `lv_obj_create()`，对象创建应放在页面函数中
- LVGL event callback 不要写复杂页面流程，优先只投递或转换 UI 事件
- 页面状态 `UI_STATE_xxx` 和 LVGL 控件状态 `LV_STATE_xxx` 必须分开
- screen 生命周期必须清楚：常驻 screen 初始化时创建，临时 screen 进入时创建、退出后删除
- 不要每次刷新都重复创建 LVGL 对象
- 裸机工程必须确认 LVGL tick 和 `lv_timer_handler()` 调用路径
- RTOS 工程必须确认 LVGL 调用线程或互斥策略
- 业务数据应从 `ui_context_t` 读取，不要把业务计算散落在控件 callback 里

---

## LVGL 触摸屏注册方案（方案 A：注册到 LVGL）

**适用场景**：用户选择"触摸屏注册到 LVGL"时使用本方案。

### 触摸屏注册到 LVGL 的完整流程

```
触摸屏硬件（SPI/I2C）
    ↓
触摸驱动芯片（XPT2046/GT911/FT6236）
    ↓
lv_indev_t 输入设备注册到 LVGL
    ↓
LVGL 内部处理触摸坐标、点击检测
    ↓
用户点击 LVGL 按钮/控件
    ↓
触发 lv_event_cb 回调
    ↓
在回调里投递 UI 事件给状态机，或直接处理业务
```

### 触摸屏驱动注册代码示例

```c
/*
 * 触摸屏输入设备注册到 LVGL
 * 大白话：把触摸屏"告诉"LVGL，让 LVGL 知道有触摸屏可以用
 * LVGL 会自动处理触摸坐标、判断点了哪个控件
 */

/* 触摸数据结构 */
static lv_indev_t *touch_indev;      /* LVGL 输入设备句柄 */
static lv_indev_data_t touch_data;   /* 当前触摸状态数据 */

/**
 * @brief 触摸屏读取回调函数
 * @param indev LVGL 输入设备句柄
 * @param data 输出参数，填入当前触摸状态
 * @note 作用：LVGL 每次刷新时会调用这个函数获取触摸坐标
 *       你需要在这里读取触摸屏芯片的数据，填到 data 里
 */
static void touch_read_cb(lv_indev_t *indev, lv_indev_data_t *data)
{
    /* 从触摸屏芯片读取坐标（以 XPT2046 为例） */
    uint16_t x, y;
    bool pressed = xpt2046_read(&x, &y);

    if (pressed) {
        data->state = LV_INDEV_STATE_PRESSED;  /* 按下了 */
        data->point.x = x;                      /* X 坐标 */
        data->point.y = y;                      /* Y 坐标 */
    } else {
        data->state = LV_INDEV_STATE_RELEASED; /* 松开了 */
    }
}

/**
 * @brief 注册触摸屏到 LVGL
 * @return true 注册成功，false 注册失败
 * @note 作用：在 LVGL 初始化时调用一次，把触摸屏接入 LVGL 事件系统
 */
bool ui_touch_init(void)
{
    /* 初始化触摸屏硬件 */
    if (!xpt2046_init()) {
        printf("Touch init failed\r\n");
        return false;
    }

    /* 创建 LVGL 输入设备 */
    touch_indev = lv_indev_create();
    if (touch_indev == NULL) {
        printf("Touch indev create failed\r\n");
        return false;
    }

    /* 设置输入设备类型为触摸屏 */
    lv_indev_set_type(touch_indev, LV_INDEV_TYPE_POINTER);

    /* 设置读取回调函数 */
    lv_indev_set_read_cb(touch_indev, touch_read_cb);

    /* 设置关联的显示器（单屏时通常是默认显示器） */
    lv_indev_set_display(touch_indev, lv_disp_get_default());

    printf("Touch registered to LVGL\r\n");
    return true;
}
```

### LVGL 控件事件回调示例

```c
/*
 * LVGL 按钮点击回调
 * 大白话：用户点了屏幕上的按钮，LVGL 会调用这个函数
 */
static void btn_click_cb(lv_event_t *e)
{
    lv_event_code_t code = lv_event_get_code(e);

    if (code == LV_EVENT_CLICKED) {
        /* 方式 1：直接在这里处理业务 */
        printf("Button clicked!\r\n");
        do_something();

        /* 方式 2：投递 UI 事件给状态机（推荐） */
        /* 这样按键和触摸的处理逻辑都集中在状态机里 */
        ui_event_t evt = UI_EVT_OK;
        ui_state_machine_dispatch(&ui_ctx, evt, 0);
    }
}

/**
 * @brief 画主菜单页面（LVGL 触摸屏版本）
 * @param ctx UI 上下文指针
 * @return true 画好了
 * @note 作用：创建 LVGL 按钮，注册点击回调
 */
bool ui_draw_main_menu(ui_context_t *ctx)
{
    if (main_menu_screen == NULL) {
        main_menu_screen = lv_obj_create(NULL);

        /* 创建按钮 */
        for (int i = 0; i < 3; i++) {
            btn[i] = lv_btn_create(main_menu_screen);
            lv_obj_set_size(btn[i], 200, 50);
            lv_obj_align(btn[i], LV_ALIGN_TOP_MID, 0, 30 + i * 60);

            /* 按钮上的文字 */
            label[i] = lv_label_create(btn[i]);
            lv_label_set_text(label[i], menu_items[i]);
            lv_obj_center(label[i]);

            /* 注册点击回调 */
            lv_obj_add_event_cb(btn[i], btn_click_cb, LV_EVENT_CLICKED, NULL);
        }
    }

    lv_scr_load(main_menu_screen);
    return true;
}
```

### 触摸屏注册方案的好处和坏处

**好处：**
- LVGL 自动处理触摸坐标、点击检测、控件交互
- 支持复杂控件（列表滚动、滑块拖动、下拉菜单）
- 开发效率高，不用自己算触摸区域

**坏处：**
- 触摸逻辑分散在各个控件回调里
- 物理按键和触摸屏的处理逻辑不统一
- 调试时触摸事件不经过状态机，不好追踪

---

## LVGL 触摸屏状态机方案（方案 B：不注册到 LVGL）

**适用场景**：用户选择"触摸屏不注册到 LVGL，走状态机"时使用本方案。

### 触摸屏走状态机的完整流程

```
触摸屏硬件（SPI/I2C）
    ↓
自己的代码读取触摸坐标（xpt2046_read / gt911_read）
    ↓
判断触摸区域（比如 x>10 && x<100 && y>20 && y<60 → 按钮1）
    ↓
翻译成 UI 事件（UI_EVT_OK / UI_EVT_BACK 等）
    ↓
投递到消息队列（或直接调用）
    ↓
ui_state_machine_dispatch() 统一处理
    ↓
状态机决定切页面 / 移焦点
    ↓
ui_update_display() 更新画面
```

### 触摸屏走状态机的代码示例

```c
/*
 * 触摸屏走状态机（不注册到 LVGL）
 * 大白话：自己读触摸坐标，自己判断点了哪里，翻译成事件给状态机
 */

/**
 * @brief 触摸屏扫描和事件翻译
 * @param ctx UI 上下文指针
 * @return 翻译后的 UI 事件，UI_EVT_NONE 表示没有触摸
 * @note 作用：读取触摸坐标，判断点了哪个区域，翻译成统一事件
 */
static ui_event_t ui_touch_scan(ui_context_t *ctx)
{
    uint16_t x, y;
    bool pressed = xpt2046_read(&x, &y);

    if (!pressed) {
        return UI_EVT_NONE;  /* 没触摸 */
    }

    /* 根据当前页面判断点了哪个区域 */
    switch (ctx->current_state) {

        case UI_STATE_MAIN_MENU:
            /* 主菜单：判断点了哪个菜单项 */
            if (x >= 10 && x <= 210) {
                if (y >= 30 && y <= 80) {
                    ctx->selected_item = 0;
                    return UI_EVT_OK;  /* 点了第 1 项 */
                } else if (y >= 90 && y <= 140) {
                    ctx->selected_item = 1;
                    return UI_EVT_OK;  /* 点了第 2 项 */
                } else if (y >= 150 && y <= 200) {
                    ctx->selected_item = 2;
                    return UI_EVT_OK;  /* 点了第 3 项 */
                }
            }
            break;

        case UI_STATE_LABEL_PREVIEW:
            /* 标签预览：点了返回区域 */
            if (x >= 10 && x <= 100 && y >= 200 && y <= 240) {
                return UI_EVT_BACK;
            }
            break;

        default:
            break;
    }

    return UI_EVT_NONE;
}

/**
 * @brief RTOS UI 任务（触摸屏走状态机版本）
 * @param arg 任务参数
 * @note 作用：从消息队列取按键事件 + 扫描触摸屏，统一交给状态机处理
 */
void Task_LvglUI(void *arg)
{
    ui_msg_t msg;

    ui_state_machine_init(&ui_ctx);

    while (1) {
        /* 第 1 步：从消息队列取按键消息 */
        if (xQueueReceive(ui_msg_queue, &msg, 0) == pdPASS) {
            if (msg.type == UI_MSG_KEY_EVENT) {
                ui_event_t evt = ui_key_to_event(msg.key_code);
                ui_state_machine_dispatch(&ui_ctx, evt, msg.param);
            }
        }

        /* 第 2 步：扫描触摸屏 */
        ui_event_t touch_evt = ui_touch_scan(&ui_ctx);
        if (touch_evt != UI_EVT_NONE) {
            ui_state_machine_dispatch(&ui_ctx, touch_evt, 0);
        }

        /* 第 3 步：刷新画面 */
        if (ui_ctx.need_refresh) {
            ui_update_display(&ui_ctx);
        }

        lv_task_handler();
        vTaskDelay(pdMS_TO_TICKS(5));
    }
}
```

### 触摸屏走状态机方案的好处和坏处

**好处：**
- 所有输入（按键 + 触摸）都走状态机，逻辑统一
- 方便调试，所有事件都经过 `ui_state_machine_dispatch()`
- 换输入设备只改翻译层，状态机不用动

**坏处：**
- 需要自己算触摸区域，开发效率低
- 复杂控件（列表滚动、滑块拖动）不好实现
- 页面布局改了，触摸区域代码也要跟着改

---

## ESP32 LVGL 工程实践要点

**以下内容来自 ESP32 社区的主流做法，供参考。**

### 双核分工（ESP32 特有）

ESP32 有两个 CPU 核心，推荐这样分工：

```
Core 0（PRO_CPU）：
  - FreeRTOS 内核任务
  - WiFi / 蓝牙协议栈
  - 其他后台任务

Core 1（APP_CPU）：
  - LVGL 主线程（lv_timer_handler + 渲染）
  - UI 任务
  - 传感器数据采集（如果不影响 UI 帧率）
```

**好处**：LVGL 渲染和网络通信互不干扰，减少卡顿。

### 线程安全（强制要求）

**RTOS 环境下，所有 LVGL 操作必须加锁！**

```c
/*
 * LVGL 互斥锁
 * 大白话：LVGL 不是线程安全的，多个任务同时操作 LVGL 对象会崩溃
 * 必须用互斥锁保护所有 LVGL 调用
 */

/* 在 LVGL 初始化时创建互斥锁 */
SemaphoreHandle_t lvgl_mutex = xSemaphoreCreateRecursiveMutex();

/* 在任何任务中操作 LVGL 对象前加锁 */
xSemaphoreTakeRecursive(lvgl_mutex, portMAX_DELAY);
lv_label_set_text(label, "New Value");
lv_obj_set_style_bg_color(btn, lv_color_red(), 0);
xSemaphoreGiveRecursive(lvgl_mutex);

/* 或者用 lvgl_port 提供的封装 */
lvgl_port_lock();
lv_obj_t *scr = lv_disp_get_scr_act(NULL);
lvgl_port_unlock();
```

**不加锁的后果**：
- UI 任务和数据更新任务同时操作同一个 LVGL 对象
- 内存损坏、程序崩溃、屏幕花屏

### 页面生命周期管理

**关键认知：`lv_scr_load()` 只是切换显示指针，不销毁旧页面！**

- 所有页面（`lv_obj_t *`）都常驻内存
- 旧页面的定时器、事件回调仍在运行
- 必须手动管理页面生命周期

**推荐页面生命周期钩子**：

```c
/*
 * 页面生命周期钩子
 * 大白话：进入页面时启动定时器，离开页面时暂停定时器
 * 避免后台页面浪费 CPU 资源
 */
typedef struct {
    ui_state_t state;
    bool (*create)(ui_context_t *ctx);   /* 首次创建控件 */
    bool (*enter)(ui_context_t *ctx);    /* 进入页面时调用 */
    bool (*refresh)(ui_context_t *ctx);  /* 刷新数据 */
    bool (*exit)(ui_context_t *ctx);     /* 离开页面时调用 */
    bool (*destroy)(ui_context_t *ctx);  /* 销毁控件释放内存 */
} ui_page_ops_t;

/* 使用示例 */
static lv_timer_t *gauge_timer = NULL;

static void gauge_page_enter(ui_context_t *ctx)
{
    /* 进入仪表盘页面时启动定时器 */
    if (gauge_timer == NULL) {
        gauge_timer = lv_timer_create(gauge_update_cb, 1000, NULL);
    }
}

static void gauge_page_exit(ui_context_t *ctx)
{
    /* 离开仪表盘页面时暂停定时器 */
    if (gauge_timer != NULL) {
        lv_timer_pause(gauge_timer);
    }
}
```

### 内存管理建议（ESP32 内存紧张）

ESP32 内存紧张（520KB SRAM），需要注意：

```c
/*
 * LVGL 内存配置建议
 * 大白话：ESP32 内存不够用，要精打细算
 */

/* lv_conf.h 配置 */
#define LV_COLOR_DEPTH 16              /* 16 位色深，省内存 */
#define LV_MEM_SIZE (64 * 1024)        /* LVGL 内存池 64KB */
#define LV_DISP_DEF_REFR_PERIOD 33     /* 刷新周期 33ms ≈ 30fps */

/* 内存优化技巧 */
1. 非活跃页面的定时器该暂停就暂停（lv_timer_pause）
2. 大控件（如 lv_chart 曲线缓冲区）在非活跃时主动释放
3. 用 PSRAM 存图片资源（lv_img_set_src 支持 PSRAM）
4. 避免同时创建太多页面，按需创建、用完销毁
5. 用 lv_obj_clean(screen) 清除页面内容后复用对象
```

**内存占用参考**：
- 3~5 个简单页面：约 120KB RAM
- 3~5 个复杂页面（含图表、曲线）：约 200KB~300KB RAM
- 超过 400KB 时要警惕内存不足

### lvgl_port 使用（ESP-IDF 官方推荐）

```c
/*
 * ESP-IDF lvgl_port 使用示例
 * 大白话：乐鑫官方提供的 LVGL 移植层，帮你搞定 FreeRTOS 集成
 */

#include "esp_lvgl_port.h"

/* LVGL 端口配置 */
const lvgl_port_cfg_t lvgl_cfg = {
    .task_priority = 5,           /* 任务优先级（低于 WiFi/BT） */
    .task_stack = 4096,           /* 任务栈大小 */
    .task_affinity = 1,           /* 绑定到 CPU1 */
    .task_max_sleep_ms = 500,     /* 最大休眠时间 */
};

/* 初始化 LVGL 端口 */
esp_lvgl_port_init(&lvgl_cfg);

/* 添加显示器 */
const esp_lvgl_port_disp_config_t disp_cfg = {
    .panel_handle = panel_handle,
    .buffer_size = LCD_WIDTH * LCD_HEIGHT,
    .double_buffer = true,
    .hres = LCD_WIDTH,
    .vres = LCD_HEIGHT,
};
lv_disp_t *disp = esp_lvgl_port_add_disp(&disp_cfg);

/* 添加触摸屏（如果注册到 LVGL） */
const esp_lvgl_port_touch_config_t touch_cfg = {
    .dev_handle = touch_handle,
};
lv_indev_t *touch_indev = esp_lvgl_port_add_touch(&touch_cfg);
```

---

## 页面如何注册到框架中

### 注册步骤（大白话版）

**第 1 步：在枚举里加一个新页面状态**

```c
typedef enum {
    UI_STATE_MAIN_MENU = 0,
    UI_STATE_LABEL_PREVIEW,
    UI_STATE_SETTING,
    UI_STATE_NEW_PAGE,     /* ← 新增页面 */
    UI_STATE_MAX
} ui_state_t;
```

**第 2 步：写一个绘制函数**

```c
/**
 * @brief 画新页面
 * @param ctx UI 上下文指针
 * @return true 画好了
 * @note 作用：在屏幕上画出新页面的内容
 */
bool ui_draw_new_page(ui_context_t *ctx)
{
    /* 创建/更新 LVGL 控件 */
    return true;
}
```

**第 3 步：在渲染路由里加一个 case**

```c
bool ui_update_display(ui_context_t *ctx)
{
    switch (ctx->current_state) {
        case UI_STATE_MAIN_MENU:
            return ui_draw_main_menu(ctx);
        case UI_STATE_LABEL_PREVIEW:
            return ui_draw_label_preview(ctx);
        case UI_STATE_SETTING:
            return ui_draw_setting(ctx);
        case UI_STATE_NEW_PAGE:        /* ← 新增 */
            return ui_draw_new_page(ctx);
        default:
            return false;
    }
}
```

**第 4 步：在状态机里加跳转逻辑**

```c
/* 在 ui_state_machine_dispatch() 的 switch 里加 */
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

**完成！** 新页面就注册到框架里了。

---

## UI 框架完整使用指南（LVGL 版本 - 大白话版）

### 一、这个框架到底是干什么的？

**一句话：让 LVGL 嵌入式 UI 代码不乱。**

没有框架时，你的按键处理、触摸处理、页面切换、LVGL 控件创建全混在一起，页面一多就改不动。

有了框架后：
- **按键/触摸** → 翻译成统一事件
- **状态机** → 决定下一步干什么
- **页面函数** → 只管创建/更新 LVGL 控件
- **LVGL 任务** → 统一调度刷新
- 各司其职，互不干扰

### 二、框架组件一览

| 组件 | 干什么的 | 你需要做什么 |
|------|---------|-------------|
| `ui_event_t` 枚举 | 定义所有可能的 UI 事件 | 根据硬件输入增减事件类型 |
| `ui_state_t` 枚举 | 定义所有页面 | 每新增一个页面就加一项 |
| `ui_context_t` 结构体 | 存所有 UI 状态和数据 | 一般不用改，除非需要存新数据 |
| `ui_state_machine_init()` | 初始化 UI 状态 | 开机时调用一次 |
| `ui_state_machine_dispatch()` | 处理 UI 事件 | 每次有按键/触摸事件时调用 |
| `ui_state_machine_switch()` | 切换页面 | 在 dispatch 里调用，不要直接改状态 |
| `ui_update_display()` | 刷新画面 | UI 任务主循环里调用 |
| `ui_draw_xxx()` | 画具体页面（创建/更新 LVGL 控件） | 每个页面写一个，在 update_display 里注册 |
| `lv_task_handler()` | LVGL 内部刷新 | UI 任务主循环里每次都调用 |

### 三、页面如何注册到框架中（LVGL 版本）

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

**第 2 步：写一个 LVGL 页面绘制函数**

```c
/**
 * @brief 画新页面（LVGL 版本）
 * @param ctx UI 上下文指针，里面存着当前页面状态和要显示的数据
 * @return true 画好了，false 画失败了
 * @note 作用：创建或更新 LVGL 控件来显示新页面内容
 */
bool ui_draw_new_page(ui_context_t *ctx)
{
    /* 静态变量保存 LVGL 对象，避免重复创建 */
    static lv_obj_t *new_page_screen = NULL;
    static lv_obj_t *title_label = NULL;
    static lv_obj_t *content_label = NULL;

    /* 第一次进入这个页面时创建控件 */
    if (new_page_screen == NULL) {
        /* 创建页面容器 */
        new_page_screen = lv_obj_create(NULL);

        /* 创建标题 */
        title_label = lv_label_create(new_page_screen);
        lv_label_set_text(title_label, "New Page");
        lv_obj_align(title_label, LV_ALIGN_TOP_MID, 0, 10);

        /* 创建内容标签 */
        content_label = lv_label_create(new_page_screen);
        lv_obj_align(content_label, LV_ALIGN_TOP_LEFT, 10, 50);
    }

    /* 更新内容（从 ctx 读数据） */
    lv_label_set_text(content_label, ctx->preview_data.l_type);

    /* 切换到这个页面（带淡入动画） */
    lv_scr_load_anim(new_page_screen, LV_SCR_LOAD_ANIM_FADE_ON, 200, 0, false);

    return true;
}
```

> 大白话：这个函数就是你的页面"长什么样"。第一次进入时创建 LVGL 控件，之后只更新数据。框架会在需要时调用它来画画面。

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

    /* LVGL 特有：设置页面切换动画 */
    ctx->anim_type = LV_SCR_LOAD_ANIM_FADE_ON;
    ctx->anim_time = 200;

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
| 2. 写绘制函数 | `bool ui_draw_about(ctx)` | 创建 LVGL 控件画"关于"页面 |
| 3. 渲染路由加 case | `ui_update_display()` 里加 case | `case UI_STATE_ABOUT: return ui_draw_about(ctx);` |
| 4. 状态机加跳转 | `dispatch()` 里加跳转逻辑 | 从主菜单按 OK 进入"关于" |

> 大白话：就是"重复 4 步"。框架不限制页面数量，你加多少个都行。

### 六、页面内部如何创建才能接入框架（LVGL 版本）

**大白话：LVGL 页面绘制函数有固定格式，照着写就能接入。**

```c
/**
 * @brief 画 XXX 页面（LVGL 版本）
 * @param ctx UI 上下文指针
 * @return true 画好了
 * @note 作用：创建或更新 LVGL 控件来显示 XXX 页面
 */
bool ui_draw_xxx(ui_context_t *ctx)
{
    /* ===== 第 1 部分：静态变量保存 LVGL 对象 ===== */
    static lv_obj_t *xxx_screen = NULL;
    static lv_obj_t *title_label = NULL;
    static lv_obj_t *data_label = NULL;

    /* ===== 第 2 部分：第一次进入时创建控件 ===== */
    if (xxx_screen == NULL) {
        /* 创建页面容器 */
        xxx_screen = lv_obj_create(NULL);

        /* 创建标题 */
        title_label = lv_label_create(xxx_screen);
        lv_label_set_text(title_label, "XXX Page");
        lv_obj_align(title_label, LV_ALIGN_TOP_MID, 0, 10);

        /* 创建数据标签 */
        data_label = lv_label_create(xxx_screen);
        lv_obj_align(data_label, LV_ALIGN_TOP_LEFT, 10, 50);
    }

    /* ===== 第 3 部分：更新数据（从 ctx 读） ===== */
    lv_label_set_text(data_label, ctx->xxx_data.name);

    /* ===== 第 4 部分：更新选中样式（根据 selected_item） ===== */
    if (ctx->selected_item == 0) {
        lv_obj_set_style_text_color(data_label, lv_color_blue(), 0);
    } else {
        lv_obj_set_style_text_color(data_label, lv_color_black(), 0);
    }

    /* ===== 第 5 部分：切换到这个页面 ===== */
    lv_scr_load_anim(xxx_screen, ctx->anim_type, ctx->anim_time, 0, false);

    return true;
}
```

**关键规则：**
- 页面函数只负责"画"（创建/更新 LVGL 控件），不要在里面写按键判断
- 数据从 `ctx` 里读，不要直接读全局变量
- 选中状态看 `ctx->selected_item`，框架会帮你更新这个值
- LVGL 对象用 `static` 变量保存，避免重复创建
- 返回 `true` 表示画好了，返回 `false` 表示画失败了
- 使用 `lv_scr_load_anim()` 切换页面，支持动画效果

### 七、按键和触摸屏如何接入（LVGL 版本）

**大白话：LVGL 工程支持两种输入方式，可以单独用也可以混用。**

#### 7.1 方案 A：物理按键走状态机（推荐）

**适用场景：** 有物理按键（上、下、确认、返回）的工程。

```
用户按了一个键
    ↓
按键驱动检测到按键（GPIO 中断 / 轮询扫描）
    ↓
把按键码投递到 RTOS 消息队列（xQueueSend）
    ↓
UI 任务从队列取出消息
    ↓
翻译成统一事件（UI_EVT_UP / UI_EVT_DOWN / UI_EVT_OK 等）
    ↓
调用 ui_state_machine_dispatch(ctx, event, param)
    ↓
状态机根据当前页面 + 事件，决定下一步干什么
```

**代码示例：**

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

/* UI 任务主循环 */
void Task_LvglUI(void *arg)
{
    ui_msg_t msg;
    ui_state_machine_init(&ui_ctx);

    while (1) {
        /* 第 1 步：从消息队列取按键消息 */
        if (xQueueReceive(ui_msg_queue, &msg, 0) == pdPASS) {
            if (msg.type == UI_MSG_KEY_EVENT) {
                ui_event_t evt = ui_key_to_event(msg.key_code);
                ui_state_machine_dispatch(&ui_ctx, evt, msg.param);
            }
        }

        /* 第 2 步：刷新画面 */
        if (ui_ctx.need_refresh) {
            ui_update_display(&ui_ctx);
        }

        /* 第 3 步：让 LVGL 处理内部事务 */
        lv_task_handler();

        vTaskDelay(pdMS_TO_TICKS(5));
    }
}
```

#### 7.2 方案 B：触摸屏注册到 LVGL（LVGL 原生方式）

**适用场景：** 大量 LVGL 控件交互（按钮点击、滑块拖动、列表滚动）。

```
触摸屏硬件（SPI/I2C）
    ↓
触摸驱动芯片（XPT2046/GT911/FT6236）
    ↓
lv_indev_t 输入设备注册到 LVGL
    ↓
LVGL 内部处理触摸坐标、点击检测
    ↓
用户点击 LVGL 按钮/控件
    ↓
触发 lv_event_cb 回调
    ↓
在回调里投递 UI 事件给状态机，或直接处理业务
```

**代码示例：**

```c
/* 触摸数据结构 */
static lv_indev_t *touch_indev;
static lv_indev_data_t touch_data;

/**
 * @brief 触摸屏读取回调函数
 * @param indev LVGL 输入设备句柄
 * @param data 输出参数，填入当前触摸状态
 * @note 作用：LVGL 每次刷新时会调用这个函数获取触摸坐标
 */
static void touch_read_cb(lv_indev_t *indev, lv_indev_data_t *data)
{
    uint16_t x, y;
    bool pressed = xpt2046_read(&x, &y);

    if (pressed) {
        data->state = LV_INDEV_STATE_PRESSED;
        data->point.x = x;
        data->point.y = y;
    } else {
        data->state = LV_INDEV_STATE_RELEASED;
    }
}

/**
 * @brief 注册触摸屏到 LVGL
 * @return true 注册成功，false 注册失败
 * @note 作用：在 LVGL 初始化时调用一次
 */
bool ui_touch_init(void)
{
    if (!xpt2046_init()) {
        printf("Touch init failed\r\n");
        return false;
    }

    touch_indev = lv_indev_create();
    if (touch_indev == NULL) {
        printf("Touch indev create failed\r\n");
        return false;
    }

    lv_indev_set_type(touch_indev, LV_INDEV_TYPE_POINTER);
    lv_indev_set_read_cb(touch_indev, touch_read_cb);
    lv_indev_set_display(touch_indev, lv_disp_get_default());

    printf("Touch registered to LVGL\r\n");
    return true;
}

/* LVGL 按钮点击回调 */
static void btn_click_cb(lv_event_t *e)
{
    lv_event_code_t code = lv_event_get_code(e);

    if (code == LV_EVENT_CLICKED) {
        /* 方式 1：直接处理业务 */
        printf("Button clicked!\r\n");
        do_something();

        /* 方式 2：投递 UI 事件给状态机（推荐） */
        ui_event_t evt = UI_EVT_OK;
        ui_state_machine_dispatch(&ui_ctx, evt, 0);
    }
}
```

#### 7.3 方案 C：混合方式（物理按键 + 触摸屏）

**适用场景：** 既有物理按键又有触摸屏的工程。

- 物理按键（上、下、确认、返回）→ 走状态机
- 触摸屏注册到 LVGL，LVGL 控件回调处理触摸交互
- LVGL 回调里也可以投递 UI 事件给状态机处理

**好处：** 物理按键逻辑集中，触摸屏交互灵活。

#### 7.4 方案 D：触摸屏走状态机（不注册到 LVGL）

**适用场景：** 自定义 UI 逻辑、不想依赖 LVGL 事件系统。

```c
/**
 * @brief 触摸屏扫描和事件翻译
 * @param ctx UI 上下文指针
 * @return 翻译后的 UI 事件，UI_EVT_NONE 表示没有触摸
 * @note 作用：读取触摸坐标，判断点了哪个区域，翻译成统一事件
 */
static ui_event_t ui_touch_scan(ui_context_t *ctx)
{
    uint16_t x, y;
    bool pressed = xpt2046_read(&x, &y);

    if (!pressed) {
        return UI_EVT_NONE;
    }

    /* 根据当前页面判断点了哪个区域 */
    switch (ctx->current_state) {
        case UI_STATE_MAIN_MENU:
            if (x >= 10 && x <= 210) {
                if (y >= 30 && y <= 80) {
                    ctx->selected_item = 0;
                    return UI_EVT_OK;
                } else if (y >= 90 && y <= 140) {
                    ctx->selected_item = 1;
                    return UI_EVT_OK;
                }
            }
            break;

        case UI_STATE_LABEL_PREVIEW:
            if (x >= 10 && x <= 100 && y >= 200 && y <= 240) {
                return UI_EVT_BACK;
            }
            break;

        default:
            break;
    }

    return UI_EVT_NONE;
}

/* UI 任务主循环（触摸屏走状态机版本） */
void Task_LvglUI(void *arg)
{
    ui_msg_t msg;
    ui_state_machine_init(&ui_ctx);

    while (1) {
        /* 第 1 步：从消息队列取按键消息 */
        if (xQueueReceive(ui_msg_queue, &msg, 0) == pdPASS) {
            if (msg.type == UI_MSG_KEY_EVENT) {
                ui_event_t evt = ui_key_to_event(msg.key_code);
                ui_state_machine_dispatch(&ui_ctx, evt, msg.param);
            }
        }

        /* 第 2 步：扫描触摸屏 */
        ui_event_t touch_evt = ui_touch_scan(&ui_ctx);
        if (touch_evt != UI_EVT_NONE) {
            ui_state_machine_dispatch(&ui_ctx, touch_evt, 0);
        }

        /* 第 3 步：刷新画面 */
        if (ui_ctx.need_refresh) {
            ui_update_display(&ui_ctx);
        }

        lv_task_handler();
        vTaskDelay(pdMS_TO_TICKS(5));
    }
}
```

### 八、数据如何传递（LVGL 版本）

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

    /* LVGL 特有：页面切换动画 */
    lv_scr_load_anim_t anim_type;
    uint32_t anim_time;
} ui_context_t;
```

#### 8.2 往页面传数据

```c
/* 其他任务（比如传感器任务、RTC 任务）往 ctx 里写数据 */
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
    static lv_obj_t *preview_screen = NULL;
    static lv_obj_t *name_label = NULL;
    static lv_obj_t *date_label = NULL;
    static lv_obj_t *count_label = NULL;

    if (preview_screen == NULL) {
        preview_screen = lv_obj_create(NULL);
        name_label = lv_label_create(preview_screen);
        date_label = lv_label_create(preview_screen);
        count_label = lv_label_create(preview_screen);

        lv_obj_align(name_label, LV_ALIGN_TOP_LEFT, 10, 10);
        lv_obj_align(date_label, LV_ALIGN_TOP_LEFT, 10, 30);
        lv_obj_align(count_label, LV_ALIGN_TOP_LEFT, 10, 50);
    }

    /* 从 ctx 读数据 */
    lv_label_set_text(name_label, ctx->preview_data.name);
    lv_label_set_text(date_label, ctx->preview_data.date);

    char buf[20];
    sprintf(buf, "Count: %d", ctx->preview_data.count);
    lv_label_set_text(count_label, buf);

    lv_scr_load_anim(preview_screen, ctx->anim_type, ctx->anim_time, 0, false);
    return true;
}
```

> 大白话：数据传递就三步——写数据到 ctx、设 need_refresh=1、框架自动调用绘制函数来画。

### 九、页面如何刷新（LVGL 版本）

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
UI 任务主循环检测到 need_refresh == 1
    ↓
调用 ui_update_display(&ui_ctx)
    ↓
根据 ctx->current_state 找到对应的 ui_draw_xxx()
    ↓
ui_draw_xxx() 创建/更新 LVGL 控件
    ↓
调用 lv_scr_load_anim() 切换页面
    ↓
画完后 ctx->need_refresh = 0（框架自动清除）
    ↓
lv_task_handler() 让 LVGL 真正刷新屏幕
```

#### 9.3 关键代码

```c
/* UI 任务主循环里 */
while (1) {
    /* ... 按键/触摸处理 ... */

    /* 只有 need_refresh == 1 时才画，省 CPU 时间 */
    if (ui_ctx.need_refresh) {
        ui_update_display(&ui_ctx);
    }

    /* 让 LVGL 处理内部事务（动画、控件刷新等） */
    lv_task_handler();

    vTaskDelay(pdMS_TO_TICKS(5));
}
```

> 大白话：`need_refresh` 就像一个 flag，有人改了数据或切了页面就把它设成 1，UI 任务看到 1 就去画，画完自动清零。没设 1 就不画，省时间。`lv_task_handler()` 是 LVGL 的刷新函数，每次循环都要调用。

### 十、关键参数说明（LVGL 版本）

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
| `anim_type` | `lv_scr_load_anim_t` | LVGL 页面切换动画类型（淡入、滑动等） |
| `anim_time` | `uint32_t` | 动画时长（毫秒） |
| `partial_update_flag` | `uint8_t` | 局部更新标志（1=只刷新局部 LVGL 控件，0=全页刷新）【强制要求】 |
| `partial_item_id` | `uint8_t` | 局部更新的区块编号（0=温度标签，1=湿度标签，2=名称标签...）【强制要求】 |

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

/* 事件分发：每次有按键/触摸事件时调用 */
bool ui_state_machine_dispatch(ui_context_t *ctx, ui_event_t event, uint32_t param);
/* ctx：UI 上下文指针 */
/* event：翻译后的 UI 事件（比如 UI_EVT_OK） */
/* param：附加参数（比如旋钮转了多少格），没有就填 0 */

/* 页面切换：在 dispatch 里调用 */
bool ui_state_machine_switch(ui_context_t *ctx, ui_state_t new_state);
/* ctx：UI 上下文指针 */
/* new_state：要切到的目标页面状态 */

/* 画面刷新：UI 任务主循环里调用 */
bool ui_update_display(ui_context_t *ctx);
/* ctx：UI 上下文指针 */
```

### 十一、如何使用局部数据更新接口（LVGL 版本，强制要求）

**大白话：传感器数据变了，不需要整个页面重画，只更新对应 LVGL 控件的文本就行。**

#### 11.1 什么时候用局部更新？

| 场景 | 用什么 | 原因 |
|------|--------|------|
| 温度从 25°C 变成 26°C | `ui_partial_update()` | 只变了数字，其他控件没变 |
| 电池电量从 80% 变成 79% | `ui_partial_update()` | 只变了数字，其他控件没变 |
| 用户按了确认键进入新页面 | `ui_state_machine_switch()` | 整个页面都变了，必须全画 |
| 用户上下移动焦点 | 设 `need_refresh = 1` | 高亮位置变了，需要全页刷新 |

#### 11.2 使用步骤

```c
/* 第 1 步：准备好新数据 */
int16_t new_temp = 26;

/* 第 2 步：调用局部更新函数 */
/* 参数说明：
 *   &ui_ctx      - UI 上下文指针
 *   0            - item_id，表示要更新第 0 号区块（比如温度标签）
 *   &new_temp    - 新数据指针
 *   0            - data_len，整型数据填 0 就行
 */
ui_partial_update(&ui_ctx, 0, &new_temp, 0);

/* 第 3 步：UI 任务自动刷新 */
/* ui_partial_update() 内部已经设了 need_refresh = 1
 * UI 任务检测到 need_refresh == 1 就会调用 ui_update_display()
 * ui_update_display() 调用 ui_draw_xxx()
 * ui_draw_xxx() 检测到 partial_update_flag == 1，只更新指定 LVGL 控件
 */
```

#### 11.3 item_id 怎么定义？

**大白话：item_id 就是页面上每个 LVGL 控件的编号，你自己定义。**

```c
/* 例如预览页面的区块定义 */
#define PREVIEW_ITEM_TEMPERATURE    0   /* 温度标签 */
#define PREVIEW_ITEM_HUMIDITY       1   /* 湿度标签 */
#define PREVIEW_ITEM_PRODUCT_NAME   2   /* 商品名称标签 */
#define PREVIEW_ITEM_DATE           3   /* 日期标签 */

/* 例如设置页面的区块定义 */
#define SETTING_ITEM_BATTERY        0   /* 电池电量标签 */
#define SETTING_ITEM_BRIGHTNESS     1   /* 亮度标签 */
```

> 大白话：每个页面有哪些显示控件，就定义多少个 item_id。不同页面的 item_id 从 0 开始重新编号，互不干扰。

#### 11.4 页面绘制函数如何支持局部更新（LVGL 版本）

```c
bool ui_draw_label_preview(ui_context_t *ctx)
{
    /* 第一次进入时创建 LVGL 控件 */
    if (preview_screen == NULL) {
        preview_screen = lv_obj_create(NULL);
        g_temp_label = lv_label_create(preview_screen);
        g_humidity_label = lv_label_create(preview_screen);
        g_name_label = lv_label_create(preview_screen);
        /* ... 对齐等样式设置 ... */
    }

    /* 检查是否是局部更新 */
    if (ctx->partial_update_flag == 1) {
        /* 只更新指定 LVGL 控件的文本 */
        switch (ctx->partial_item_id) {
            case PREVIEW_ITEM_TEMPERATURE:
                if (g_temp_label != NULL) {
                    char buf[16];
                    sprintf(buf, "%d C", ctx->preview_data.temperature);
                    lv_label_set_text(g_temp_label, buf);
                }
                break;

            case PREVIEW_ITEM_HUMIDITY:
                if (g_humidity_label != NULL) {
                    char buf[16];
                    sprintf(buf, "%.1f%%", ctx->preview_data.humidity);
                    lv_label_set_text(g_humidity_label, buf);
                }
                break;
        }

        /* 清除局部更新标志 */
        ctx->partial_update_flag = 0;
        ctx->partial_item_id = 0;

        /* 切换到这个页面 */
        lv_scr_load_anim(preview_screen, ctx->anim_type, ctx->anim_time, 0, false);
        return true;
    }

    /* 不是局部更新 → 全页刷新：更新所有控件 */
    /* ... 更新所有 LVGL 控件文本 ... */
    lv_scr_load_anim(preview_screen, ctx->anim_type, ctx->anim_time, 0, false);
    return true;
}
```

> 大白话：页面函数里先检查 `partial_update_flag`，如果是 1 就只更新指定 LVGL 控件的文本，更新完把标志清零。如果是 0 就更新所有控件。

#### 11.5 外部传感器数据如何接入（LVGL 版本）

```c
/* 传感器任务：每 1 秒读一次温度 */
void Task_Sensor(void *arg)
{
    while (1) {
        /* 第 1 步：读取传感器数据 */
        int16_t temp = DHT11_ReadTemperature();

        /* 第 2 步：判断当前在哪个页面 */
        if (ui_ctx.current_state == UI_STATE_LABEL_PREVIEW) {
            /* 当前在预览页 → 直接局部更新温度标签 */
            ui_partial_update(&ui_ctx, PREVIEW_ITEM_TEMPERATURE, &temp, 0);
        } else {
            /* 当前不在预览页 → 只更新数据，不刷新显示 */
            ui_ctx.preview_data.temperature = temp;
            /* 等用户切到预览页时自然会显示最新数据 */
        }

        vTaskDelay(1000);
    }
}
```

> 大白话：传感器数据变了，先看当前在哪个页面。如果在显示这个数据的页面，就调 `ui_partial_update()` 局部刷新 LVGL 控件。如果不在，就只更新数据，等用户切过去时自然会显示最新值。

### 十二、如何使用跨页面数据更新接口（LVGL 版本，强制要求）

**大白话：数据需要显示在别的页面，先切到那个页面，再局部更新 LVGL 控件。**

#### 12.1 什么时候用跨页面更新？

| 场景 | 用什么 | 原因 |
|------|--------|------|
| 当前在主菜单，需要更新预览页温度 | `ui_cross_page_update()` | 数据在别的页面 |
| 当前在设置页，需要更新预览页商品名 | `ui_cross_page_update()` | 数据在别页面 |
| 当前在预览页，需要更新预览页温度 | `ui_partial_update()` | 数据在当前页面，不需要切页 |

#### 12.2 使用步骤

```c
/* 第 1 步：准备好新数据 */
int16_t new_temp = 26;

/* 第 2 步：调用跨页面更新函数 */
/* 参数说明：
 *   &ui_ctx                  - UI 上下文指针
 *   UI_STATE_LABEL_PREVIEW   - 目标页面状态（数据要更新到哪个页面）
 *   PREVIEW_ITEM_TEMPERATURE - 目标页面上的区块编号
 *   &new_temp                - 新数据指针
 *   0                        - data_len
 */
ui_cross_page_update(&ui_ctx, UI_STATE_LABEL_PREVIEW,
                     PREVIEW_ITEM_TEMPERATURE, &new_temp, 0);

/* 执行流程：
 * 1. 检查当前是否在 UI_STATE_LABEL_PREVIEW
 * 2. 如果不在 → 调用 ui_state_machine_switch() 切过去
 * 3. 调用 ui_partial_update() 局部更新温度标签
 * 4. 自动设 need_refresh = 1，lv_task_handler() 刷新显示
 */
```

#### 12.3 跨页面更新 vs 手动切页+局部更新

```c
/* 方式 1：用跨页面更新函数（推荐，一行搞定） */
ui_cross_page_update(&ui_ctx, UI_STATE_LABEL_PREVIEW,
                     PREVIEW_ITEM_TEMPERATURE, &new_temp, 0);

/* 方式 2：手动切页 + 局部更新（效果一样，但要写两行） */
if (ui_ctx.current_state != UI_STATE_LABEL_PREVIEW) {
    ui_state_machine_switch(&ui_ctx, UI_STATE_LABEL_PREVIEW);
}
ui_partial_update(&ui_ctx, PREVIEW_ITEM_TEMPERATURE, &new_temp, 0);
```

> 大白话：`ui_cross_page_update()` 就是把"切页 + 局部更新"两步合成了一步，写起来更方便。

#### 12.4 外部传感器数据如何跨页面更新（LVGL 版本）

```c
/* 传感器任务：每 1 秒读一次温度 */
void Task_Sensor(void *arg)
{
    while (1) {
        int16_t temp = DHT11_ReadTemperature();

        /* 不管当前在哪个页面，都强制更新到预览页 */
        ui_cross_page_update(&ui_ctx, UI_STATE_LABEL_PREVIEW,
                             PREVIEW_ITEM_TEMPERATURE, &temp, 0);

        vTaskDelay(1000);
    }
}
```

> 大白话：如果希望传感器数据变化时，不管用户在哪个页面，都强制切到目标页面并更新 LVGL 控件显示，就用 `ui_cross_page_update()`。

### 十三、局部更新与跨页面更新流程图（LVGL 版本）

```
外部传感器数据变化
    ↓
┌─────────────────────────────────────┐
│ 判断：数据要更新到哪个页面？          │
└──────────────┬──────────────────────┘
               │
       ┌───────┴───────┐
       ↓               ↓
  当前页面          其他页面
       │               │
       ↓               ↓
 ui_partial_update()  ui_cross_page_update()
 （局部更新 LVGL 控件）│
       │               ↓
       │        ┌──────────────────────┐
       │        │ 当前是否在目标页面？   │
       │        └──────┬───────────────┘
       │               │
       │        ┌──────┴──────┐
       │        ↓             ↓
       │       是            否
       │        │             │
       │        │             ↓
       │        │    ui_state_machine_switch()
       │        │    （先切换到目标页面）
       │        │             │
       │        └──────┬──────┘
       │               ↓
       │        ui_partial_update()
       │        （局部更新 LVGL 控件）
       │               │
       └───────┬───────┘
               ↓
    ctx->need_refresh = 1
    ctx->partial_update_flag = 1
    ctx->partial_item_id = item_id
               ↓
    UI 任务检测到 need_refresh == 1
               ↓
    ui_update_display() → ui_draw_xxx()
               ↓
    ui_draw_xxx() 检测 partial_update_flag == 1
               ↓
    只更新指定 LVGL 控件文本（lv_label_set_text）
               ↓
    lv_scr_load_anim() 切换页面
               ↓
    lv_task_handler() 让 LVGL 真正刷新屏幕
               ↓
    清除 partial_update_flag = 0
```

### 十四、扩展接口函数使用（LVGL 版本）

#### 14.1 页面栈操作（用于多级返回）

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

#### 14.2 页面生命周期管理（LVGL 特有）

**大白话：LVGL 页面有创建、进入、刷新、退出、销毁五个阶段，需要手动管理。**

```c
/*
 * 页面生命周期钩子
 * 大白话：进入页面时启动定时器，离开页面时暂停定时器
 * 避免后台页面浪费 CPU 资源
 */
typedef struct {
    ui_state_t state;
    bool (*create)(ui_context_t *ctx);   /* 首次创建控件 */
    bool (*enter)(ui_context_t *ctx);    /* 进入页面时调用 */
    bool (*refresh)(ui_context_t *ctx);  /* 刷新数据 */
    bool (*exit)(ui_context_t *ctx);     /* 离开页面时调用 */
    bool (*destroy)(ui_context_t *ctx);  /* 销毁控件释放内存 */
} ui_page_ops_t;

/* 使用示例 */
static lv_timer_t *gauge_timer = NULL;

static void gauge_page_enter(ui_context_t *ctx)
{
    /* 进入仪表盘页面时启动定时器 */
    if (gauge_timer == NULL) {
        gauge_timer = lv_timer_create(gauge_update_cb, 1000, NULL);
    }
}

static void gauge_page_exit(ui_context_t *ctx)
{
    /* 离开仪表盘页面时暂停定时器 */
    if (gauge_timer != NULL) {
        lv_timer_pause(gauge_timer);
    }
}
```

#### 14.3 线程安全（RTOS 环境强制要求）

**大白话：LVGL 不是线程安全的，多个任务同时操作 LVGL 对象会崩溃！必须加锁！**

```c
/* 在 LVGL 初始化时创建互斥锁 */
SemaphoreHandle_t lvgl_mutex = xSemaphoreCreateRecursiveMutex();

/* 在任何任务中操作 LVGL 对象前加锁 */
xSemaphoreTakeRecursive(lvgl_mutex, portMAX_DELAY);
lv_label_set_text(label, "New Value");
lv_obj_set_style_bg_color(btn, lv_color_red(), 0);
xSemaphoreGiveRecursive(lvgl_mutex);

/* 或者用 lvgl_port 提供的封装 */
lvgl_port_lock();
lv_obj_t *scr = lv_disp_get_scr_act(NULL);
lvgl_port_unlock();
```

**不加锁的后果：**
- UI 任务和数据更新任务同时操作同一个 LVGL 对象
- 内存损坏、程序崩溃、屏幕花屏

#### 14.4 数据更新接口

```c
/**
 * @brief 更新预览页面数据
 * @param ctx UI 上下文指针
 * @param name 商品名称
 * @param date 日期
 * @param count 数量
 * @return true 更新成功
 * @note 作用：其他任务通过这个函数更新预览页数据，不要直接操作 ctx
 */
bool ui_update_preview_data(ui_context_t *ctx, const char *name,
                            const char *date, uint8_t count)
{
    if (ctx == NULL || name == NULL) {
        return false;
    }

    /* RTOS 环境：加锁保护 */
    xSemaphoreTakeRecursive(lvgl_mutex, portMAX_DELAY);

    strncpy(ctx->preview_data.name, name, sizeof(ctx->preview_data.name) - 1);
    strncpy(ctx->preview_data.date, date, sizeof(ctx->preview_data.date) - 1);
    ctx->preview_data.count = count;
    ctx->need_refresh = 1;  /* 数据变了，标记刷新 */

    xSemaphoreGiveRecursive(lvgl_mutex);

    return true;
}
```

### 十五、完整接入流程图（LVGL 版本）

```
┌──────────┐    按键/触摸     ┌──────────┐
│  硬件输入 │ ──────────────→ │ 消息队列  │
└──────────┘                 └────┬─────┘
                                  │
                                  ↓
                          ┌──────────────┐
                          │  UI 任务      │
                          │ (RTOS 线程)   │
                          └──────┬───────┘
                                 │
                    ┌────────────┼────────────┐
                    ↓            ↓            ↓
              取消息        翻译事件       调用状态机
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
                    │ ui_draw_xxx()             │
                    │  创建/更新 LVGL 控件       │
                    │  lv_scr_load_anim()       │
                    └────────────┬────────────┘
                                 ↓
                    ┌─────────────────────────┐
                    │ lv_task_handler()         │
                    │  LVGL 真正刷新屏幕        │
                    │  need_refresh = 0          │
                    └─────────────────────────┘
```

### 十六、LVGL 特有注意事项

#### 16.1 内存管理（ESP32 内存紧张）

```c
/* lv_conf.h 配置建议 */
#define LV_COLOR_DEPTH 16              /* 16 位色深，省内存 */
#define LV_MEM_SIZE (64 * 1024)        /* LVGL 内存池 64KB */
#define LV_DISP_DEF_REFR_PERIOD 33     /* 刷新周期 33ms ≈ 30fps */

/* 内存优化技巧 */
1. 非活跃页面的定时器该暂停就暂停（lv_timer_pause）
2. 大控件（如 lv_chart 曲线缓冲区）在非活跃时主动释放
3. 用 PSRAM 存图片资源（lv_img_set_src 支持 PSRAM）
4. 避免同时创建太多页面，按需创建、用完销毁
5. 用 lv_obj_clean(screen) 清除页面内容后复用对象
```

**内存占用参考：**
- 3~5 个简单页面：约 120KB RAM
- 3~5 个复杂页面（含图表、曲线）：约 200KB~300KB RAM
- 超过 400KB 时要警惕内存不足

#### 16.2 双核分工（ESP32 特有）

```
Core 0（PRO_CPU）：
  - FreeRTOS 内核任务
  - WiFi / 蓝牙协议栈
  - 其他后台任务

Core 1（APP_CPU）：
  - LVGL 主线程（lv_timer_handler + 渲染）
  - UI 任务
  - 传感器数据采集（如果不影响 UI 帧率）
```

**好处**：LVGL 渲染和网络通信互不干扰，减少卡顿。

### 十七、常见操作速查表（LVGL 版本）

| 我想... | 怎么做 |
|---------|--------|
| 新增一个页面 | 枚举加状态 → 写绘制函数（创建 LVGL 控件）→ update_display 加 case → dispatch 加跳转 |
| 改开机起始页 | 改 `ui_state_machine_init()` 里的 `ctx->current_state` |
| 从 A 页跳到 B 页 | 在 dispatch 里 A 的 case 下调用 `ui_state_machine_switch(ctx, UI_STATE_B)` |
| 加一个新按键 | `ui_key_to_event()` 加 case → dispatch 里加处理逻辑 |
| 加触摸屏支持 | 注册到 LVGL（方案 B）或走状态机（方案 D） |
| 往页面传数据 | 写数据到 `ctx->xxx_data` → 设 `need_refresh = 1` |
| 刷新当前页面 | 设 `ctx->need_refresh = 1`，UI 任务会自动画 |
| 返回上一级 | dispatch 里调 `ui_state_machine_switch(ctx, ctx->previous_state)` 或用页面栈 |
| 移动选中项 | dispatch 里改 `ctx->selected_item` + 设 `need_refresh = 1` |
| 操作 LVGL 对象 | RTOS 环境必须加互斥锁保护 |
| 管理页面生命周期 | 进入页面启动定时器，离开页面暂停定时器 |
| 省内存 | 非活跃页面暂停定时器，大控件用完释放 |
| 局部更新当前页面数据 | 调用 `ui_partial_update(ctx, item_id, data, data_len)` |
| 跨页面更新数据 | 调用 `ui_cross_page_update(ctx, target_state, item_id, data, data_len)` |
| 传感器数据更新到 UI | 当前页用 `ui_partial_update()`，其他页用 `ui_cross_page_update()` |

---

## 修改前检查清单

动手改代码前必须确认：

- 已确认工程中使用了 LVGL 图形库
- 已确认 `lvgl.h`、`lv_timer_handler()`、screen 切换方式、输入设备接入方式和事件 callback 位置
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
- **没有使用 `goto`**
- **新增函数都有中文大白话注释**
- **新增枚举值、结构体字段都有中文大白话注释**
- 新增日志为英文
- `NULL` 参数路径有明确返回值
- 页面绘制函数返回值与调用方式匹配

---

## UI 框架使用指南（大白话版）

### 这个框架到底是干什么的？

**一句话：让嵌入式 UI 代码不乱。**

没有这个框架时，你的按键处理、页面切换、画面绘制可能全混在一起，页面一多就改不动。

有了这个框架后：
- **按键** → 翻译成统一事件
- **状态机** → 决定下一步干什么
- **页面函数** → 只管画画面
- 各司其职，互不干扰

### 框架组件说明

| 组件 | 干什么的 | 你需要做什么 |
|------|---------|-------------|
| `ui_event_t` 枚举 | 定义所有可能的 UI 事件 | 根据硬件输入增减事件类型 |
| `ui_state_t` 枚举 | 定义所有页面 | 每新增一个页面就加一项 |
| `ui_context_t` 结构体 | 存所有 UI 状态信息 | 一般不用改，除非需要存新数据 |
| `ui_state_machine_init()` | 初始化 UI 状态 | 开机时调用一次 |
| `ui_state_machine_dispatch()` | 处理 UI 事件 | 每次有按键事件时调用 |
| `ui_state_machine_switch()` | 切换页面 | 在 dispatch 里调用，不要直接改状态 |
| `ui_update_display()` | 刷新画面 | 主循环里每次都调用 |
| `ui_draw_xxx()` | 画具体页面 | 每个页面写一个，在 update_display 里注册 |

### 如何使用（分步指南）

**RTOS + LVGL 工程**

```c
void Task_LvglUI(void *arg)
{
    ui_msg_t msg;
    ui_state_machine_init(&ui_ctx);

    while (1) {
        if (xQueueReceive(ui_msg_queue, &msg, 0) == pdPASS) {
            switch (msg.type) {
                case UI_MSG_KEY_EVENT:
                    ui_event_t evt = ui_key_to_event(msg.key_code);
                    ui_state_machine_dispatch(&ui_ctx, evt, msg.param);
                    break;
                case UI_MSG_RTC_TIME:
                    ui_update_time(&ui_ctx, msg.param);
                    ui_ctx.need_refresh = 1;
                    break;
            }
        }

        if (ui_ctx.need_refresh) {
            ui_update_display(&ui_ctx);
        }

        lv_task_handler();
        vTaskDelay(pdMS_TO_TICKS(5));
    }
}
```

### 如何新增一个页面

1. 在 `ui_state_t` 枚举里加一项
2. 写一个 `ui_draw_xxx(ctx)` 绘制函数
3. 在 `ui_update_display()` 的 switch 里加一个 case
4. 在 `ui_state_machine_dispatch()` 里加跳转逻辑

### 如何处理新的按键

1. 在 `ui_key_to_event()` 或等价函数里加一个 case
2. 把硬件按键码翻译成已有的 `ui_event_t`（比如 `UI_EVT_OK`）
3. 如果需要全新的事件类型，在 `ui_event_t` 枚举里加一项
4. 在 `ui_state_machine_dispatch()` 里加处理逻辑

### 如何从页面 A 跳转到页面 B

```c
/* 在 ui_state_machine_dispatch() 里，页面 A 的 case 下 */
case UI_STATE_A:
    if (event == UI_EVT_OK) {
        return ui_state_machine_switch(ctx, UI_STATE_B);
    }
    break;
```

**不要**直接在页面绘制函数里调 `ui_state_machine_switch()`，跳转逻辑应该集中在状态机里。

### 流程图总结

```
┌──────────┐    按键/触摸     ┌──────────┐
│  硬件输入 │ ──────────────→ │ 消息队列  │
└──────────┘                 └────┬─────┘
                                  │
                                  ↓
                          ┌──────────────┐
                          │ UI 任务/主循环 │
                          └──────┬───────┘
                                 │
                    ┌────────────┼────────────┐
                    ↓            ↓            ↓
              取消息        翻译事件       调用状态机
                    │            │            │
                    └────────────┼────────────┘
                                 ↓
                    ┌────────────────────────┐
                    │ ui_state_machine_dispatch│
                    │  决定：切页/移焦点/刷新   │
                    └────────────┬───────────┘
                                 ↓
                    ┌────────────────────────┐
                    │ ui_state_machine_switch  │
                    │  更新 current_state      │
                    │  设 need_refresh = 1     │
                    └────────────┬───────────┘
                                 ↓
                    ┌────────────────────────┐
                    │ ui_update_display()      │
                    │  根据 current_state      │
                    │  调用 ui_draw_xxx()      │
                    └────────────┬───────────┘
                                 ↓
                    ┌────────────────────────┐
                    │ lv_task_handler() (LVGL) │
                    │ 或 LCD 刷新完成           │
                    └────────────────────────┘
```

---

## 页面命名与拼写

处理已有工程时不要贸然全局改名。比如工程中已有 `UI_STATE_SETING`、`ui_draw_seting()`、`ui_seting_data_t`，除非用户明确要求统一拼写，否则继续沿用，避免引入大范围编译错误。

新工程可使用标准拼写：

- `UI_STATE_SETTING`
- `ui_draw_setting()`
- `ui_setting_data_t`

---

## UI 公共接口规范（强制要求）

**所有 UI 模块必须通过公共接口与外界交互，其他组件不能直接调用页面绘制函数。**

### 公共接口设计原则

1. **单一入口**：所有显示更新都通过 `ui_update_display()` 一个函数进入
2. **上下文传递**：所有数据都通过 `ui_context_t` 结构体传递，不依赖全局变量
3. **状态驱动**：根据 `current_state` 决定显示哪个页面，外部不需要知道具体页面函数
4. **数据载荷**：每个页面的数据独立封装在 `ui_context_t` 中，页面间数据隔离
5. **LVGL 对象管理**：所有 `lv_obj_t` 对象都在页面函数内部管理，外部不直接操作

### 公共接口头文件示例

```c
/******************* UI 接口头文件 *******************/
#ifndef __ui_interface_H__
#define __ui_interface_H__

#include <stdint.h>
#include <stdbool.h>
#include "lvgl.h"  /* LVGL 工程需要包含 */

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
    
    /* LVGL 特有：页面切换动画 */
    lv_scr_load_anim_t anim_type;  /* 页面切换动画类型 */
    uint32_t anim_time;            /* 动画时长（毫秒） */
} ui_context_t;

/* ========== 公共接口函数 ========== */
/**
 * @brief 根据当前状态刷新屏幕
 * @param ctx UI 上下文结构体指针
 * @return 成功返回 true；ctx 为 NULL 返回 false
 * @note 作用：这是 UI 模块的唯一入口，其他组件只需要调用这个函数
 *       内部会根据 current_state 自动调用对应页面的绘制函数
 *       LVGL 工程会自动处理 lv_scr_load() 和动画
 */
bool ui_update_display(ui_context_t *ctx);

#endif /* __ui_interface_H__ */
```

### 公共接口实现示例（LVGL 版本）

```c
/* ========== 显示更新函数实现（LVGL 版本） ========== */
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
 *       LVGL 版本会自动处理页面切换和动画
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
    bool result = false;
    switch (ctx->current_state) {
        case UI_STATE_MAIN_MENU:
            result = ui_draw_main_menu(ctx);
            break;
        case UI_STATE_LABEL_PREVIEW:
            result = ui_draw_label_preview(ctx);
            break;
        case UI_STATE_DATE:
            result = ui_draw_date(ctx);
            break;
        case UI_STATE_SETING:
            result = ui_draw_seting(ctx);
            break;
        default:
            printf("UI render skipped: unknown state=%d\r\n", ctx->current_state);
            break;
    }

    /* 画完了，清除刷新标志 */
    if (result) {
        ctx->need_refresh = 0;
    }

    return result;
}

/* ========== LVGL 页面绘制函数示例 ========== */
/**
 * @brief 画主菜单页面（LVGL 版本）
 * @param ctx UI 上下文指针
 * @return true 画好了，false 画失败了
 * @note 作用：创建或更新主菜单的 LVGL 控件
 *       第一次进入时创建控件，之后只更新选中样式
 *       所有 lv_obj_t 都在这个函数内部管理，外部不直接操作
 */
bool ui_draw_main_menu(ui_context_t *ctx)
{
    /* 第一次进入这个页面时创建控件 */
    static lv_obj_t *main_menu_screen = NULL;
    static lv_obj_t *menu_labels[3] = {NULL};
    
    if (main_menu_screen == NULL) {
        /* 创建页面容器 */
        main_menu_screen = lv_obj_create(NULL);
        
        /* 创建标题 */
        lv_obj_t *title = lv_label_create(main_menu_screen);
        lv_label_set_text(title, "Main Menu");
        lv_obj_align(title, LV_ALIGN_TOP_MID, 0, 10);
        
        /* 创建菜单列表 */
        const char *menu_items[] = {"Preview", "Setting", "Exit"};
        for (int i = 0; i < 3; i++) {
            menu_labels[i] = lv_label_create(main_menu_screen);
            lv_label_set_text(menu_labels[i], menu_items[i]);
            lv_obj_align(menu_labels[i], LV_ALIGN_TOP_LEFT, 10, 30 + i * 30);
        }
    }
    
    /* 更新选中样式：选中的项加粗，其他正常 */
    for (int i = 0; i < 3; i++) {
        if (i == ctx->selected_item) {
            lv_obj_set_style_text_font(menu_labels[i], &lv_font_montserrat_16, 0);
        } else {
            lv_obj_set_style_text_font(menu_labels[i], &lv_font_montserrat_14, 0);
        }
    }
    
    /* 切换到这个页面（带淡入动画） */
    lv_scr_load_anim(main_menu_screen, LV_SCR_LOAD_ANIM_FADE_ON, 200, 0, false);
    
    return true;
}
```

### 其他组件如何调用 UI（LVGL 版本）

```c
/* 大白话：其他组件（比如按键处理、传感器数据更新）只需要这样调用 UI： */

/* 步骤 1：修改数据 */
ui_ctx.preview_data.l_print_page = 5;
strcpy(ui_ctx.preview_data.l_type, "COOKED 01 JUN 2025");

/* 步骤 2：标记需要刷新 */
ui_ctx.need_refresh = 1;

/* 步骤 3：调用公共接口 */
ui_update_display(&ui_ctx);

/* 步骤 4：让 LVGL 处理渲染（在 UI 任务主循环中） */
lv_task_handler();

/* 不需要知道当前在哪个页面，也不需要直接调用 ui_draw_xxx() */
/* ui_update_display() 内部会自动根据 current_state 调用正确的页面函数 */
/* LVGL 会自动处理页面切换动画和控件刷新 */
```

### 公共接口的好处（LVGL 版本）

1. **解耦**：其他组件不需要知道 UI 内部有哪些页面、页面函数叫什么
2. **安全**：所有数据都通过 `ui_context_t` 传递，不会出现野指针
3. **统一**：所有显示更新都走同一个入口，方便调试和追踪
4. **易扩展**：新增页面只需要在 `ui_update_display()` 里加一个 case，外部调用代码不用改
5. **LVGL 对象管理**：所有 `lv_obj_t` 都在页面函数内部管理，避免外部直接操作导致的生命周期混乱
6. **线程安全**：通过 `ui_update_display()` 统一入口，可以在这里加互斥锁保护 LVGL 操作

### 公共接口约束（LVGL 版本）

- **禁止**其他组件直接调用 `ui_draw_xxx()` 页面绘制函数
- **禁止**绕过 `ui_update_display()` 直接调用 `lv_scr_load()` 或操作 `lv_obj_t`
- **必须**通过 `ui_context_t` 传递数据，不要依赖全局变量
- **必须**在修改数据后设置 `need_refresh = 1`，否则画面不会更新
- **必须**在 RTOS 环境中，所有 LVGL 操作都要加互斥锁保护
- **必须**在页面函数内部管理 `lv_obj_t` 的生命周期，不要在外部删除或修改控件

---

## 输出给用户的总结要求

完成任务后，用大白话总结：

1. 改了哪些文件
2. 每个文件主要改了什么
3. 为什么这样改
4. 有没有引入新变量或新函数
5. 复核时重点看了什么
6. 是否运行了编译或检查命令，结果如何
7. 有没有引入新的库或头文件
8. 生成一个UI界面使用指南及其流程框架的.md文件，**必须包含以上"UI 框架完整使用指南（LVGL 版本 - 大白话版）"中的所有核心章节**（页面注册4步流程、起始页设定、多页面注册、按键/触摸接入、数据传递、刷新机制、关键参数说明、扩展接口、流程图、速查表等），不能省略

总结要直接、具体，不要只说"已优化"。
