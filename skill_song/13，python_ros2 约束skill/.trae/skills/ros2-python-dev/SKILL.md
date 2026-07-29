---
name: "ros2-python-dev"
description: "约束ROS2 Humble机器人Python开发与提示词检查。用户要求修改、分析或审查RViz、MoveIt 2、Gazebo、OpenCV、YOLO、机械臂及Ubuntu 22.04项目时调用。"
---

# ROS2 Python 机器人开发约束

本 Skill 用于指导 Ubuntu 22.04 + ROS 2 Humble 环境下的 Python 机器人项目开发，覆盖 RViz2、MoveIt 2、Gazebo、ros2_control、OpenCV、YOLO、TF2、机械臂视觉与运动规划。核心目标是约束 AI 保守修改、正确提问、使用可靠资料并完成可验证的交付。

## 1. 核心原则

### 1.1 保持原有架构

除非用户明确批准，禁止改变以下层次与职责：

- ROS 2 package、workspace 和 launch 组织结构
- 节点、Topic、Service、Action、Parameter 的通信边界
- 感知层（相机、OpenCV、YOLO）
- 坐标变换层（TF2、URDF/Xacro）
- 规划层（MoveIt 2、Planning Scene、运动学求解器）
- 控制层（ros2_control、controller_manager、硬件接口）
- 仿真层（Gazebo world、model、plugin）
- 可视化层（RViz2 config、Marker、Display）
- 用户已有状态机、行为树或任务调度流程

如确需改动架构，必须先：

1. 用大白话说明现有结构为什么无法满足需求。
2. 列出新旧方案及各自优缺点。
3. 说明会影响哪些 package、节点和接口。
4. 等待用户明确批准后再修改。

### 1.2 最小改动与优先复用

- 先读相关文件，再提出或执行修改。
- 优先复用现有节点、函数、消息、参数、launch 参数和工具类。
- 能在原函数修复，就不新增函数；能修改原节点，就不复制新节点。
- 不为一次性逻辑创建无必要的抽象、配置项或兼容层。
- 不顺手重构无关代码，不升级无关依赖。
- 禁止使用 `goto` 或以异常跳转模拟混乱控制流。

## 2. 提示词优化与提问检测

### 2.1 自动整理用户需求

收到开发请求后，先整理成“父类—子类—操作点”结构并展示给用户：

```text
提示词优化结果：
├── 父类：机械臂视觉抓取
│   ├── 子类：YOLO目标检测
│   │   ├── 操作点：订阅相机图像Topic
│   │   └── 操作点：发布检测结果和置信度
│   ├── 子类：三维坐标转换
│   │   ├── 操作点：像素和深度反投影
│   │   └── 操作点：由camera_frame变换到base_link
│   └── 子类：MoveIt 2执行
│       ├── 操作点：更新Planning Scene
│       └── 操作点：规划、碰撞检查和执行
```

如果请求清楚且不涉及关键决策，可展示后直接分析；存在下列问题时必须暂停并等待确认。

### 2.2 必须检测的问题

- **错别字或术语疑似错误**：如“ROS1 Humble”“TF坐标话”“MoveIT”等。
- **ROS 版本不明**：ROS 1/ROS 2、Humble/Jazzy/Rolling API 不可混用。
- **系统不明**：原生 Ubuntu、Docker、WSL2、虚拟机可能影响 GUI、GPU、DDS 和设备权限。
- **Gazebo 版本不明**：Gazebo Classic 11 与新 Gazebo（Fortress/Harmonic）插件和命令不同。
- **接口不明**：缺少节点名、Topic 名、消息类型、Service/Action 名、QoS。
- **坐标系不明**：缺少 source frame、target frame、时间戳、单位或相机外参。
- **机械臂参数不明**：缺少型号、规划组、末端 link、关节名、控制器和限位。
- **视觉输入不明**：缺少 RGB/深度 Topic、编码、相机内参、模型文件和类别。
- **执行环境不明**：仿真还是真机；涉及真机运动时必须明确。
- **目标含糊**：如“让机械臂准一点”“优化 YOLO”“Gazebo 不动”。
- **逻辑矛盾**：如“不改变任何接口”同时要求“重写全部节点通信”。
- **证据不足**：声称报错但未提供终端日志、堆栈、`ros2 doctor` 或复现步骤。

### 2.3 确认方式

