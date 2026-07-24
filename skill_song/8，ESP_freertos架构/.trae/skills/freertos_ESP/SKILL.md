---
name: "freertos-architect"
description: "专为 ESP 系列芯片（ESP32/ESP32-S2/S3/C3/C6/H2 等）生成 ESP-IDF FreeRTOS 多任务架构框架。当用户需要为 ESP 系列创建 FreeRTOS 项目架构、规划任务划分、设计消息队列通信、集成传感器驱动时使用。支持 UI 框架适配、传感器集成、第三方库整合，自动生成清晰的文件结构和 CMake 配置。仅限 ESP 系列芯片使用。"
---

# ESP 系列 FreeRTOS 架构生成器

## 适用范围

**本 Skill 专门用于 ESP 系列芯片**：
- ESP32 / ESP32-S2 / ESP32-S3
- ESP32-C2 / ESP32-C3 / ESP32-C6
- ESP32-H2 / ESP32-P4
- 其他 ESP 系列芯片

**开发框架**：ESP-IDF (v4.4 / v5.x)

## 功能概述

本 Skill 专为 ESP 系列芯片生成清晰、可维护的 FreeRTOS 多任务架构框架。核心特点：
- **UI 框架适配**：不改变原有 UI 框架，通过外部接口调用
- **app_arch 封装层**：提供线程安全的 UI 局部更新接口，外部任务无需直接操作 UI 消息队列
- **任务最小化**：合并同类功能，减少任务数量
- **传感器集成**：按需集成用户指定的传感器和第三方库
- **清晰分层**：任务层、app_arch 层、中间层、驱动层、组件层分离
- **低耦合设计**：通过消息队列解耦，避免直接跨层调用
- **完整文档**：自动生成 CMake 配置和中文使用指南

---

## 4 步核心流程

> **总览**：1. 询问需求 → 2. 生成架构 → 3. 创建文件 → 4. 写使用指南

---

### 第一步：询问需求

> 在动手之前，必须先把用户的需求摸清楚。这一步包含两件事：检测现有架构 + 收集功能需求。

#### 1.1 检测现有架构并决策（必须先执行）

在开始任何架构规划前，**必须先扫描当前项目代码**，判断是否已经存在 FreeRTOS 架构或任务代码。

**检测方法**：
- 搜索项目中是否包含 `xTaskCreate`、`xQueueCreate`、`TaskHandle_t` 等 FreeRTOS API
- 搜索是否已有 `tasks/`、`middleware/`、`drivers/` 等架构目录
- 搜索 `app_main.c` 中是否已有任务创建逻辑
- **重点检测 LVGL 相关任务**：搜索 `esp_lvgl_port`、`lvgl_port`、`lv_task_handler`、`lvgl_port_lock`、`lvgl_port_unlock` 等关键字，判断项目中是否已有 LVGL 显示任务

**如果发现已有架构**，必须暂停，向用户输出以下分析（用大白话），让用户选择：

---

**示例输出模板**（根据实际代码情况填写）：

> 我检测到你的项目中已经有一套 FreeRTOS 架构，大致情况如下：
> - 现有 X 个任务：Task_A（负责...）、Task_B（负责...）
> - 现有消息队列：queue_xxx
> - 现有文件结构：tasks/xxx.c ...
>
> 现在你想加入的新功能是：（列出新功能）
>
> 这里有两条路可以走，我用大白话给你说清楚：
>
> **方案一：在现有架构上合并改造**
> - 优点：不用推翻重来，已有代码不用大改，省时间
> - 缺点：新旧代码风格可能不一致，可能需要打补丁适配，后期维护可能有点乱
> - 适合：现有架构和新架构差别不大，任务划分逻辑差不多
>
> **方案二：不合并，重新生成全新架构**
> - 优点：代码风格统一，结构清晰，后期好维护
> - 缺点：已有的代码需要迁移过来，工作量稍大
> - 适合：现有架构和新架构差别很大，或者现有架构本身不太合理
>
> **方案三：保留现有架构，只补充缺失的部分**
> - 优点：改动最小，只加新东西，不动旧代码
> - 缺点：新旧架构可能并存，风格不统一，中间件可能有冗余
> - 适合：现有架构基本能用，只是缺一些功能模块
>
> 你想选哪个方案？或者你有其他想法？

---

**等待用户回复后**，根据用户选择执行：
- **选方案一（合并）**：读取现有代码，将新功能合并进现有架构，调整任务划分
- **选方案二（重建）**：生成全新架构，提示用户手动迁移旧代码
- **选方案三（补充）**：只生成缺失的文件和模块，不改动现有文件

**如果未检测到已有架构**，跳过此步骤，直接进入 1.2 收集功能需求。

#### 1.2 收集功能需求（必须询问用户）

在生成架构前，**必须**向用户确认以下信息：

1. **UI 框架信息**：
   - 是否已有 UI 框架？（LVGL、TFT_eSPI 等）
   - UI 框架是否提供外部调用接口？
   - UI 任务优先级需求
   - ⭐ **项目中是否已有 LVGL 任务？**（如 `esp_lvgl_port`、`lvgl_port_task` 等）
     - **如果已有 LVGL 任务** → 不新建 UI 任务，直接复用现有 LVGL 任务，在其中添加消息接收逻辑
     - **如果没有 LVGL 任务** → 新建 1 个独立的 UI 任务（Task_LvglUI）
   - ⭐⭐ **强制询问：UI 框架的接口文件在哪里？**（必须执行）
     - **必须扫描项目代码**，找到 UI 框架的接口文件位置（如 `ui_interface.h`、`ui_api.h`、`lvgl_port.h` 等）
     - **必须向用户确认**：UI 框架的接口文件在哪个路径？接口函数有哪些？
     - **等待用户回复后**，将 `app_arch` 层与用户确认的 UI 框架接口绑定
     - **绝对不改变原 UI 框架的内容**，只在 `app_arch` 层调用 UI 框架提供的外部接口
     - **示例输出**：
       > 我检测到你的项目中已有 UI 框架，接口文件在 `ui/ui_interface.h`，包含以下接口：
       > - `ui_update_label(label_id, text)` → 更新标签文字
       > - `ui_update_image(img_id, data)` → 更新图片
       > 
       > 请确认：
       > 1. 这些接口是否正确？
       > 2. 还有其他需要暴露的接口吗？
       > 3. 接口文件路径是 `ui/ui_interface.h` 吗？
       > 
       > 确认后，我会将 `app_arch` 层与这些接口绑定，不修改原 UI 框架代码。

2. **传感器列表**：
   - 需要集成哪些传感器？（DS1302、DHT11、BMP280、ADC 电池检测等）
   - 每个传感器的读取频率？（实时、1 秒、10 秒等）

3. **第三方库/组件**：
   - 需要集成哪些第三方库？（MQTT、WiFi、蓝牙、文件系统 FatFS 等）
   - 是否有外部 Flash/EEPROM 存储需求？

4. **执行器/输出设备**：
   - 是否有打印机、电机、继电器等执行器？
   - 执行器是否需要独立任务？（如热敏打印需要阻塞任务）

5. **通信需求**：
   - 任务间是否需要消息队列？
   - 是否需要事件组或信号量同步？

6. **ESP 芯片型号**：
   - 具体哪款 ESP 芯片？（ESP32 / ESP32-S3 / ESP32-C3 / ESP32-C6 / ESP32-H2 等）
   - ESP-IDF 版本？（v4.4 / v5.0 / v5.1 / v5.2 等）
   - 是否有 PSRAM？（影响任务栈大小分配）
   - Flash 大小？（4MB / 8MB / 16MB，影响分区表设计）

**需求收集完成后，进入第二步。**

---

### 第二步：生成架构

> 根据第一步收集到的需求，设计完整的 FreeRTOS 架构方案，包括任务划分、消息队列、代码模板。这一步完成后会输出一份架构设计方案给用户确认，确认后再进入第三步创建文件。

#### 2.1 UI 框架适配策略

