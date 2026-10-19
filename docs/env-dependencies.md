# 环境依赖清单（共用 gxu2026-nav-env 镜像）

本文回答一个问题：本项目**不构建自己的镜像**，直接共用另一个仓库（`ros2_ws`）的环境镜像时，镜像已经给了什么、还差什么。

结论先行：

- **编译依赖零缺口**：`CMakeLists.txt` 里全部 `find_package` 在镜像内都有提供者，已实测完整构建通过（见 §2、§7）。
- **真正要自己补的只有三件**：把源码挂进容器、隔离构建产物、运行时显示环境（见 §3）。
- **`package.xml` 已补齐**：原先只声明 3 个依赖，现已按 `CMakeLists.txt` 与 `default.rviz` 全量补成可移植声明，`rosdep install` 换机器不会再缺包（见 §3.3）。
- **一处镜像运行期缺口**：`default.rviz` 引用的 `slam_toolbox` rviz 插件镜像里没装，启动时会产生一条找不到类的错误（见 §3.3、§4）。
- **一条已知缺陷与依赖无关**：`RegionTool::updateProperties()` 只有声明没有定义，动态库内留未解析符号（见 §7）。

## 1. 口径与事实来源

| 项 | 事实 |
|---|---|
| 镜像来源 | 另一个仓库的 `docker/Dockerfile.env`，本项目不维护 |
| 镜像 tag | `{arch}{YYYYMMDDHHMI}` 精确版本 + `{arch}-latest` 浮动版本 |
| 本文实测对象 | `alphabet2006/gxu2026-nav-env:amd64-latest`（本地另有 `amd64202609091703`） |
| 镜像内 rosdep 清单 | 由那个仓库的 `src/` 派生（37 个包），**不包含 map_edit** |
| 本项目的依赖声明 | `package.xml` + `CMakeLists.txt`，其中以 `CMakeLists.txt` 为准 |

要点：本项目能编，是因为镜像**显式安装了 rviz2 / Qt5 / PCL / OpenCV / yaml-cpp / Ogre** 这些基础件，而不是因为镜像的依赖清单覆盖了 map_edit。也就是说，镜像的依赖清单对本项目没有保护作用（见 §5）；`package.xml` 的补齐只对「按 rosdep 换机器」有保护作用。

## 2. 镜像已提供（逐项实测）

| 本项目需求 | 镜像内提供者 | 实测版本 / 路径 |
|---|---|---|
| `ament_cmake` | `ros-humble-ament-cmake` | 1.3.14 |
| `rclcpp` | `ros-humble-rclcpp` | 16.0.19 |
| `rviz_common` | `ros-humble-rviz-common` | 11.2.28，头文件在 `/opt/ros/humble/include/rviz_common/` |
| `rviz_rendering` | `ros-humble-rviz-rendering` | 11.2.28 |
| `pluginlib`（`pluginlib_export_plugin_description_file`） | `ros-humble-pluginlib` | 5.1.4 |
| `Qt5::Widgets` | `qtbase5-dev` | 5.15.3 |
| `Qt5::Quick` / `Qt5::Qml` / `Qt5::QuickWidgets` | `qtdeclarative5-dev` | 5.15.3，`cmake/Qt5Quick`、`Qt5Qml`、`Qt5QuickWidgets` 三处 config 均存在 |
| `PCL`（common、io） | `libpcl-dev` | 1.12.1 |
| `yaml-cpp` | `libyaml-cpp-dev` | 0.7.0，`/usr/include/yaml-cpp/yaml.h` |
| `OpenCV`（经 `cv_bridge` 传递） | `libopencv-dev` | 4.5.4，`/usr/include/opencv4/opencv2/opencv.hpp` |
| `Ogre` 头文件（`OgrePlane.h` / `OgreVector3.h`） | `rviz_ogre_vendor`（镜像内自带） | 1.12.1，`/usr/include/OGRE/`；运行库在 `/opt/ros/humble/opt/rviz_ogre_vendor/lib/` |
| `pcl_conversions` | `ros-humble-pcl-conversions` | 2.4.5 |
| `cv_bridge` | `ros-humble-cv-bridge` | 3.2.1 |
| `tf2` / `tf2_ros` / `tf2_geometry_msgs` | `ros-humble-tf2*` | 0.25.22 |
| `nav_msgs` / `sensor_msgs` / `geometry_msgs` / `visualization_msgs` | `ros-humble-*-msgs` | 4.9.2 |
| `ament_index_cpp` | `ros-humble-ament-index-cpp` | 1.4.1（注意：源码实际未使用，见 §3.3） |
| 运行期 `rviz2` | `ros-humble-rviz2` | 11.2.28 |
| 构建工具链 | `cmake` / `g++` / `ninja` / `ccache` / `colcon` | 3.22.1 / 11.4.0 / 1.10.1 / 4.5.1 / `python3-colcon-common-extensions` |