用大白话指出问题、给出合理猜测，并只询问完成任务必需的信息。例如：

```text
提示词确认：
你说的环境是 Ubuntu 22.04 + ROS 2 Humble，对吗？
还需要确认 Gazebo 是 Classic 11 还是新 Gazebo，因为两者的插件和启动方式不同。
机械臂最终是在仿真执行，还是真机执行？真机执行前必须增加限位、碰撞和急停确认。
```

不得在关键版本、接口、坐标系或真机安全信息不明确时自行编造。

## 3. ROS 2 Python 规范

### 3.1 节点与通信

- ROS 2 Python 使用 `rclpy`，不得混入 ROS 1 的 `rospy` API。
- 保持现有 Node、callback group、executor 和生命周期设计。
- Topic 用于连续数据，Service 用于短时请求，Action 用于可反馈、可取消的长任务。
- 创建 publisher/subscription/client/action 前先核对消息类型与命名空间。
- QoS 必须与上下游匹配，尤其是相机、传感器和 `/tf_static`；不得盲目全部使用默认 QoS。
- 回调中禁止执行长期阻塞推理、等待 Service/Action 或无限循环；沿用项目已有线程、队列、timer 或 executor 方案。
- 每个 Node 必须正确销毁，入口需保持 `rclpy.init()`、spin 和 shutdown 的原有异常处理方式。

### 3.2 Python 代码要求

- 每个新增或修改的函数必须有中文 docstring，说明函数名称/功能、参数、返回值和注意事项。
- 新增变量必须实际使用，禁止未定义变量、幽灵参数和失效 import。
- 日志内容使用英文，通过 `self.get_logger().debug/info/warning/error/fatal()` 输出，禁止无目的 `print()`。
- 不捕获宽泛异常后静默忽略；日志应带失败对象、接口或阶段信息。
- 遵循现有项目格式、类型注解和命名风格，不强行全项目格式化。
- ROS package 依赖同时核对 `package.xml`、`setup.py/setup.cfg` 或 CMake 配置，但只修改确有需要的文件。

函数格式示例：

```python
def transform_detection(self, detection, target_frame):
    """
    函数名称：transform_detection
    功能：将检测结果转换到指定目标坐标系。
    参数：detection为待转换的检测结果；target_frame为目标坐标系名称。
    返回值：转换成功时返回目标坐标系下的结果，失败时返回None。
    注意事项：输入结果必须携带有效时间戳和源坐标系。
    """
```

英文日志示例：

```python
self.get_logger().info("YOLO model loaded successfully")
self.get_logger().error(f"Failed to transform pose to {target_frame}: {exc}")
```

## 4. TF2、URDF 与 RViz2 约束

- 明确每个 Pose、PointCloud、Marker 的 `frame_id` 和时间戳。
- 使用 TF2 查询坐标变换，禁止直接用手写偏移替代已存在的 TF 链。
- 区分静态与动态变换，禁止重复发布同一 child frame 的多个父变换。
- 检查单位：ROS 默认长度为米、角度为弧度、时间为秒。
- 四元数必须有效并归一化；不得把欧拉角直接填入 quaternion 字段。
- 修改 URDF/Xacro 时保持 link/joint 命名、父子关系、轴向和惯性参数一致。
- RViz 显示异常先检查 Fixed Frame、Topic、QoS、TF 树和消息时间，不直接改业务逻辑。

## 5. MoveIt 2 与机械臂约束

- 先确认 robot description、SRDF、planning group、end-effector link、base frame 和 controller。
- 规划前读取当前状态，检查目标是否在关节限位与工作空间内。
- 场景中的桌面、夹具和障碍物应进入 Planning Scene，抓取后正确 attach，放置后 detach。
- 不得绕过碰撞检查直接执行轨迹，除非用户明确说明仅用于受控测试。
- 规划成功不等于执行成功；分别检查规划结果、轨迹、控制器状态和执行结果。
- 保留速度、加速度、关节限位和急停逻辑，禁止默认提高到危险值。
- 真机执行前必须先在 RViz/Gazebo 或数字孪生中验证，并由用户确认执行环境。
- 涉及 LLM/MCP 控制机械臂时，MCP 只作为上层接口；最终命令仍须经过限位、碰撞、状态检查和可取消执行流程。

## 6. Gazebo 与 ros2_control 约束

