# CMake 构建环境与当前状态

## 当前进度

- 截至 2026-08-27，项目 `x64-Debug` 已完成 Visual Studio CMake generation；`build.ninja`、`compile_commands.json`、CMake code model 和 include paths 均已生成。
- CMake 已识别 MSVC 19.51.36246.0、Qt 6.11.0、Arrow/Parquet 25.0.1、Npcap PCAP/Packet 库和 vcpkg 依赖。
- CMake 目标已存在：`SignalParser`、`test_someip_debug`、`print_someip_schema`。
- `x64-Debug` 已完成编译阶段，但链接 `SignalParser.exe` 失败；`build_error.txt` 中有 322 个 `LNK2038`、1 个 `LNK4098` 和汇总错误 `LNK1319`。
- 322 个不匹配均由 Debug/Release ABI 混链引起：Debug 对象为 `/MDd`、`_ITERATOR_DEBUG_LEVEL=2`，而 `mdf.lib` 与当前 PcapPlusPlus 二进制为 `/MD`、`_ITERATOR_DEBUG_LEVEL=0`。
- 用户已确认 `x64-Release` Build 成功；当前 `out/build/x64-Release/build.ninja`、`compile_commands.json` 和 `SignalParser.exe` 均存在。
- `SignalParser.exe` 的直接非系统依赖已用 `dumpbin /dependents` 检查，Qt6、SQLite、Arrow/Parquet、pugixml、expat、zlib 和 Npcap DLL 均已位于 Release 输出目录。
- 补齐运行库后是否已经成功启动 GUI，当前**没有记录**；仍需用户实际运行确认。

## 已确认的环境与决定

- Qt 根目录为 `C:/_Storage/00_base/msvc2022_64/msvc2022_64`；vcpkg 根目录为 `C:/_Storage/00_base/vcpkg`；Npcap SDK 根目录为 `C:/_Storage/00_base/npcap-sdk-1.16`。
- vcpkg 已安装 Arrow（含 Parquet）、pugixml、zlib、expat、rapidjson 等项目依赖；项目使用 `x64-windows` triplet。
- 离线 PCAP/PCAPNG 分析仍使用 PcapPlusPlus 的 `PcapFileDevice`/`IFileReaderDevice`，因此构建期仍需要 Npcap SDK 的头文件和 `.lib`，不能仅因不做实时抓包就移除 PCAP 依赖。
- `CMakeSettings.json` 使用固定的当前机器路径，避免 Visual Studio 进程未刷新用户环境变量或旧 CMakeCache 路径覆盖新配置。
- 项目内 PcapPlusPlus 包名及内容表明它只有 VS2022 x64 Release 静态库；当前不应让 `x64-Debug` 链接这些库，也不应通过 `/NODEFAULTLIB` 忽略 ABI 冲突。
- `CMakeSettings.json` 已新增 `x64-Release`，作为当前可用依赖组合的标准构建配置。
- 首次切换 `x64-Release` 时 Targets 视图为空，并非 `SignalParser` 定义丢失；该 build tree 没有生成 `build.ninja` 或 CMake code model，CMake File API 返回 `no buildsystem generated`。
- `CMakeLists.txt` 会校验 vcpkg/Npcap 路径，修正无效的旧 toolchain cache，并从项目根目录或 Windows System32 查找 Npcap DLL；找到后在构建后复制到目标目录，方便离线分析程序启动。
- `QT6_ROOT` 只用于 CMake 查找 Qt，不会把 Qt `bin` 加入 Windows 运行时 DLL 搜索路径；SQLite 源码目录同样不属于运行时搜索路径。
- 运行时缺失 DLL 的根因是 Windows DLL 搜索路径不包含项目 `lib/sqlite-dll-win-x64-3530100` 和 Qt `bin`；设置 `QT6_ROOT` 本身不能解决运行时加载。
- `CMakeLists.txt` 已为 `SignalParser` 增加 POST_BUILD 部署：复制 `sqlite3.dll`、用 `$<TARGET_RUNTIME_DLLS:SignalParser>` 复制 Qt/其他已知共享库，并复制 `platforms/qwindows.dll`。
- 当前采用 CMake 原生部署规则而不是依赖 `windeployqt`，以避免部署步骤依赖外部 Qt 工具进程；这样至少覆盖主程序启动所需 Qt DLL 和 Windows 平台插件。