**⭐⭐ 强制要求：必须先找到 UI 框架的接口文件并与用户确认**
- 在生成架构前，**必须扫描项目代码**，找到 UI 框架的接口文件位置
- **必须向用户确认**接口文件路径和接口函数列表
- **等待用户确认或补充后**，才能将 `app_arch` 层与 UI 框架接口绑定
- **绝对不改变原 UI 框架的内容**，只在 `app_arch` 层调用 UI 框架提供的外部接口

**场景 A：已有 UI 框架且提供外部接口**
- 以 UI 框架接口为主导
- `app_arch` 层通过 UI 提供的 API 更新界面
- 示例：LVGL 的 `lv_label_set_text()` 只能在 UI 任务中调用
- **必须确认接口文件位置**：如 `ui/ui_interface.h`，包含哪些接口函数

**场景 B：已有 UI 框架但无外部接口**
- 创建 UI 适配层中间件
- 其他任务通过消息队列发送 UI 更新请求
- UI 任务接收消息后调用 UI 框架 API
- **必须确认**：UI 框架的哪些函数可以被外部调用？

**场景 C：无 UI 框架**
- 创建独立的 UI 任务，集成 LVGL 或其他框架
- UI 任务负责初始化和刷新

**场景 D：项目中已有 LVGL 任务（如 esp_lvgl_port）** ⭐ 重要
- **绝对不要新建 Task_UI 或类似的 UI 任务**
- 必须以项目中已有的 LVGL 任务为主（如 `esp_lvgl_port` 任务）
- 其他任务通过消息队列或事件组将 UI 更新请求发送给 LVGL 任务
- LVGL 任务负责接收消息并调用 LVGL API 更新界面
- 示例：如果项目中有 `esp_lvgl_port` 任务，应该：
  1. 保留 `esp_lvgl_port` 任务作为唯一的 UI 任务
  2. 在 `esp_lvgl_port` 任务中添加消息接收逻辑
  3. 其他任务通过消息队列发送 UI 更新请求给 `esp_lvgl_port`
  4. `esp_lvgl_port` 任务接收消息后调用 `lv_label_set_text()` 等 LVGL API
- **必须确认**：LVGL 相关的接口文件在哪里？用户是否需要暴露额外的接口？

**判断是否有 LVGL 任务的方法**：
- 搜索 `esp_lvgl_port`、`lvgl_port_task`、`lv_task_handler` 等关键字
- 检查 `app_main.c` 或 `main.c` 中是否调用了 `esp_lvgl_port_init()` 或类似函数
- 查看是否有专门的任务在调用 `lv_task_handler()`

#### 2.2 任务数量动态决定策略

**核心原则**：任务数量不是固定的，必须根据用户输入的功能需求动态判断。

**⭐⭐ 强制规则：每个任务必须独立成组件**
- **每个新建的任务都必须创建一个独立的 ESP-IDF 组件**
- **组件名 = 任务名**（如：Task_UI → task_ui 组件，Task_SystemPeriph → task_system_periph 组件）
- **每个任务组件必须有独立的 CMakeLists.txt**，配置 ESP32 相关依赖
- **任务组件放在 components/ 目录下**，与其他第三方组件同级
- **示例**：
  - 创建 Task_UI 任务 → 必须创建 `components/task_ui/` 组件
  - 创建 Task_SystemPeriph 任务 → 必须创建 `components/task_system_periph/` 组件
  - 创建 Task_Printer 任务 → 必须创建 `components/task_printer/` 组件

**任务合并原则**：
- 相同优先级、相同 IO 访问模式的功能合并
- 定时采集类传感器合并到一个任务（如 RTC+ADC+温湿度）
- 阻塞型执行器独立成任务（避免阻塞其他功能）
- 通信类功能可独立或合并（根据是否阻塞）

**任务数量判断流程**：
1. **统计用户功能需求**：UI、传感器、执行器、通信
2. **按以下规则合并**：
   - **UI 相关**：
     - ⭐ **如果项目中已有 LVGL 任务（如 esp_lvgl_port）→ 复用该任务，不新建 Task_UI**
     - 如果没有 LVGL 任务 → 新建 1 个 UI 任务
   - 非阻塞传感器/外设 → 合并为 1 个系统外设任务
   - 每个阻塞型执行器 → 独立 1 个任务（打印机、电机等）
   - WiFi/MQTT/蓝牙 → 可独立 1 个通信任务，或合并到系统外设
3. **最小任务数**：1 个（仅 UI 或仅简单轮询）
4. **典型任务数**：2-5 个（根据复杂度）

**示例场景**：

**场景 A：简单项目（2 个任务）**
- 用户需求：按键 + OLED 显示 + LED 控制
- 任务划分：
  1. UI 任务：OLED 显示 + LED 状态
  2. 系统任务：按键扫描

**场景 B：中等项目（3 个任务）**
- 用户需求：LVGL 界面 + 按键 + DS1302 + ADC 电池检测
- 任务划分：
  1. UI 任务：LVGL 界面刷新
  2. 系统外设任务：按键 + DS1302 + ADC
  3. （无阻塞执行器，不需要额外任务）

**场景 C：复杂项目（4 个任务）**
- 用户需求：LVGL 界面 + 按键 + DS1302 + 热敏打印机 + WiFi
- 任务划分：
  1. UI 任务：LVGL 界面刷新
  2. 系统外设任务：按键 + DS1302
  3. 打印任务：热敏打印机（阻塞型）
  4. 通信任务：WiFi + MQTT

**场景 D：极简项目（1 个任务）**
- 用户需求：DHT11 温湿度读取 + 串口输出
- 任务划分：
  1. 主任务：DHT11 读取 + 串口输出（无需 UI，无需解耦）

**场景 E：项目中已有 LVGL 任务（如 esp_lvgl_port）** ⭐ 重要
- 用户需求：LVGL 界面（已有 esp_lvgl_port 任务）+ 按键 + DS1302 + 热敏打印机
- 任务划分：
  1. **复用 esp_lvgl_port 任务**：LVGL 界面刷新 + 消息接收（不新建 Task_UI）
  2. 系统外设任务：按键 + DS1302
  3. 打印任务：热敏打印机（阻塞型）
- **关键点**：
  - 在 esp_lvgl_port 任务中添加消息队列接收逻辑
  - 其他任务通过 ui_msg_queue 发送 UI 更新请求给 esp_lvgl_port
  - 不创建 task_ui.c 文件，不创建 Task_LvglUI 任务

#### 2.3 解耦与合耦策略

**解耦场景**：
- 传感器采集与 UI 显示：通过 `app_arch` 层（`app_arch_ui_update()`）解耦，传感器任务不直接碰 UI 消息队列
- 打印请求与打印执行：通过打印队列解耦
- WiFi 数据与本地处理：通过事件组解耦

**合耦场景**：
- DS1302 和 Flash 都使用 SPI/GPIO：合并到同一任务，避免 IO 竞争
- 按键扫描和 ADC 采样都是定时任务：合并到系统外设任务

#### 2.4 app_arch 线程安全 UI 更新封装层

**⭐⭐ 强制要求：必须与用户确认的 UI 框架接口绑定**
- `app_arch` 层**必须调用用户确认的 UI 框架接口**，不能自己假设接口
- **绝对不改变原 UI 框架的内容**，只在 `app_arch` 层调用 UI 框架提供的外部接口
- 如果用户没有提供 UI 框架接口，**必须先询问用户**，等待用户确认后再继续
- 示例：如果用户确认 UI 框架接口在 `ui/ui_interface.h`，包含 `ui_update_label()` 等函数，则 `app_arch` 层必须调用这些函数

**设计目的**：
- 为外部任务（传感器任务、通信任务等）提供线程安全的 UI 局部更新接口
- 外部任务无需知道 UI 消息队列和 UI 框架的存在，只需调用简单函数
- 降低耦合度：传感器任务不直接操作 `ui_msg_queue`，而是通过 `app_arch` 层封装的接口
- **不改变原 UI 框架内容**：`app_arch` 层只调用 UI 框架提供的外部接口，不修改 UI 框架代码