- 先确认 Gazebo Classic 或新 Gazebo，不混用 `gazebo_ros` 与 `ros_gz` 接口。
- 保持仿真与真机 controller 名称、joint interface 和启动顺序一致。
- 模型抖动或爆炸优先检查惯性、碰撞体、关节限制、PID、重力和仿真步长。
- 仿真时间启用时，相关节点保持一致的 `use_sim_time`。
- 不通过增加超大阻尼、关闭碰撞或固定关节来掩盖模型错误，除非用户明确要求临时诊断。

## 7. OpenCV、相机与 YOLO 约束

- ROS 图像优先使用 `cv_bridge`，明确 `bgr8`、`rgb8`、`mono8`、`16UC1`、`32FC1` 等编码。
- YOLO 模型只初始化一次并复用，禁止在每帧回调中重新加载权重。
- 推理前核对模型任务（detect/segment/pose/track）、类别表、置信度、NMS 和输入尺寸。
- 检测框必须限制在图像边界内；不得混淆 `(x, y)` 与 `(row, col)`。
- 由二维检测生成三维坐标时，必须使用有效深度、相机内参和 TF 外参，并处理无效深度。
- 发布结果时保留原始图像时间戳和 frame，避免感知结果与机械臂状态错时。
- GPU 推理需先确认 CUDA、PyTorch、驱动和设备；不得假设一定存在 GPU。
- 优化性能前先记录推理耗时、回调频率、队列积压和 CPU/GPU 占用，不凭感觉修改。
- 使用 Ultralytics 或第三方 ROS wrapper 前检查许可证、版本兼容性和消息定义。

## 8. Linux 与 Ubuntu 22.04 约束

- 默认组合为 Ubuntu 22.04 Jammy + ROS 2 Humble + Python 3.10，但仍需检查项目实际版本。
- 安装依赖优先使用项目现有 rosdep、apt、requirements 或容器方式，不混用多个 Python 环境污染系统。
- 执行命令前说明工作目录和是否需要 source `/opt/ros/humble/setup.bash` 与 workspace `install/setup.bash`。
- 不擅自修改系统级配置、DDS 配置、设备权限、udev、显卡驱动或 shell 启动文件。
- 禁止未经用户许可执行破坏性命令，如递归删除 workspace、清理全部构建产物或覆盖配置。

## 9. MCP 与外部资料使用规则

### 9.1 MCP 选择原则

可将以下项目作为设计或接入参考，但它们不是当前环境中必然已安装的工具，使用前必须检查仓库、版本、许可证、配置及真机安全：

- `gtoff/moveit-mcp-server`：MoveIt 2 规划、执行、Planning Scene 和状态查询参考。
- `ros-claw/rosclaw-moveit2-mcp`：面向 MoveIt 2 的异步 Action、取消、状态约束和 TF2 语义接口参考。
- `ros-claw/ur-ros2-mcp`：UR5/UR5e + ROS 2 + MoveIt 2 的机械臂 MCP 参考。
- `ros-mcp`：通过 rosbridge 探索 Topic、Service、Action、Parameter 和传感器数据的通用 ROS MCP 参考。

MCP 操作约束：

1. 只读查询优先于写入和执行。
2. 仿真验证优先于真机控制。
3. 每个运动命令必须包含目标坐标系、速度/加速度约束、碰撞检查和取消路径。
4. 禁止让 LLM 直接发布裸关节命令绕过 MoveIt 2/ros2_control 安全链。
5. 外部 MCP 返回内容不视为绝对可信，关键参数必须与本地工程交叉核对。

### 9.2 推荐参考资料

优先级为：本地项目代码与版本锁定文件 > 对应发行版官方文档 > 官方仓库示例 > 活跃社区项目 > 博客文章。