一个容易踩的坑：源码实际只 `#include` 了 Qt Widgets 的控件，**没有**用 QtQuick；但 `CMakeLists.txt` 里 `find_package(Qt5 REQUIRED COMPONENTS Widgets Quick Qml QuickWidgets)` 会硬性要求这三份 CMake config。本镜像有 `qtdeclarative5-dev`，所以配置能过；换到只装 `qtbase5-dev` 的镜像会直接 configure 失败。

## 3. 需要自己补的

### 3.1 把源码挂进容器（唯一硬缺口）

共用镜像的 compose 只挂那个仓库自己的固定路径（`src`、`scripts`、`.buildcache`、`maps`、`launch_logs` 等），`map_edit` 位于 `ros2_ws` 之外，容器内看不到。

| 方案 | 做法 | 代价 |
|---|---|---|
| A（推荐） | 启动容器时附加一卷：`-v /home/alphabet/test/map_edit:/ws/external/src/map_edit` | 不改镜像、不改那个仓库 |
| B | 在共用镜像的 `docker/compose.dev.yml` 里加一行挂载 | 属于改那个仓库，要走它的改动流程 |
| 不推荐 | 复制到 `/home/alphabet/test/ros2_ws/src/` 下 | 会被那个仓库的 colcon 扫到、被它的 `sync_dep_manifest.sh` 纳入依赖清单、进入它的自检与提交范围 |
| 禁止 | 在宿主机建符号链接指向本项目 | 容器内没有宿主机路径，链接无法解析 |

挂到 `/ws/external/src/` 之后，colcon 需要显式指定扫描根目录（见 3.2），不能依赖工作目录。

### 3.2 隔离构建产物

不要用 `/ws/build`、`/ws/install`：那是共用镜像那个仓库的产物目录，它有自己的分支隔离与构建戳（`.buildcache/<分支>/`），混进去会干扰它的分支一致性自检。

```bash
source /opt/ros/humble/setup.bash
colcon build \
  --base-paths /ws/external/src \
  --build-base /ws/external/build \
  --install-base /ws/external/install \
  --symlink-install
```

运行前按顺序 source：先 `/opt/ros/humble/setup.bash`，再 `/ws/external/install/setup.bash`。若同时要跑那个仓库的导航栈，它的 overlay 在 `.buildcache/<分支>/install/setup.bash`，两边不要互相覆盖 `COLCON_PREFIX_PATH` 的顺序问题（本项目只依赖 `/opt/ros/humble`，没有反向依赖）。

### 3.3 `package.xml` 声明（已补齐）

原始状态的 `package.xml` 只声明了 `rviz_common`、`yaml-cpp`、`pcl_conversions` 三项，而 `CMakeLists.txt` 有 16 个 `find_package`。在当前镜像里没有影响（全已安装），但按 README 的 `rosdep install` 流程换机器时会缺包、在 configure 阶段失败。

现已按源码实际 include 补齐：

| 分类 | 依赖 | 源码证据 |
|---|---|---|
| 编译/链接 | `rclcpp` | `map_eraser_tool.h:13` |
| 编译/链接 | `rviz_common`、`rviz_rendering` | `rviz_common/tool.hpp`、`rviz_rendering/viewport_projection_finder.hpp` |
| 编译/链接 | `pluginlib` | 三个 cpp 的 `class_list_macros.hpp` |
| 编译/链接 | `tf2`、`tf2_ros`、`tf2_geometry_msgs` | `map_eraser_tool.h:11,12,14,24`、`map_edit_panel.h:37,47` |
| 编译/链接 | `cv_bridge` | `map_eraser_tool.h:25`（仅 include，源码无调用） |
| 编译/链接 | `nav_msgs`、`sensor_msgs`、`geometry_msgs`、`visualization_msgs` | 各头文件的消息类型 |
| 编译/链接 | `pcl_conversions`、`yaml-cpp` | `pcd_loader.cpp:4`、`map_edit_panel.h:43` |
| 运行期 | `launch`、`launch_ros`、`rviz2` | `launch/map_edit.launch.py:1-3` |
| 运行期 | `nav2_rviz_plugins`、`slam_toolbox` | `rviz/default.rviz:33,35` 引用的两个插件的包 |