**核心接口**：
```c
// 在 app_arch/app_arch_ui.h 中定义
typedef enum {
    UI_UPDATE_KEY,          // 按键事件
    UI_UPDATE_RTC,          // RTC 时间
    UI_UPDATE_BATTERY,      // 电池电压
    UI_UPDATE_SENSOR,       // 传感器数据
    UI_UPDATE_CUSTOM,       // 自定义数据
} ui_update_type_t;

typedef struct {
    ui_update_type_t type;
    union {
        uint8_t key_id;
        float voltage;
        rtc_time_t rtc_time;
        sensor_data_t sensor;
        void *custom_data;  // 自定义数据指针
    } data;
} ui_update_msg_t;

// 线程安全的 UI 更新接口（任何任务都可调用）
esp_err_t app_arch_ui_update(ui_update_type_t type, void *data);

// 示例调用（在传感器任务中）：
void Task_SystemPeriph(void *arg)
{
    while(1)
    {
        // 读取传感器数据
        float temp = dht11_read_temp();
        
        // 通过 app_arch 层更新 UI（线程安全）
        app_arch_ui_update(UI_UPDATE_SENSOR, &temp);
        
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

**内部实现**：
```c
// 在 app_arch/app_arch_ui.c 中实现
static QueueHandle_t ui_msg_queue = NULL;

esp_err_t app_arch_ui_update(ui_update_type_t type, void *data)
{
    if (ui_msg_queue == NULL) {
        return ESP_ERR_INVALID_STATE;
    }
    
    ui_update_msg_t msg;
    msg.type = type;
    
    // 根据类型拷贝数据
    switch(type) {
        case UI_UPDATE_KEY:
            msg.data.key_id = *(uint8_t *)data;
            break;
        case UI_UPDATE_BATTERY:
            msg.data.voltage = *(float *)data;
            break;
        case UI_UPDATE_RTC:
            memcpy(&msg.data.rtc_time, data, sizeof(rtc_time_t));
            break;
        case UI_UPDATE_SENSOR:
            memcpy(&msg.data.sensor, data, sizeof(sensor_data_t));
            break;
        default:
            msg.data.custom_data = data;
            break;
    }
    
    // 非阻塞发送到 UI 消息队列
    if (xQueueSend(ui_msg_queue, &msg, 0) == pdPASS) {
        return ESP_OK;
    }
    return ESP_ERR_NO_MEM;  // 队列满，丢弃
}
```

**UI 任务接收**：
```c
void Task_LvglUI(void *arg)
{
    ui_update_msg_t msg;
    while(1)
    {
        if(xQueueReceive(ui_msg_queue, &msg, 0) == pdPASS)
        {
            switch(msg.type)
            {
                case UI_UPDATE_KEY:
                    lv_label_set_text(label_key, key_name[msg.data.key_id]);
                    break;
                case UI_UPDATE_RTC:
                    update_clock_display(msg.data.rtc_time);
                    break;
                case UI_UPDATE_BATTERY:
                    update_battery_display(msg.data.voltage);
                    break;
                case UI_UPDATE_SENSOR:
                    update_sensor_display(msg.data.sensor);
                    break;
                default:
                    break;
            }
        }
        lv_task_handler();
        vTaskDelay(pdMS_TO_TICKS(5));
    }
}
```

**优势**：
- ✅ 外部任务只需调用 `app_arch_ui_update()`，无需知道消息队列细节
- ✅ 线程安全：内部使用 FreeRTOS 消息队列，自动处理并发
- ✅ 解耦：传感器任务不依赖 UI 框架，只依赖 `app_arch` 层
- ✅ 可扩展：新增 UI 更新类型只需在枚举和 switch-case 中添加

#### 2.5 中间件简化原则

- **最多一层中间件**：任务层 → app_arch 层 → 中间层（可选）→ 驱动层
- 中间件职责单一：仅做数据格式转换或消息路由
- 避免多层封装：直接在任务中调用驱动 API
- app_arch 层是可选的，仅在需要跨任务 UI 更新时使用

#### 2.5 ESP32 特有配置（重要）

**栈深度单位说明**：
- ESP-IDF 中 `xTaskCreate` 的栈深度参数单位是**字（Word）**，不是字节
- ESP32/ESP32-S3 中 1 Word = 4 字节
- 示例：`xTaskCreate(task, "name", 2048, ...)` 实际分配 2048 × 4 = 8192 字节 = 8KB
- **常见错误**：把 2048 当成字节，导致栈空间不足，系统重启

**栈大小估算方法**：
- 局部变量开销：大型数组（如 `uint8_t buffer[2048]`）占用栈空间
- 函数调用深度：每个函数调用压入返回地址、保存寄存器
- `printf` 开销巨大：单次 `printf` 可能占用 1-2KB 栈空间
- FreeRTOS 内核开销：约 100-200 字节用于上下文保存
- **推荐策略**：初始值设 2048（8KB），运行后用 `uxTaskGetStackHighWaterMark()` 检测最小剩余值，再增加 20%-50% 余量

**双核任务绑定（ESP32/ESP32-S3）**：
- ESP32 有双核（Core 0 和 Core 1）
- 使用 `xTaskCreatePinnedToCore()` 可将任务绑定到指定核心
- 示例：UI 任务绑定 Core 1，通信任务绑定 Core 0，实现硬件级并行
- 如果不需要双核，使用普通 `xTaskCreate()` 即可，系统自动分配

**任务看门狗配置**：
- ESP-IDF 默认启用任务看门狗（Task Watchdog）
- 如果任务阻塞时间过长（>5 秒），看门狗会触发系统重启
- 在长时间阻塞的任务中，需要定期调用 `vTaskDelay()` 或 `taskWATCHDOG_FEED()` 喂狗
- 可在 `menuconfig` 中调整看门狗超时时间

**内存管理策略**：
- 动态分配（`xTaskCreate`）：适合快速原型，但可能因内存碎片导致分配失败
- 静态分配（`xTaskCreateStatic`）：适合安全关键系统，分配时间确定，不会失败
- **推荐**：重要任务使用静态分配，普通任务使用动态分配

**日志系统使用**：
- 使用 ESP-IDF 的 `ESP_LOGI()`、`ESP_LOGW()`、`ESP_LOGE()` 替代 `printf()`
- 日志级别可配置：Error、Warning、Info、Debug、Verbose
- 日志系统栈开销比 `printf()` 小，且支持按模块过滤

#### 2.6 代码生成规范

##### 2.6.1 禁止事项
- ❌ 禁止使用 `goto` 语句
- ❌ 禁止跨任务直接调用 UI API（如 `lv_label_set_text()`）
- ❌ 禁止外部任务直接操作 UI 消息队列（应通过 `app_arch_ui_update()` 接口）
- ❌ 禁止在多个任务中访问同一 IO（如 DS1302 只能在系统外设任务中访问）
- ❌ 禁止多层中间件封装

##### 2.6.2 推荐做法
- ✅ 使用 `app_arch_ui_update()` 接口更新 UI（外部任务统一调用此接口）
- ✅ 使用 `switch-case` 分支处理消息类型
- ✅ 使用消息队列解耦任务
- ✅ 使用 `xQueueSend(..., 0)` 非阻塞发送（允许丢包的场景）
- ✅ 使用 `xQueueReceive(..., portMAX_DELAY)` 阻塞接收（等待任务）
- ✅ 在 `app_config.h` 中集中定义引脚和任务参数

##### 2.6.3 消息队列设计

**UI 消息队列结构**：
```c
typedef enum {
    UI_MSG_KEY_EVENT,       // 按键事件
    UI_MSG_RTC_TIME,        // RTC 时间
    UI_MSG_BAT_VOLT,        // 电池电压
    UI_MSG_SENSOR_DATA,     // 传感器数据
} ui_msg_type_t;