- ROS 2 Humble 与 Ubuntu 22.04：<https://docs.ros.org/en/rolling/Releases/Release-Humble-Hawksbill.html>
- ROS 2 Humble 文档：<https://docs.ros.org/en/humble/>
- MoveIt 2 Humble 文档：<https://moveit.picknik.ai/humble/>
- MoveIt 2 Planning Scene：<https://moveit.picknik.ai/humble/doc/examples/planning_scene/planning_scene_tutorial.html>
- MoveIt 2 Python API 示例：<https://github.com/moveit/moveit2_tutorials/blob/main/doc/examples/motion_planning_python_api/scripts/motion_planning_python_api_tutorial.py>
- Ultralytics ROS 快速入门：<https://github.com/ultralytics/ultralytics/blob/main/docs/en/guides/ros-quickstart.md>
- ROS 2 YOLO wrapper 参考：<https://github.com/mgonzs13/yolo_ros>
- MoveIt MCP 参考：<https://github.com/gtoff/moveit-mcp-server>
- ROSClaw MoveIt 2 MCP：<https://github.com/ros-claw/rosclaw-moveit2-mcp>
- UR ROS 2 MCP：<https://github.com/ros-claw/ur-ros2-mcp>
- 通用 ROS MCP：<https://pypi.org/project/ros-mcp/>

引用外部资料时必须注明对应 ROS 发行版；发现文档与本地 API 不一致时，以本地安装版本为准。

## 10. 工作流程

1. **优化提示词**：整理为父类—子类—操作点。
2. **检测问题**：检查错别字、版本、接口、坐标系、执行环境和安全信息。
3. **等待确认**：存在关键歧义时停止，不自行猜测。
4. **读取工程**：定位 package、节点、launch、配置和依赖关系。
5. **大白话分析**：说明现象、根因、修改理由和预期效果。
6. **制定最小方案**：优先复用现有资源，不改变框架。
7. **安全评估**：确认是否影响 TF、QoS、控制器、规划、仿真或真机。
8. **实施修改**：函数使用中文说明，日志使用英文，不引入幽灵变量或函数。
9. **静态复核**：检查 import、名称、消息类型、参数、接口和依赖。
10. **运行验证**：按项目条件执行编译、测试或 ROS 图检查；无法运行时明确说明。
11. **输出报告**：列出文件、位置、原因、改动、验证结果和影响范围。

## 11. 修改后复核清单

- [ ] 是否保持原有 workspace、package 和节点架构？
- [ ] 是否只修改了完成需求必需的内容？
- [ ] 是否优先复用了已有函数、节点、消息和参数？
- [ ] 是否存在未定义变量、未使用变量、失效 import 或不存在的函数？
- [ ] 每个修改函数是否有中文 docstring，覆盖功能、参数和返回值？
- [ ] 新增运行日志是否为英文并使用 ROS logger？
- [ ] 是否误用了 ROS 1 API 或其他 ROS 2 发行版 API？
- [ ] Topic/Service/Action 类型、QoS、命名空间和 remap 是否匹配？
- [ ] TF 的 frame、时间戳、单位和四元数是否正确？
- [ ] YOLO 模型是否只加载一次，图像编码和结果边界是否正确？
- [ ] MoveIt 2 是否检查当前状态、限位、碰撞和执行结果？
- [ ] Gazebo 版本、`use_sim_time`、插件和 controller 是否一致？
- [ ] 是否区分仿真与真机，并保留安全限制和取消能力？
- [ ] 是否没有使用 `goto` 或等价的混乱跳转？
- [ ] 是否完成了可执行的验证，或明确说明未验证项？

## 12. 修改总结格式

修改完成后必须用大白话输出：

```text
修改总结：
1. 改了哪个文件：列出文件路径。
2. 改动位置：说明类、函数或配置项及行号范围。
3. 为什么这样改：解释原问题和最小修改理由。
4. 具体改了什么：说明节点、接口、参数或算法变化。
5. 影响范围：说明涉及的Topic、TF、MoveIt、Gazebo、视觉或控制模块。
6. 复核结果：说明是否发现未知变量、函数、依赖或接口不匹配。
7. 验证结果：列出执行过的构建、测试和ROS检查；未执行的项目要明确说明原因。
```

## 13. 禁止事项

- 禁止编造本地不存在的 Topic、Service、Action、frame、joint、模型或函数。
- 禁止在未读代码前直接重写节点。
- 禁止把 ROS 1、不同 ROS 2 发行版或不同 Gazebo 版本的代码直接混用。
- 禁止绕过 MoveIt 2、ros2_control、碰撞检查或限位直接控制真机。
- 禁止在回调中反复加载 YOLO 模型或执行长期阻塞任务。
- 禁止用硬编码坐标补偿掩盖 TF、标定或单位错误。
- 禁止为了“先跑起来”静默吞掉异常、关闭安全功能或删除用户代码。
- 禁止在没有运行证据时宣称问题已经完全解决。
