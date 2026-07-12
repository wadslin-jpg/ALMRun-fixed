# ALMRun 在线编译指南（GitHub Actions）

## 概述

利用 GitHub 提供的免费 Windows 云服务器，自动编译修改后的 ALMRun。
全程不需要安装任何软件，编译完成后直接下载 exe。

---

## 第一步：注册 GitHub 账号（已有可跳过）

1. 打开 https://github.com/signup
2. 填写用户名、邮箱、密码
3. 验证邮箱

---

## 第二步：创建新仓库

1. 登录 GitHub 后，点击右上角 **+** → **New repository**
2. Repository name 填：`ALMRun-fixed`
3. 选择 **Public**（必须公开，才能使用免费编译）
4. **不要**勾选 "Add a README file"
5. 点击 **Create repository**

---

## 第三步：上传代码到 GitHub

在你的电脑上打开命令行（按 Win+R，输入 cmd 回车），依次执行：

```cmd
cd C:\Users\wadsl\WorkBuddy\2026-07-12-21-49-50\ALMRun

git add -A
git commit -m "Add shutdown fix and GitHub Actions build"
git branch -M main
git remote add origin https://github.com/你的用户名/ALMRun-fixed.git
git push -u origin main
```

> 把"你的用户名"替换成你的 GitHub 用户名
> 第一次推送时会弹出登录窗口，按提示登录即可

---

## 第四步：等待自动编译

1. 推送代码后，GitHub 会自动开始编译
2. 进入你的仓库页面，点击顶部 **Actions** 标签
3. 可以看到正在运行的 "Build ALMRun (with shutdown fix)"
4. 编译大约需要 15-25 分钟（首次需要编译 wxWidgets）
   - 后续编译只需 2-3 分钟（wxWidgets 会被缓存）

如果编译失败：
- 点击进入失败的运行
- 查看红色 ❌ 的步骤
- 错误信息会显示在日志中

---

## 第五步：下载编译好的程序

1. 编译成功后（绿色 ✓），点击该次运行
2. 在页面底部找到 **Artifacts** 区域
3. 点击 **ALMRun-fixed-windows** 下载
4. 解压后即可使用！

压缩包包含：
```
ALMRun.exe              ← 修复后的主程序
ALMRunShutdownGuard.exe ← 关机守护程序（额外赠送）
lua51.dll               ← Lua 运行时
Everything32.dll        ← Everything 搜索支持
ALMRun.ini              ← 配置文件
config/                 ← 配置目录
LuaEx/                  ← Lua 扩展
skin/                   ← 皮肤
ShortCutList.txt        ← 快捷方式列表
```

---

## 手动触发编译

如果需要重新编译（比如修改了代码）：

1. 进入仓库 → **Actions** 标签
2. 左侧选择 "Build ALMRun (with shutdown fix)"
3. 点击右侧 **Run workflow** 按钮
4. 选择 main 分支，点击 **Run workflow**

---

## 常见问题

**Q: 推送代码时提示需要登录？**
A: GitHub 已不支持密码登录，需要使用 Personal Access Token：
   1. 打开 https://github.com/settings/tokens
   2. 点击 "Generate new token (classic)"
   3. 勾选 "repo" 权限
   4. 生成后复制 token，在命令行中作为密码粘贴

**Q: 编译失败了怎么办？**
A: 在 Actions 页面查看失败日志，把错误信息发给我，我帮你修。

**Q: 编译要收费吗？**
A: 公开仓库完全免费，每月有 2000 分钟的 Windows 编译额度，绰绰有余。

**Q: 可以下载给别人用吗？**
A: 可以，编译出来的 exe 没有任何限制，随便分发。