typedef struct {
    ui_msg_type_t type;
    union {
        uint8_t key_id;
        float bat_vol;
        rtc_time_t rtc_time;
        sensor_data_t sensor;
    } data;
} ui_msg_t;
```

**打印任务队列结构**：
```c
typedef struct {
    char text[256];
    uint8_t print_mode;
} print_job_t;
```

#### 2.7 示例代码模板

##### 任务创建入口模板

> **核分配原则（创建任务前必须确认）：**
> - **单核芯片**（如 ESP32-S2、ESP32-C3 等）：使用 `xTaskCreate()`，所有任务运行在唯一核心上。
> - **多核芯片**（如 ESP32、ESP32-S3 等双核）：使用 `xTaskCreatePinnedToCore()`，必须为每个任务指定运行核心。
>   - **Core 0 (PRO_CPU)**：系统级任务（外设驱动、通信协议栈、看门狗等）。
>   - **Core 1 (APP_CPU)**：用户应用任务（UI、业务逻辑、打印等）。
>   - 若任务需跨核调度或不确定归属，可传入 `tskNO_AFFINITY`。

**单核芯片版本：**
```c
esp_err_t app_arch_init(void)
{
    // 1. 创建消息队列
    ui_msg_queue = xQueueCreate(8, sizeof(ui_msg_t));
    print_job_queue = xQueueCreate(6, sizeof(print_job_t));

    // 2. 创建任务（单核：使用 xTaskCreate）
    xTaskCreate(Task_LvglUI, "Task_LvglUI", 12288, NULL, 5, NULL);
    xTaskCreate(Task_SystemPeriph, "Task_SystemPeriph", 4096, NULL, 3, NULL);
    xTaskCreate(Task_Printer, "Task_Printer", 4096, NULL, 2, NULL);

    return ESP_OK;
}
```

**多核芯片版本（以 ESP32/S3 双核为例）：**
```c
esp_err_t app_arch_init(void)
{
    // 1. 创建消息队列
    ui_msg_queue = xQueueCreate(8, sizeof(ui_msg_t));
    print_job_queue = xQueueCreate(6, sizeof(print_job_t));

    // 2. 创建任务（多核：使用 xTaskCreatePinnedToCore 指定核心）
    //    Core 0 (PRO_CPU)：系统外设、通信等底层任务
    //    Core 1 (APP_CPU)：UI、业务逻辑等应用层任务
    xTaskCreatePinnedToCore(Task_LvglUI,        "Task_LvglUI",        12288, NULL, 5, NULL, 1);  // APP_CPU
    xTaskCreatePinnedToCore(Task_SystemPeriph,   "Task_SystemPeriph",   4096, NULL, 3, NULL, 0);  // PRO_CPU
    xTaskCreatePinnedToCore(Task_Printer,        "Task_Printer",        4096, NULL, 2, NULL, 1);  // APP_CPU

    return ESP_OK;
}
```

##### UI 任务模板

**情况 A：新建 UI 任务（项目中无 LVGL 任务时）**
```c
void Task_LvglUI(void *arg)
{
    ui_msg_t msg;
    while(1)
    {
        if(xQueueReceive(ui_msg_queue, &msg, 0) == pdPASS)
        {
            switch(msg.type)
            {
                case UI_MSG_KEY_EVENT:
                    // 调用 UI 框架接口更新界面
                    break;
                case UI_MSG_RTC_TIME:
                    // 更新时间显示
                    break;
                default:
                    break;
            }
        }
        lv_task_handler();
        vTaskDelay(pdMS_TO_TICKS(5));
    }
}
```

**情况 B：复用现有 LVGL 任务（项目中有 esp_lvgl_port 时）** ⭐ 重要
```c
// ⚠️ 不要新建 Task_LvglUI！
// ⚠️ 应该在现有的 esp_lvgl_port 任务中添加消息接收逻辑

// 示例：在 esp_lvgl_port 任务的循环中添加消息处理
void esp_lvgl_port_task(void *arg)
{
    ui_msg_t msg;
    while(1)
    {
        // 1. 先处理消息队列（新增的逻辑）
        if(xQueueReceive(ui_msg_queue, &msg, 0) == pdPASS)
        {
            switch(msg.type)
            {
                case UI_MSG_KEY_EVENT:
                    // 调用 UI 框架接口更新界面
                    lv_label_set_text(label_key, key_name[msg.data.key_id]);
                    break;
                case UI_MSG_RTC_TIME:
                    // 更新时间显示
                    update_clock_display(msg.data.rtc_time);
                    break;
                default:
                    break;
            }
        }
        
        // 2. 原有的 LVGL 刷新逻辑（保持不变）
        lv_task_handler();
        vTaskDelay(pdMS_TO_TICKS(5));
    }
}

// 在 app_main.c 中：
// ❌ 不要这样做：xTaskCreate(Task_LvglUI, "Task_LvglUI", 12288, NULL, 5, NULL);
// ✅ 应该这样做：直接使用已有的 esp_lvgl_port 任务，只创建其他任务
esp_err_t app_arch_init(void)
{
    // 1. 创建消息队列
    ui_msg_queue = xQueueCreate(8, sizeof(ui_msg_t));
    
    // 2. 只创建其他任务，不创建 UI 任务（因为已有 esp_lvgl_port）
    xTaskCreate(Task_SystemPeriph, "Task_SystemPeriph", 4096, NULL, 3, NULL);
    xTaskCreate(Task_Printer, "Task_Printer", 4096, NULL, 2, NULL);
    
    // 3. esp_lvgl_port 任务已经在其他地方创建，这里不需要再创建
    
    return ESP_OK;
}
```

##### 系统外设任务模板
```c
void Task_SystemPeriph(void *arg)
{
    TickType_t tick_last = xTaskGetTickCount();
    while(1)
    {
        // 按键扫描
        key_scan_process();

        // 定时采集传感器
        if(xTaskGetTickCount() - tick_last > pdMS_TO_TICKS(1000))
        {
            ui_msg_t msg;
            
            // 读取 RTC
            ds1302_read_time(&msg.data.rtc_time);
            msg.type = UI_MSG_RTC_TIME;
            xQueueSend(ui_msg_queue, &msg, 0);

            tick_last = xTaskGetTickCount();
        }

        vTaskDelayUntil(&tick_last, pdMS_TO_TICKS(10));
    }
}
```

**第二步完成后，将架构方案展示给用户确认，确认后进入第三步。**

---

### 第三步：创建文件

> 根据第二步确认的架构方案，创建所有项目文件，包括目录结构、源代码文件和 CMake 配置。

#### 3.1 项目目录结构

根据用户需求动态生成，基础结构如下：

```
project/
├── main/
│   ├── CMakeLists.txt              # ESP-IDF 主 CMake 配置
│   ├── app_main.c                  # 程序入口
│   └── app_config.h                # 全局配置（引脚定义、任务参数）
│
├── components/                     # 所有组件（ESP-IDF 组件管理）
│   ├── lvgl/                       # LVGL 库（如使用）
│   ├── ds1302/                     # DS1302 驱动组件
│   │   ├── CMakeLists.txt
│   │   ├── ds1302.h
│   │   └── ds1302.c
│   ├── dht11/                      # DHT11 驱动组件
│   │   ├── CMakeLists.txt
│   │   ├── dht11.h
│   │   └── dht11.c
│   │
│   ├── task_ui/                    # ⭐ UI 任务组件（组件名 = 任务名）
│   │   ├── CMakeLists.txt          # 任务组件的 CMake 配置
│   │   ├── task_ui.h               # 任务接口定义
│   │   └── task_ui.c               # 任务实现（仅当项目中无 LVGL 任务时创建）
│   │
│   ├── task_system_periph/         # ⭐ 系统外设任务组件（组件名 = 任务名）
│   │   ├── CMakeLists.txt          # 任务组件的 CMake 配置
│   │   ├── task_system_periph.h    # 任务接口定义
│   │   └── task_system_periph.c    # 任务实现
│   │
│   ├── task_printer/               # ⭐ 打印任务组件（组件名 = 任务名）
│   │   ├── CMakeLists.txt          # 任务组件的 CMake 配置
│   │   ├── task_printer.h          # 任务接口定义
│   │   └── task_printer.c          # 任务实现（如有）
│   │
│   └── app_arch/                   # 应用架构层（线程安全 UI 更新封装）
│       ├── CMakeLists.txt
│       ├── app_arch_ui.h           # UI 更新接口定义（外部任务调用）
│       └── app_arch_ui.c           # UI 更新接口实现（内部封装消息队列）
│
├── middleware/                     # 中间件（可选，仅当需要解耦时）
│   ├── CMakeLists.txt
│   ├── msg_queue.h                 # 消息队列定义
│   └── msg_queue.c                 # 消息队列操作封装
│
├── drivers/                        # 硬件驱动（非组件化的驱动）
│   ├── CMakeLists.txt
│   ├── adc_battery.c               # 电池 ADC 采样
│   └── key_scan.c                  # 按键扫描
│
└── docs/
    └── ARCH_GUIDE.md               # 架构使用指南（第四步生成）
