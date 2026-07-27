# RenderMumu

RenderDoc v1.46 改名版，用于捕获 MuMu 模拟器的 Vulkan 渲染进程 `MuMuVMMHeadless.exe`。

上游项目：[baldurk/renderdoc](https://github.com/baldurk/renderdoc)（MIT，见 [LICENSE.md](LICENSE.md)）

---

## 一、构建

**环境**：Visual Studio 2022（自动使用 v143 工具集，无需 v140）。Qt / Python / SWIG 已随源码树打包，无需额外安装。

构建整个解决方案：

```powershell
& "C:\Program Files\Microsoft Visual Studio\2022\Community\MSBuild\Current\Bin\MSBuild.exe" `
    "D:\RenderDocSource\renderdocmumu\renderdoc.sln" `
    /p:Configuration=Release /p:Platform=x64 /m
```

编译前需关闭模拟器和 `qrendermumu.exe`，否则 DLL 被占用会导致链接失败。

**产物**（`x64\Release\`）：

| 文件 | 说明 |
|---|---|
| `qrendermumu.exe` | UI 主程序 |
| `rendermumu.dll` | 核心捕获引擎 |
| `rendermumucmd.exe` | 命令行工具 |
| `rendermumushim64.dll` | 全局 Hook shim |
| `rendermumu.json` | Vulkan 层清单（构建时自动生成） |

---

## 二、首次配置（一次性，需管理员）

MuMu 的渲染进程由服务拉起，只能通过 Vulkan 隐式层捕获。

### 1. 设置机器级环境变量

```powershell
[Environment]::SetEnvironmentVariable('ENABLE_VULKAN_RENDERMUMU_CAPTURE','1','Machine')
```

**设置后需重启 Windows**，服务进程才能继承该变量。

### 2. 注册 Vulkan 层

在 `qrendermumu.exe` 中按提示注册即可。注册的是 JSON 的路径，修改 JSON 内容后无需重新注册。

### 3. 确认只注册了一个 `renderdoc` 层

MuMu 只加载层名含 `renderdoc` 的 Vulkan 层。若注册了多个（如官方 RenderDoc、其它改版），加载器可能选错。

检查：

```powershell
$k = Get-ItemProperty "HKLM:\SOFTWARE\Khronos\Vulkan\ImplicitLayers"
$k.PSObject.Properties | Where-Object { $_.Name -match '\.json$' } | ForEach-Object {
  if (Test-Path $_.Name) {
    $j = Get-Content $_.Name -Raw | ConvertFrom-Json
    "{0,-34} match={1,-6} {2}" -f $j.layer.name, ($j.layer.name -like '*renderdoc*'), $_.Name
  }
}
```

`match=True` 的应只有一条，且指向本项目的 `x64\Release\rendermumu.json`。删除多余项：

```powershell
Remove-ItemProperty -Path "HKLM:\SOFTWARE\Khronos\Vulkan\ImplicitLayers" -Name "<json完整路径>"
```

恢复：

```powershell
New-ItemProperty -Path "HKLM:\SOFTWARE\Khronos\Vulkan\ImplicitLayers" -Name "<json完整路径>" -PropertyType DWord -Value 0
```

---

## 三、抓帧

1. 完全退出模拟器。
2. 启动 `x64\Release\qrendermumu.exe`。
3. 从 UI 启动 MuMu，进入游戏渲染画面。
4. API 显示为 **Vulkan** 即可抓帧。

确认层已加载：

```powershell
Get-Process MuMuVMMHeadless | ForEach-Object { $_.Modules | Where-Object { $_.ModuleName -match 'rendermumu' } }
```

日志位置：`%TEMP%\RenderMumu\`

---

## 四、注意事项

修改以下内容会导致无法捕获或程序崩溃：

| 项目 | 要求 |
|---|---|
| Vulkan 层名 | 必须为 `VK_LAYER_RENDERDOC_Capture`（MuMu 只放行含 `renderdoc` 的层名）。定义在 `renderdoc/common/globalconfig.h` 与 `renderdoc/renderdoc.vcxproj` 的 `VulkanLayerName`，两处需一致 |
| `rendermumu.json` 的 `enable_environment` | 必须保留，否则捕获层会加载进 UI 自身导致崩溃 |
| replay marker | `renderdoc_replay.h` 中定义端与 `RDOC_BASE_NAME` 派生的检测端必须一致，否则 UI 崩溃 |
| `RENDERDOC_` 开头的 C API 导出符号 | 保持不变 |
| C++ 命名空间、源码目录、`#include` 路径、`3rdparty/` | 保持不变 |

层名与层的导出函数符号名是两套独立配置：层名为 `VK_LAYER_RENDERDOC_Capture`，导出符号为 `VK_LAYER_RENDERMUMU_Capture*`，两者在 JSON 中各自对应，不要统一。

---

## 五、常见问题

| 现象 | 处理 |
|---|---|
| API 显示 `None`，日志无 VMMHeadless 记录 | 检查环境变量是否已设并重启过系统；检查层注册（第二节） |
| VMMHeadless 加载了 `vulkan-1.dll` 但无 `rendermumu.dll` | 层名不含 `renderdoc`，或注册了多个同名层 |
| UI 启动即崩溃 | 检查 `rendermumu.json` 的 `enable_environment` 是否存在 |
| `LNK1181: breakpad_common.lib` | 需构建整个 `.sln`，不能单编 `renderdoc.vcxproj` |
| `LNK1104: rendermumu.dll` | DLL 被占用，关闭模拟器和 qrendermumu |

---

## 六、改名对照

核心 DLL 名由 `RDOC_BASE_NAME` 统一控制（`CMakeLists.txt` 与 `renderdoc/renderdoc.vcxproj`）。

| 原名 | 改后 |
|---|---|
| `renderdoc.dll` | `rendermumu.dll` |
| `qrenderdoc.exe` | `qrendermumu.exe` |
| `renderdoccmd.exe` | `rendermumucmd.exe` |
| `renderdocshim64.dll` | `rendermumushim64.dll` |
| `renderdoc__replay__marker` | `rendermumu__replay__marker` |
| `renderdocGLclass` | `rendermumuGLclass` |
| `RenderDocGlobalHookData64/32` | `RenderMumuGlobalHookData64/32` |
| `RENDERDOC_CRASHHANDLE` | `RENDERMUMU_CRASHHANDLE` |
| `RenderDocBreakpadServer` | `RenderMumuBreakpadServer` |
| `RenderDoc.RDCCapture.1` | `RenderMumu.RDCCapture.1` |
| `%TEMP%\RenderDoc\` | `%TEMP%\RenderMumu\` |
| `VK_LAYER_RENDERDOC_Capture*`（导出符号） | `VK_LAYER_RENDERMUMU_Capture*` |
| `ENABLE_VULKAN_RENDERDOC_CAPTURE` | `ENABLE_VULKAN_RENDERMUMU_CAPTURE` |
| 版本资源 ProductName `RenderDoc` | `RenderMumu` |

Android、Linux/macOS、安装包（`util/installer`）、CI 与文档中的名称未改动，PC 端抓帧不涉及。