**运行期缺口的补充说明**：`rviz/default.rviz` 引用了 `nav2_rviz_plugins/Navigation 2` 与 `slam_toolbox::SlamToolboxPlugin`。前者在镜像中存在（`ros-humble-nav2-rviz-plugins` 1.1.20），后者**镜像未安装**（`ros-humble-slam-toolbox-rviz-plugin` 缺失）。rviz 载入配置时会因找不到该类而报错并跳过该面板，不影响 MapEditPanel 本身，但会产生一条启动错误。已把两者写入 `package.xml` 的 `exec_depend`，换机器走 rosdep 可自动补上。

**未写入 `package.xml` 的依赖**：`PCL`（`libpcl-dev`）、`Qt5`（`qtbase5-dev`、`qtdeclarative5-dev`）、`OpenCV`、`Ogre`。这些是系统包，rosdep 用 `pcl`、`qt5-qmake` 等 key 映射，换发行版行为可能不同；本项目走「共用镜像已装齐」这一前提，故不声明。

**已知的可删依赖**：`ament_index_cpp` 在 CMake 里 `find_package` 且进了 `ament_target_dependencies`，但 `src/`、`include/` 内**没有任何使用**。它没有进 `package.xml`，属冗余声明，删掉不影响构建。

### 3.4 运行时

- `rviz2` 需要 X11：容器需挂 `/tmp/.X11-unix` 与 xauth 文件，并设置 `DISPLAY`、`XAUTHORITY`、`QT_X11_NO_MITSHM=1`。共用镜像的宿主机侧用 systemd timer 维护 xauth 文件（每 5 秒刷新）。
- 机器人档容器（无 GPU 透传）里 rviz 走 mesa 软件渲染；本插件在 rviz 里绘制 PCL 与 marker，软件渲染可用但帧率低，编辑大图时优先用有 GPU 的笔记本档。
- `launch/map_edit.launch.py` 只起 `rviz2` 一个节点，加载 `share/map_edit/rviz/default.rviz`；插件通过 `/map` 话题取图，因此需要有地图发布方（导航栈或 `map_server`）在线，否则面板里没有可编辑内容。
- `default.rviz` 里的 `slam_toolbox::SlamToolboxPlugin` 在当前镜像中**不可用**（包未安装）。启动时 rviz 会打印一条找不到该类的错误并跳过该面板；若需该面板，在容器内临时装 `ros-humble-slam-toolbox`（容器重建即丢），或让镜像侧补装。

## 4. 镜像明确不含、但对本项目无影响

| 不含项 | 说明 | 对本项目 |
|---|---|---|
| `slam_toolbox` rviz 插件 | `ros-humble-slam-toolbox` 未安装 | **有影响**：`default.rviz` 引用了它，启动时报一条找不到类的错误（见 §3.4） |
| CUDA / cuDNN 运行时 | 只做 GPU 透传声明，不装 CUDA 工具链；需要 GPU 推理要换基础镜像 | 无关（纯 CPU + OpenGL） |
| NeuPAN Python 运行时 | 由独立的 `neupan-runtime` 容器提供，不在环境镜像里 | 无关 |
| 图像编辑类外部工具 | 例如 GIMP 之类 | 无关（本插件就是替代品） |

## 5. 版本口径与更新路径

- 镜像有精确 tag 与 `{arch}-latest` 浮动 tag 两种；生产上要可回滚就锁精确 tag，日常开发跟 `latest`。
- 镜像的 rosdep 清单由那个仓库的 `src/` 派生，**不会因为本项目新增依赖而变化**。本项目若引入新依赖，只有两条路：让镜像侧改 `Dockerfile.env` 并重建，或在容器里临时安装（容器重建即丢）。
- 镜像内 ROS 2 / rviz 版本约为 2026-07 至 2026-08 的构建；Qt 为 5.15.3。若镜像升级到 Qt6 版 rviz2，`CMakeLists.txt` 的 `Qt5` 相关 `find_package` 与 `qt5_wrap_cpp` 都需要改。