```

**⭐⭐ 强制规则说明**：
- **每个任务都是独立的组件**，放在 `components/` 目录下
- **组件名 = 任务名**（小写，下划线分隔）
- **每个任务组件必须包含**：
  - `CMakeLists.txt`：配置 ESP32 相关依赖
  - `task_xxx.h`：任务接口定义（可选，但推荐）
  - `task_xxx.c`：任务实现代码
- **示例**：
  - Task_UI → `components/task_ui/`
  - Task_SystemPeriph → `components/task_system_periph/`
  - Task_Printer → `components/task_printer/`

#### 3.2 CMake 配置生成

为每个目录生成 `CMakeLists.txt`：

**main/CMakeLists.txt 示例**：
```cmake
idf_component_register(
    SRCS "app_main.c"
    INCLUDE_DIRS "."
    REQUIRES task_ui task_system_periph task_printer app_arch middleware drivers lvgl ds1302
)
```

**components/ds1302/CMakeLists.txt 示例**：
```cmake
idf_component_register(
    SRCS "ds1302.c"
    INCLUDE_DIRS "."
    REQUIRES driver
)
```

**app_arch/CMakeLists.txt 示例**：
```cmake
idf_component_register(
    SRCS "app_arch_ui.c"
    INCLUDE_DIRS "."
    REQUIRES middleware freertos
)
```

**⭐⭐ 每个任务组件的 CMakeLists.txt 示例**（强制要求）：

**components/task_ui/CMakeLists.txt**：
```cmake
idf_component_register(
    SRCS "task_ui.c"
    INCLUDE_DIRS "."
    REQUIRES app_arch middleware lvgl freertos
)
```

**components/task_system_periph/CMakeLists.txt**：
```cmake
idf_component_register(
    SRCS "task_system_periph.c"
    INCLUDE_DIRS "."
    REQUIRES app_arch middleware drivers ds1302 freertos
)
```

**components/task_printer/CMakeLists.txt**：
```cmake
idf_component_register(
    SRCS "task_printer.c"
    INCLUDE_DIRS "."
    REQUIRES app_arch middleware freertos
)
```

**⭐⭐ 任务组件 CMake 配置规则**：
- **每个任务组件必须有独立的 CMakeLists.txt**
- **REQUIRES 中必须包含该任务依赖的所有组件**
- **必须包含 `freertos` 依赖**（因为使用了 FreeRTOS API）
- **如果任务需要更新 UI，必须依赖 `app_arch` 组件**
- **如果任务需要访问硬件，必须依赖相应的 `drivers` 或传感器组件**

**文件创建完成后，进入第四步。**

---

### 第四步：写使用指南

> 所有文件创建完成后，生成一份大白话的架构使用指南 `docs/ARCH_GUIDE.md`，让用户一看就懂。

#### 4.1 指南内容结构

生成的 `docs/ARCH_GUIDE.md` 必须包含以下内容：

##### 4.1.1 整体架构说明
- 任务划分逻辑：为什么这样分任务，每个任务负责什么
- **LVGL 任务说明**（重要）：
  - 如果项目中已有 LVGL 任务（如 esp_lvgl_port）→ 说明复用了该任务，不新建 Task_UI
  - 如果没有 LVGL 任务 → 说明新建了 Task_UI
- 消息流向图：用箭头画出消息从哪个任务到哪个任务
- 优先级分配原因：为什么 UI 任务优先级最高，为什么打印任务最低

##### 4.1.2 各组件作用（必须具体到文件）
- **每个传感器/芯片**：写在哪个文件中调用，数据怎么流向 UI
  - 示例：**DS1302 芯片**：在 `task_system_periph.c` 中调用，每秒读取时间并通过 `ui_msg_queue` 发送给 UI 任务
- **每个驱动模块**：写在哪个文件中，被哪个任务调用
  - 示例：**按键扫描**：在 `task_system_periph.c` 中调用，10ms 扫描一次，检测到按键后发送消息到 UI
- **每个执行器**：怎么投递任务，怎么执行
  - 示例：**打印功能**：任何任务调用 `send_print_task()` 投递到 `print_job_queue`，由 `task_printer.c` 阻塞执行

##### 4.1.3 不同组件间框架架构
- 任务层与驱动层的关系：任务通过 `#include` 调用驱动，驱动不反向调用任务
- 任务层与中间层的关系：任务通过消息队列通信，中间层只负责消息定义和路由
- 组件层与驱动层的区别：组件是独立的第三方库（有自己的 CMakeLists），驱动是项目内部的硬件操作代码

##### 4.1.4 框架使用大白话指南（强制输出要求）

> **⚠️ 强制要求：生成 ARCH_GUIDE.md 时，必须实际写出以下内容，不能省略、不能只写标题、不能只写"请参考代码"。必须基于实际生成的代码，用大白话写出完整说明。**

**（1）框架整体使用流程说明（必须输出）**

必须用大白话写出程序启动后的完整执行流程，包含：
- app_main() 里做了什么
- 创建了哪些消息队列
- 创建了哪些任务
- 任务之间怎么协作

输出格式参考（必须按此格式写出实际内容）：
```
程序启动流程（大白话版）：

1. 上电后，系统先跑 app_main()（在 app_main.c 里）
2. app_main() 调用 app_arch_init()，这一步会：
   - 创建消息队列（相当于建好几个"信箱"，任务之间通过信箱传消息）
     * ui_msg_queue：用于传感器数据、按键事件传给 UI
     * print_job_queue：用于打印任务接收打印请求
   - 创建各个任务（相当于雇了几个工人，每人负责不同的活）
     * Task_LvglUI：负责界面刷新
     * Task_SystemPeriph：负责按键扫描、传感器读取
     * Task_Printer：负责热敏打印
3. 任务创建完后，app_main() 就结束了，剩下的全交给 FreeRTOS 调度
4. 各个任务各自循环跑，互不干扰，通过消息队列通信

简单理解：
- app_main() = 包工头，负责招人（创建任务）、建信箱（创建队列）
- 各个 Task = 工人，各干各的活，需要沟通就通过信箱（消息队列）传纸条
```

**（2）消息队列跨任务调用说明（必须输出）**

必须用大白话写出消息队列在任务间怎么传递数据，包含：
- 消息队列是什么（用比喻解释）
- 发送端代码示例（基于实际生成的 task_system_periph.c）
- 接收端代码示例（基于实际生成的 task_ui.c）
- 使用注意点