## 曾失败但已定位的方案

- 早期 generation 失败的原因是 Qt、vcpkg、Npcap 的旧机器绝对路径不存在，导致没有生成 buildsystem；已改为当前路径并成功 generation。
- vcpkg 初次安装 Arrow 时 rapidjson 下载 GitHub 源码超时，并曾遇到 vcpkg 目录写权限问题；后续 `vcpkg list` 已确认相关包安装完成。
- 直接从普通 PowerShell 调用 CMake 曾出现 `LIBCMT.lib`/环境继承或命令超时问题；Visual Studio 的 `inheritEnvironments: msvc_x64_x64` generation 已成功，因此后续应优先从 Visual Studio 配置/构建，不要把普通 shell 的失败误判为 CMakeLists 失败。
- 代理侧尝试用普通 shell 调用 Visual Studio CMake / `VsDevCmd.bat` 重新配置或构建时多次超时；这不代表 Visual Studio 内部构建失败，用户随后已确认 `x64-Release` Build 成功。
- 代理侧直接调用 `windeployqt.exe` 失败，报 `Unable to query qtpaths: Error running binary qtpaths: pipe: The system cannot find the file specified.`；直接运行 `qtpaths.exe` 本身成功，因此该失败归因于当前执行环境对子进程调用的限制，改用 CMake 原生 DLL/插件复制规则。

## 关键文件与当前风险

- 关键配置文件：`CMakeLists.txt`、`CMakeSettings.json`、`.vscode/settings.json`、`.gitignore`；项目根目录还放有 `Packet.dll` 和 `wpcap.dll`。
- 当前 generation 有两个非阻塞提示：`CMP0144` 的 `QT6_ROOT` 大写变量兼容性警告，以及可选 `WrapVulkanHeaders` 缺失。
- Release 输出目录已确认存在 `SignalParser.exe`、Npcap DLL、vcpkg 运行库；2026-08-27 又补齐了 `sqlite3.dll`、Qt6Core/Gui/Widgets/Network DLL 和 `platforms/qwindows.dll`。
- 当前 POST_BUILD 规则只明确部署 Qt 运行库和 `platforms/qwindows.dll`；如果程序后续使用 HTTPS、图片格式或其他 Qt 插件，可能还需要部署 `tls`、`imageformats` 等插件目录。
- 失败的 `out/build/x64-Release` 曾包含命令行测试写入的 `CMAKE_CXX_COMPILER_FORCED` 等残留缓存；2026-08-27 已确认无构建进程后删除整个可再生 Release build tree，下一次应由 Visual Studio 重新 generation。
- 根目录 Npcap DLL 是第三方二进制，是否提交远程仓库需按项目发布/许可证策略决定；`out/` 已加入忽略，不应提交。
- 本次检查时根目录中的 `build_error.txt` 和 `build_release_configure.log` 均不存在；它们只能作为此前会话中的历史文件，不能作为当前状态依据。

## 下一步

1. 首先直接运行 `out/build/x64-Release/SignalParser.exe`，确认补齐 DLL 后 GUI 是否成功启动；若仍失败，记录新的完整错误文本。
2. GUI 启动后，用离线 PCAP/PCAPNG、MDF 和 SQLite/Arrow 相关流程做最小运行验证。
3. 若运行时提示缺少 Qt 插件，再补充部署 `tls`、`imageformats` 等实际需要的插件；不要把整个 Qt 根目录加入项目仓库。
4. 如果必须调试第三方库边界，则需要另行获取或编译 Debug 版 PcapPlusPlus，并让 MDF、pugixml、expat 等库按配置选择 Debug 版本；不能继续混用 Release C++ 静态库。
5. 若构建成功但启动项仍为空，再将 `SignalParser` 设为 Startup Item，必要时清理 Visual Studio `.vs` 状态并重新打开项目；不要编辑 `CMakeConfigureLog.yaml`。