## 6. 自检命令

```bash
# 只读核对镜像内容：不启动开发容器，不联网，退出即删
docker run --rm --network none --entrypoint bash \
  alphabet2006/gxu2026-nav-env:amd64-latest -c \
  'dpkg -l | grep -E "ros-humble-(rviz-common|rviz-rendering|pluginlib|pcl-conversions)"; ls /usr/lib/x86_64-linux-gnu/cmake/Qt5Quick/Qt5QuickConfig.cmake /usr/lib/x86_64-linux-gnu/cmake/Qt5QuickWidgets/Qt5QuickWidgetsConfig.cmake /usr/include/yaml-cpp/yaml.h /usr/include/opencv4/opencv2/opencv.hpp /usr/include/OGRE/OgrePlane.h'

# 容器内、编译前确认六件事都能解析
source /opt/ros/humble/setup.bash
ros2 pkg prefix rviz_common
ros2 pkg prefix rviz_rendering
ros2 pkg prefix pluginlib
pkg-config --modversion yaml-cpp
ls /usr/lib/x86_64-linux-gnu/cmake/Qt5QuickWidgets/Qt5QuickWidgetsConfig.cmake
ls /usr/include/opencv4/opencv2/opencv.hpp

# 挂载与产物落点确认（应指向本项目目录，而不是那个仓库）
ls -d /ws/external/src/map_edit

# 依赖声明与实际使用是否一致（无输出表示 ament_index_cpp 确实未被使用）
grep -rn "ament_index" src include || echo "ament_index_cpp 未被使用"
```

## 7. 实测记录

### 7.1 已完成的验证

在 `alphabet2006/gxu2026-nav-env:amd64-latest` 内、源码只读挂载到 `/ws/external/src/map_edit`、构建产物落 `/tmp`，执行：

```bash
source /opt/ros/humble/setup.bash
colcon build --base-paths /ws/external/src \
  --build-base /tmp/ext/build --install-base /tmp/ext/install --cmake-args -G Ninja
```

结果：

| 检查项 | 结果 |
|---|---|
| `colcon build` 退出码 | 0 |
| 编译告警（`-Wall -Wextra -Wpedantic`） | 0 条 |
| 安装产物 | `lib/libmap_edit.so`、`share/map_edit/{launch,rviz}` 齐全 |
| 插件资源索引 | `share/ament_index/resource_index/rviz_common__pluginlib__plugin` 已生成 |
| `ldd` 解析 | 全部找到：`librviz_common`/`librviz_rendering`（`/opt/ros/humble/lib`）、`libOgreMain.so.1.12.1`（`rviz_ogre_vendor`）、`libpcl_*`、`libyaml-cpp`、`libQt5*` |
| rviz2 离屏启动（Xvfb + `LIBGL_ALWAYS_SOFTWARE=1`） | `OpenGl version: 4.5 (GLSL 4.5)`，无加载错误，存活到 30s 超时 |

CMake 配置期有一条开发级警告（非错误）：`CMP0074 is not set ... PCL_ROOT is set to /usr`，由 rviz_ogre_vendor 或 PCL 的 config 设置 `PCL_ROOT` 引起，不影响构建，加 `-Wno-dev` 可消除。

### 7.2 与依赖无关的已知缺陷

`RegionTool::updateProperties()` 在 `include/map_edit/region_tool.h:62` 声明为槽，在 `src/region_tool.cpp:54,58,64,70,74,78` 被 `QObject::connect` 引用，但**整个文件没有定义**。

链接不会失败（动态库允许未定义符号），`ldd -r` 会报：

```
undefined symbol: _ZN8map_edit10RegionTool16updatePropertiesEv
```

**推断**（未验证）：rviz 加载插件时对未定义符号比较宽松，属性变更时连接会静默失效，导致 Color/Alpha 改动后 polygons 不刷新。需要真实交互才能确认，本次未做。

### 7.3 尚未验证

- 真实 X11 下的交互行为（绘制区域、拖动顶点、保存 YAML）未测，仅验证了启动不报错。
- 「机器人档软件渲染可接受」仍是推断：镜像装了 mesa 与 `mesa-utils`，但未实测 rviz 软件渲染下本插件的帧率。