输出格式参考（必须按此格式写出实际内容）：
```
消息队列怎么用？（大白话版）：

打个比方：消息队列就是一个"信箱"。
- 任务 A 往信箱里塞一封信（xQueueSend）
- 任务 B 从信箱里取出信来看（xQueueReceive）
- 信的内容就是你定义的结构体（比如 ui_msg_t）

【发送端示例】—— 在 task_system_periph.c 中，按键检测到后发消息给 UI：

    void Task_SystemPeriph(void *arg)
    {
        while(1)
        {
            uint8_t key = key_scan();           // 扫描按键
            if(key != 0)                         // 如果检测到按键按下
            {
                ui_msg_t msg;                    // 准备一封信
                msg.type = UI_MSG_KEY_EVENT;     // 信封上写：这是按键事件
                msg.data.key_id = key;           // 信里写：按下了哪个键
                xQueueSend(ui_msg_queue, &msg, 0); // 塞进信箱（非阻塞，信箱满了就丢掉）
            }
            vTaskDelay(pdMS_TO_TICKS(10));       // 休息 10ms 再扫
        }
    }

【接收端示例】—— 在 task_ui.c 中，UI 任务从信箱取消息并更新界面：

    void Task_LvglUI(void *arg)
    {
        ui_msg_t msg;
        while(1)
        {
            // 从信箱里取信，取不到就不等（参数 0 = 不等待）
            if(xQueueReceive(ui_msg_queue, &msg, 0) == pdPASS)
            {
                switch(msg.type)                  // 看信封上的类型
                {
                    case UI_MSG_KEY_EVENT:        // 是按键事件
                        lv_label_set_text(label_key, key_name[msg.data.key_id]);
                        break;                    // 更新界面上的按键显示
                    case UI_MSG_RTC_TIME:         // 是时间事件
                        update_clock_display(msg.data.rtc_time);
                        break;                    // 更新时钟显示
                    default:
                        break;
                }
            }
            lv_task_handler();                    // LVGL 刷新界面
            vTaskDelay(pdMS_TO_TICKS(5));         // 休息 5ms
        }
    }

注意点：
- xQueueSend 第三个参数是等待时间，0 = 非阻塞（信箱满了直接丢掉）
- xQueueReceive 第三个参数是等待时间，portMAX_DELAY = 一直等到有信为止
- 消息结构体不要太大，建议不超过 64 字节，太大占内存
- 一个队列可以被多个任务发送，但建议只有一个任务接收，避免混乱
```

**（3）app_arch 层：线程安全的 UI 更新接口（必须输出）** ⭐ 新增

必须用大白话讲清楚 app_arch 层的作用和使用方法，包含：
- app_arch 层是什么（用比喻解释）
- 为什么需要 app_arch 层（解耦、线程安全）
- 外部任务怎么调用 app_arch 接口更新 UI
- 代码示例（基于实际生成的 task_system_periph.c）

输出格式参考（必须按此格式写出实际内容）：
```
app_arch 层怎么用？（大白话版）：

打个比方：app_arch 层就是一个"翻译官"或者"前台接待"。
- 传感器任务想更新 UI，但不想直接操作消息队列（太麻烦、容易出错）
- 传感器任务只需调用 app_arch_ui_update() 函数，把数据交给"翻译官"
- "翻译官"会自动把数据打包成消息，塞进消息队列
- UI 任务从消息队列取出来，更新界面

【为什么需要 app_arch 层？】

1. 解耦：传感器任务不需要知道消息队列的存在，只需要调用简单函数
2. 线程安全：app_arch 内部使用 FreeRTOS 消息队列，自动处理并发
3. 统一接口：所有外部任务都通过同一个接口更新 UI，方便维护
4. 可扩展：新增 UI 更新类型只需在枚举和 switch-case 中添加

【发送端示例】—— 在 task_system_periph.c 中，传感器读取后通过 app_arch 层更新 UI：

    #include "app_arch_ui.h"  // 引入 app_arch 层接口
    
    void Task_SystemPeriph(void *arg)
    {
        while(1)
        {
            // 读取传感器数据
            float temp = dht11_read_temp();
            
            // 通过 app_arch 层更新 UI（线程安全，一行搞定）
            app_arch_ui_update(UI_UPDATE_SENSOR, &temp);
            
            // 或者更新电池电压
            float voltage = adc_read_battery();
            app_arch_ui_update(UI_UPDATE_BATTERY, &voltage);
            
            // 或者更新 RTC 时间
            rtc_time_t time;
            ds1302_read_time(&time);
            app_arch_ui_update(UI_UPDATE_RTC, &time);
            
            vTaskDelay(pdMS_TO_TICKS(1000));
        }
    }

【接收端示例】—— 在 task_ui.c 中，UI 任务从消息队列取消息并更新界面：

    void Task_LvglUI(void *arg)
    {
        ui_update_msg_t msg;
        while(1)
        {
            // 从消息队列取消息
            if(xQueueReceive(ui_msg_queue, &msg, 0) == pdPASS)
            {
                switch(msg.type)
                {
                    case UI_UPDATE_SENSOR:
                        // 更新传感器显示
                        update_sensor_display(msg.data.sensor);
                        break;
                    case UI_UPDATE_BATTERY:
                        // 更新电池电压显示
                        update_battery_display(msg.data.voltage);
                        break;
                    case UI_UPDATE_RTC:
                        // 更新时钟显示
                        update_clock_display(msg.data.rtc_time);
                        break;
                    default:
                        break;
                }
            }
            lv_task_handler();
            vTaskDelay(pdMS_TO_TICKS(5));
        }
    }

【app_arch 层内部实现】—— 在 app_arch/app_arch_ui.c 中：

    // 内部封装了消息队列操作
    esp_err_t app_arch_ui_update(ui_update_type_t type, void *data)
    {
        ui_update_msg_t msg;
        msg.type = type;
        
        // 根据类型拷贝数据
        switch(type) {
            case UI_UPDATE_SENSOR:
                memcpy(&msg.data.sensor, data, sizeof(sensor_data_t));
                break;
            case UI_UPDATE_BATTERY:
                msg.data.voltage = *(float *)data;
                break;
            // ... 其他类型
        }
        
        // 非阻塞发送到 UI 消息队列
        return xQueueSend(ui_msg_queue, &msg, 0) == pdPASS ? ESP_OK : ESP_ERR_NO_MEM;
    }

注意点：
- 外部任务只需 #include "app_arch_ui.h" 并调用 app_arch_ui_update()
- 不需要知道消息队列的细节，app_arch 层自动处理
- 线程安全：多个任务可以同时调用 app_arch_ui_update()，不会冲突
- 队列满了会丢弃消息（非阻塞），避免阻塞传感器任务
- 新增 UI 更新类型：在 ui_update_type_t 枚举中添加，在 switch-case 中处理
```

##### 4.1.5 组件与任务内部函数关联说明（强制输出要求）

> **⚠️ 强制要求：生成 ARCH_GUIDE.md 时，必须针对项目中用到的每一个组件、每一个驱动、每一个任务，按以下模板格式逐一输出说明。不能省略、不能合并、不能只写"请参考源码"。必须基于实际生成的代码写出。**

**每个组件/驱动必须按以下模板输出（必须实际写出，不能省略）：**

```
【组件名称】：XXX（例如：DS1302 RTC 时钟芯片）

【干什么用的】：
    用大白话解释这个组件的功能。

【在哪个文件里】：
    - 驱动文件：components/xxx/xxx.c
    - 头文件：  components/xxx/xxx.h
    - 在哪个任务里调用：task_xxx.c（xxx 任务）

【内部函数说明】：
    - xxx_init()        → 初始化，干什么用
    - xxx_read()        → 读取数据，返回什么
    - xxx_write()       → 写入数据，什么时候用

【怎么调用（示例）】：
    // 在实际调用的任务文件中，写出调用示例代码
    #include "xxx.h"
    
    void Task_Xxx(void *arg)
    {
        xxx_init();                          // 初始化
        while(1)
        {
            // 调用示例
            xxx_read(&data);
            // 通过消息队列发送
            xQueueSend(queue, &msg, 0);
            vTaskDelay(pdMS_TO_TICKS(1000));
        }
    }

【注意点】：
    - 引脚配置在哪里
    - 只能在哪个任务里调用
    - 其他任务怎么获取数据（通过消息队列，不要直接调用）
```

**每个任务必须按以下模板输出（必须实际写出，不能省略）：**

```
【任务名称】：Task_Xxx（xxx 任务）

【干什么用的】：
    用大白话解释这个任务的职责。

【在哪个文件里】：tasks/task_xxx.c

【内部函数列表】：
    - Task_Xxx()         → 任务主函数（入口），while(1) 循环
    - func_a()           → 干什么用（在 drivers/xxx.c 中）
    - func_b()           → 干什么用（在 components/xxx/ 中）

【调用关系（大白话）】：
    Task_Xxx
        ├── 调用 func_a()     → 做什么 → 结果怎么处理（发消息/直接处理）
        ├── 调用 func_b()     → 做什么 → 结果怎么处理
        └── 调用 func_c()     → 做什么 → 结果怎么处理

【注意点】：
    - 这个任务里不能做什么（比如阻塞操作）
    - 必须做什么（比如定期喂狗）
    - 如果某个组件挂了会怎样，怎么处理
```

