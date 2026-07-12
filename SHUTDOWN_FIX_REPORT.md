# ALMRun 关机内存错误修复报告

## 问题描述

ALMRun 在 Windows 关机/注销时会弹出内存错误对话框。

## 根本原因分析

经过对源代码的深入分析，发现了 **4 个关键问题**：

### 问题 1：缺少 WM_QUERYENDSESSION / WM_ENDSESSION 处理（核心原因）

Windows 关机时，系统会向所有顶层窗口发送 `WM_QUERYENDSESSION` 消息。ALMRun 的代码中**完全没有**处理这个消息。

当前的处理流程：
1. Windows 发送 `WM_QUERYENDSESSION`
2. wxWidgets 默认处理器生成 `wxCloseEvent`
3. 由于 `EVT_CLOSE` 在事件表中被注释掉，默认处理器调用 `Destroy()`
4. `Destroy()` 将窗口加入 `wxPendingDelete`（**延迟销毁**）
5. 在延迟销毁真正执行之前，事件循环可能继续处理其他消息（paint、timer 等）
6. 这些消息的处理器访问即将被释放或已被释放的全局对象 → **内存错误**

### 问题 2：OnClose() 中的析构顺序错误

```cpp
// 原来的顺序（有问题）：
wxDELETE(g_lua);      // 1. 先删除 Lua 状态
wxDELETE(g_commands); // 2.
wxDELETE(g_config);   // 3.
wxDELETE(g_hotkey);   // 4.
wxDELETE(g_timers);   // 5. 后删除定时器
wxDELETE(g_controller); // 6.
```

`MerryTimer::Notify()` 中直接访问 `g_lua->GetLua()` 和 `g_timers->ClearTimer(this)`，**没有任何空指针检查**。如果定时器在 `g_lua` 被删除后、`g_timers` 被删除前触发，就会崩溃。

### 问题 3：OnShowEvent 缺少空指针检查

```cpp
// 原来的代码（没有检查）：
if (g_config->get(PlayPopupNotify))     // g_config 可能为 NULL
    ...
g_controller->SetWindowPos(...)          // g_controller 可能为 NULL
```

关机过程中，`OnClose()` 删除全局对象后，如果 `OnShowEvent` 被触发（例如 `EvtActive` 调用 `Hide()`），就会访问已释放的内存。

### 问题 4：EvtActive 在关机时触发 Hide()

`MerryApp::EvtActive` 在应用失去焦点时调用 `m_frame->Hide()`，这会触发 `OnShowEvent`。关机时应用必然失去焦点，此时全局对象可能已被释放。

---

## 修复方案

### 修复 1：拦截 WM_QUERYENDSESSION / WM_ENDSESSION（MerryFrame.h + MerryFrame.cpp）

在 `MerryFrame` 中重写 `MSWWindowProc`，直接拦截关机消息：

```cpp
#ifdef __WXMSW__
WXLRESULT MerryFrame::MSWWindowProc(WXUINT message, WXWPARAM wParam, WXLPARAM lParam)
{
    if (message == WM_QUERYENDSESSION)
    {
        // 同步执行清理，不走延迟销毁路径
        if (!m_isClosing && g_lua)
            OnClose();
        return TRUE; // 允许关机
    }
    if (message == WM_ENDSESSION)
    {
        if (!m_isClosing && g_lua)
            OnClose();
        return 0;
    }
    return wxFrame::MSWWindowProc(message, wParam, lParam);
}
#endif
```

**效果**：关机时同步执行 `OnClose()` 清理所有资源，绕过 wxWidgets 的延迟销毁机制，避免事件循环在清理后继续运行。

### 修复 2：添加 m_isClosing 标志（MerryFrame.h + MerryFrame.cpp）

```cpp
// MerryFrame.h
bool m_isClosing;

// MerryFrame.cpp 构造函数
m_isClosing = false;

// OnClose() 开头
m_isClosing = true;

// OnShowEvent() 开头
if (m_isClosing)
    return;
```

**效果**：一旦开始关闭流程，所有后续的事件处理立即返回，不再访问可能已释放的全局对象。

### 修复 3：修正 OnClose() 析构顺序（MerryFrame.cpp）

```cpp
// 修复后的顺序：
wxDELETE(g_timers);      // 1. 先删除定时器（停止所有定时器回调）
wxDELETE(g_lua);         // 2. 再删除 Lua 状态
wxDELETE(g_commands);    // 3.
wxDELETE(g_config);      // 4.
wxDELETE(g_hotkey);      // 5.
wxDELETE(g_controller);  // 6.
```

**效果**：先停止所有定时器，确保不会有定时器回调在 Lua 状态被释放后访问它。

### 修复 4：MerryTimer::Notify() 添加空指针检查（MerryTimer.cpp）

```cpp
void MerryTimer::Notify()
{
    if (!g_lua || !g_timers)
        return;
    // ...
}
```

**效果**：双重保险，即使定时器在关机过程中意外触发，也不会崩溃。

### 修复 5：OnShowEvent 添加空指针检查（MerryFrame.cpp）

```cpp
if (g_config && g_config->get(PlayPopupNotify))  // 添加 g_config 检查
if (g_controller)                                  // 添加 g_controller 检查
    g_controller->SetWindowPos(...)
if (g_config && g_config->get(AutoPopup))         // 添加 g_config 检查
```

### 修复 6：EvtActive 添加 g_config 检查（MerryApp.cpp）

```cpp
void MerryApp::EvtActive(wxActivateEvent &e)
{
    if (!m_frame)
        return;
    if (!g_config)    // 关机时 g_config 已被释放，直接返回
        return;
    // ...
}
```

---

## 修改的文件清单

| 文件 | 修改内容 |
|------|----------|
| `src/MerryFrame.h` | 添加 `MSWWindowProc` 声明、`m_isClosing` 成员变量 |
| `src/MerryFrame.cpp` | 添加 `MSWWindowProc` 实现、`m_isClosing` 逻辑、修正析构顺序、添加空指针检查、添加 `<windows.h>` |
| `src/MerryTimer.cpp` | `Notify()` 添加 `g_lua`/`g_timers` 空指针检查 |
| `src/MerryApp.cpp` | `EvtActive()` 添加 `g_config` 空指针检查 |

## 编译说明

修复后的代码需要与原项目相同的编译环境：
- Visual Studio 2012+
- CMake >= 2.8
- wxWidgets >= 2.9.5 (建议 3.0.1)

编译步骤：
```
cd ALMRun
cd Build
cmake ..
打开 ALMRun.sln 编译
```