##### 4.1.6 UI 框架使用指南（强制输出要求）

> **⚠️ 强制要求：如果项目中使用了 UI 框架（如 LVGL、TFT_eSPI 等），生成的 ARCH_GUIDE.md 必须包含完整的"UI 框架使用指南（大白话版）"章节，以下所有核心章节都不能省略。必须基于实际生成的代码写出。**

**必须包含的核心章节（缺一不可）：**

**（1）页面注册 4 步流程（必须输出）**

用大白话讲清楚怎么注册一个新页面，包含完整的 4 个步骤和代码示例：
```
页面注册 4 步流程（大白话版）：

第 1 步：创建页面文件
    - 在 ui/pages/ 目录下新建 page_xxx.c 和 page_xxx.h
    - 复制一个现有页面作为模板，改个名字

第 2 步：写页面初始化函数
    - 在 page_xxx.c 里写 page_xxx_init() 函数
    - 在这个函数里创建页面的 UI 控件（按钮、标签等）
    - 示例：
      void page_xxx_init(void) {
          // 创建标题
          lv_obj_t *label = lv_label_create(lv_scr_act());
          lv_label_set_text(label, "页面标题");
          
          // 创建按钮
          lv_obj_t *btn = lv_btn_create(lv_scr_act());
          // ... 设置按钮位置、大小等
      }

第 3 步：在页面管理器中注册
    - 打开 ui/page_manager.c
    - 在 page_manager_init() 函数里添加你的页面
    - 示例：
      void page_manager_init(void) {
          pages[PAGE_HOME] = page_home_init;
          pages[PAGE_SETTINGS] = page_settings_init;
          pages[PAGE_XXX] = page_xxx_init;  // ← 加这一行
      }

第 4 步：在枚举中定义页面 ID
    - 打开 ui/page_manager.h
    - 在 page_id_t 枚举里添加你的页面 ID
    - 示例：
      typedef enum {
          PAGE_HOME = 0,
          PAGE_SETTINGS,
          PAGE_XXX,  // ← 加这一行
          PAGE_MAX
      } page_id_t;
```

**（2）起始页设定（必须输出）**

用大白话讲清楚怎么设置开机显示的第一个页面：
```
起始页设定（大白话版）：

在 ui/page_manager.c 的 page_manager_init() 函数最后，有一行：
    current_page = PAGE_HOME;  // ← 改这里

把 PAGE_HOME 改成你想显示的页面 ID 就行了。

比如想让开机显示设置页面：
    current_page = PAGE_SETTINGS;
```

**（3）多页面注册（必须输出）**

用大白话讲清楚怎么注册多个页面，以及页面之间怎么切换：
```
多页面注册（大白话版）：

假设你要注册 3 个页面：主页、设置页、关于页

1. 先创建 3 个页面文件：
   - ui/pages/page_home.c
   - ui/pages/page_settings.c
   - ui/pages/page_about.c

2. 在每个文件里写 init 函数：
   - page_home_init()
   - page_settings_init()
   - page_about_init()

3. 在 ui/page_manager.h 的枚举里定义 ID：
   typedef enum {
       PAGE_HOME = 0,
       PAGE_SETTINGS,
       PAGE_ABOUT,
       PAGE_MAX
   } page_id_t;

4. 在 ui/page_manager.c 的 page_manager_init() 里注册：
   pages[PAGE_HOME] = page_home_init;
   pages[PAGE_SETTINGS] = page_settings_init;
   pages[PAGE_ABOUT] = page_about_init;

5. 页面切换（在任意地方调用）：
   page_manager_switch_to(PAGE_SETTINGS);  // 切换到设置页
```

**（4）按键/触摸接入（必须输出）**

用大白话讲清楚怎么把按键或触摸事件接入到 UI 系统：
```
按键/触摸接入（大白话版）：

【按键接入】：

1. 在 drivers/key_scan.c 里扫描按键
2. 检测到按键后，通过消息队列发给 UI 任务：
   ui_msg_t msg;
   msg.type = UI_MSG_KEY_EVENT;
   msg.data.key_id = key;
   xQueueSend(ui_msg_queue, &msg, 0);

3. 在 task_ui.c 里接收按键消息：
   if(xQueueReceive(ui_msg_queue, &msg, 0) == pdPASS) {
       if(msg.type == UI_MSG_KEY_EVENT) {
           // 处理按键事件
           switch(msg.data.key_id) {
               case KEY_UP:
                   // 上键按下
                   break;
               case KEY_DOWN:
                   // 下键按下
                   break;
           }
       }
   }

【触摸接入】：

1. 在 drivers/touch.c 里初始化触摸驱动
2. 在 task_ui.c 里定期读取触摸坐标：
   touch_point_t point;
   if(touch_read(&point)) {
       // 有触摸，处理坐标
       if(point.y < 50) {
           // 点击了顶部区域
       }
   }
```

**（5）数据传递（必须输出）**

用大白话讲清楚怎么在页面之间传递数据：
```
数据传递（大白话版）：

【方法 1：通过全局变量】

适合简单数据（比如当前温度、时间）：

1. 在 ui/ui_data.h 里定义全局变量：
   extern float current_temperature;
   extern rtc_time_t current_time;

2. 在 ui/ui_data.c 里定义实际变量：
   float current_temperature = 0.0f;
   rtc_time_t current_time = {0};

3. 在 task_system_periph.c 里更新数据：
   #include "ui_data.h"
   current_temperature = dht11_read_temp();

4. 在 page_xxx.c 里读取数据：
   #include "ui_data.h"
   char buf[32];
   sprintf(buf, "%.1f°C", current_temperature);
   lv_label_set_text(label_temp, buf);

【方法 2：通过消息队列】

适合复杂数据或事件（比如按键事件、传感器数据）：

1. 在 middleware/msg_queue.h 里定义消息结构：
   typedef struct {
       ui_msg_type_t type;
       union {
           uint8_t key_id;
           float temperature;
       } data;
   } ui_msg_t;

2. 发送方：
   ui_msg_t msg;
   msg.type = UI_MSG_TEMP_UPDATE;
   msg.data.temperature = 25.5f;
   xQueueSend(ui_msg_queue, &msg, 0);

3. 接收方（在 task_ui.c）：
   if(xQueueReceive(ui_msg_queue, &msg, 0) == pdPASS) {
       if(msg.type == UI_MSG_TEMP_UPDATE) {
           // 更新温度显示
       }
   }
```

**（6）刷新机制（必须输出）**

用大白话讲清楚 UI 怎么刷新：
```
刷新机制（大白话版）：

LVGL 的刷新机制（大白话）：

1. lv_task_handler() 是 LVGL 的核心刷新函数
2. 必须在 UI 任务里定期调用（建议 5-10ms 一次）：
   void Task_LvglUI(void *arg) {
       while(1) {
           // 处理消息队列
           // ...
           
           lv_task_handler();  // ← 刷新 UI
           vTaskDelay(pdMS_TO_TICKS(5));  // 5ms 刷新一次
       }
   }

3. lv_task_handler() 会自动：
   - 检查哪些控件需要重绘
   - 只重绘变化的部分（省电）
   - 处理动画、触摸事件等

4. 你不需要手动调用 lv_refr_now()，lv_task_handler() 会搞定

【注意】：
- 不要在 lv_task_handler() 之外直接操作 LVGL 控件
- 所有 UI 更新都应该通过消息队列发给 UI 任务
- 如果其他任务想更新 UI，发消息，不要直接调用 lv_xxx()
```

**（7）关键参数说明（必须输出）**

用大白话讲清楚 UI 框架的关键配置参数：
```
关键参数说明（大白话版）：

【LVGL 配置参数】（在 lv_conf.h 里）：

1. LV_HOR_RES_MAX / LV_VER_RES_MAX
   - 屏幕分辨率（宽 x 高）
   - 示例：320x240、240x320
   - 必须和实际屏幕一致，否则显示错位

2. LV_COLOR_DEPTH
   - 颜色深度
   - 16 = 16位色（65536色，推荐）
   - 32 = 32位色（更鲜艳，但占内存）

3. LV_DISP_BUF_SIZE
   - 显示缓冲区大小
   - 建议设为屏幕宽度的 1/10
   - 示例：320x240 屏幕，设为 320*24 = 7680

4. LV_TICK_CUSTOM
   - 是否使用自定义时钟
   - 设为 1，然后在代码里调用 lv_tick_inc(x)
   - 或者设为 0，让 LVGL 自己处理

【任务参数】（在 app_config.h 里）：

1. TASK_UI_STACK_SIZE
   - UI 任务栈大小
   - 建议 8192-12288（8-12KB）
   - LVGL 比较吃栈，别设太小

2. TASK_UI_PRIORITY
   - UI 任务优先级
   - 建议 5（最高优先级）
   - UI 必须流畅，不能卡
```

**（8）扩展接口（必须输出）**

用大白话讲清楚怎么扩展 UI 功能：
```
扩展接口（大白话版）：

【添加新控件】：

1. 在 page_xxx.c 里创建控件：
   // 创建滑块
   lv_obj_t *slider = lv_slider_create(lv_scr_act());
   lv_obj_set_pos(slider, 50, 100);
   lv_obj_set_size(slider, 200, 30);
   lv_slider_set_range(slider, 0, 100);
   lv_slider_set_value(slider, 50, LV_ANIM_ON);

2. 绑定事件回调：
   lv_obj_add_event_cb(slider, slider_event_cb, LV_EVENT_VALUE_CHANGED, NULL);

3. 写回调函数：
   static void slider_event_cb(lv_event_t *e) {
       lv_obj_t *slider = lv_event_get_target(e);
       int value = lv_slider_get_value(slider);
       // 处理滑块值变化
   }

【添加自定义字体】：

1. 用 LVGL 在线字体转换器生成 .c 文件：
   https://lvgl.io/tools/fontconverter

2. 把生成的 .c 文件放到 ui/fonts/ 目录

3. 在 page_xxx.c 里引用：
   LV_FONT_DECLARE(my_font);
   lv_obj_set_style_text_font(label, &my_font, 0);

【添加图片资源】：

1. 用 LVGL 在线图片转换器生成 .c 文件：
   https://lvgl.io/tools/imageconverter

2. 把生成的 .c 文件放到 ui/images/ 目录

3. 在 page_xxx.c 里显示：
   LV_IMG_DECLARE(my_image);
   lv_obj_t *img = lv_img_create(lv_scr_act());
   lv_img_set_src(img, &my_image);
```

**（9）流程图（必须输出）**

用大白话画出 UI 系统的整体流程图：
```
UI 系统流程图（大白话版）：

┌─────────────────────────────────────────────────────────┐
│                    系统启动                               │
│                      ↓                                   │
│              app_main() 调用                             │
│                      ↓                                   │
│           app_arch_init() 创建                           │
│          ┌──────────┴──────────┐                        │
│          ↓                      ↓                        │
│    创建消息队列            创建 UI 任务                   │
│    (ui_msg_queue)          (Task_LvglUI)                │
│          ↓                      ↓                        │
│    创建其他任务            UI 任务初始化                  │
│    (外设、打印等)          (page_manager_init)           │
│                                 ↓                        │
│                          加载起始页面                    │
│                          (PAGE_HOME)                     │
│                                 ↓                        │
│                    ┌────────────┴────────────┐          │
│                    ↓                         ↓          │
│             等待消息队列               lv_task_handler() │
│             (xQueueReceive)             (刷新 UI)       │
│                    ↓                         ↑          │
│             收到消息？                    循环执行       │
│             ├─ 是 → 处理消息 ────────────┘              │
│             └─ 否 → 继续等待                            │
└─────────────────────────────────────────────────────────┘

【消息流向】：

按键任务 ──→ ui_msg_queue ──→ UI 任务 ──→ lv_label_set_text()
传感器任务 ──→ ui_msg_queue ──→ UI 任务 ──→ lv_label_set_text()
打印任务 ←── print_job_queue ←── 任意任务
```

**（10）速查表（必须输出）**

用大白话提供常用 API 速查表：
```
UI 常用 API 速查表（大白话版）：

【页面管理】：
page_manager_init()              → 初始化页面管理器
page_manager_switch_to(PAGE_X)   → 切换到指定页面
page_manager_get_current()       → 获取当前页面 ID

【标签（文字）】：
lv_label_create(parent)          → 创建标签
lv_label_set_text(label, "文本")  → 设置文字
lv_label_set_text_fmt(label, "%d", value) → 格式化设置文字

【按钮】：
lv_btn_create(parent)            → 创建按钮
lv_obj_add_event_cb(btn, cb, LV_EVENT_CLICKED, NULL) → 绑定点击事件
lv_obj_set_size(btn, w, h)       → 设置大小
lv_obj_set_pos(btn, x, y)        → 设置位置

【输入框】：
lv_textarea_create(parent)       → 创建输入框
lv_textarea_set_text(ta, "文本")  → 设置文字
lv_textarea_get_text(ta)         → 获取文字

【滑块】：
lv_slider_create(parent)         → 创建滑块
lv_slider_set_range(slider, min, max) → 设置范围
lv_slider_set_value(slider, val, LV_ANIM_ON) → 设置值
lv_slider_get_value(slider)      → 获取值

【图片】：
lv_img_create(parent)            → 创建图片
lv_img_set_src(img, &img_dsc)    → 设置图片源

【通用控件操作】：
lv_obj_set_pos(obj, x, y)        → 设置位置
lv_obj_set_size(obj, w, h)       → 设置大小
lv_obj_add_flag(obj, LV_OBJ_FLAG_HIDDEN) → 隐藏
lv_obj_clear_flag(obj, LV_OBJ_FLAG_HIDDEN) → 显示
```

##### 4.1.7 修改指南
- 如何添加新传感器：在 drivers/ 下新建文件 → 在 task_system_periph.c 中 #include 并调用 → 在 `app_arch_ui.h` 的 `ui_update_type_t` 枚举中添加新类型 → 在 `app_arch_ui.c` 的 switch-case 中添加数据拷贝逻辑 → 在 UI 任务的 switch-case 中添加显示处理
- 如何添加新任务：⭐⭐ **必须创建独立组件** → 在 `components/` 下新建 `task_xxx/` 目录 → 创建 `task_xxx.c`、`task_xxx.h`、`CMakeLists.txt` → 在 `CMakeLists.txt` 中配置 ESP32 依赖（必须包含 `freertos`） → 在 `main/CMakeLists.txt` 的 REQUIRES 中添加 `task_xxx` → 在 `app_arch_init()` 中 xTaskCreate
- 如何调整任务优先级：修改 app_arch_init() 中 xTaskCreate 的第 5 个参数
- 如何新增 UI 更新类型：在 `app_arch/app_arch_ui.h` 的 `ui_update_type_t` 枚举中添加新类型 → 在 `app_arch_ui.c` 的 `app_arch_ui_update()` 函数中添加对应的数据拷贝逻辑 → 在 UI 任务的 `xQueueReceive` 后的 switch-case 中添加对应的 UI 更新处理
- 如何使用 app_arch 层更新 UI：在外部任务中 `#include "app_arch_ui.h"` → 调用 `app_arch_ui_update(UI_UPDATE_XXX, &data)` → 无需直接操作消息队列

---

## 触发条件

当用户提出以下需求时，自动触发本 Skill：
- "帮我创建一个 FreeRTOS 项目架构"
- "我需要规划 ESP32 的多任务"
- "如何集成 DS1302 和 LVGL"
- "帮我设计一个打印机项目的 FreeRTOS 架构"
- "生成一个包含 WiFi 和传感器的 FreeRTOS 框架"

## 注意事项

1. **必须先询问用户需求**：传感器、第三方库、UI 框架等信息
2. **不要假设**：如果用户未说明 UI 框架，必须询问
3. **保持简单**：不要过度设计，按需添加中间件
4. **CMake 必须**：ESP-IDF 项目必须生成正确的 CMakeLists.txt
5. **中文文档**：架构使用指南必须用大白话，方便理解
