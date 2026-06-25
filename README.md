# MD Reader

一个面向 Windows 的 Markdown 阅读器，基于 [Neilooo/md-reader](https://github.com/Neilooo/md-reader) 修改维护。

这个版本保留了原项目轻量、快速、所见即所得预览的特点，并在界面、标签页、源码编辑、右键菜单、状态栏和导出体验上做了二次优化。

## 主要功能

- 打开单个 Markdown 文件，或打开文件夹浏览多个 Markdown 文件
- 像浏览器一样用标签页切换多个文档
- 支持预览模式和源码模式，源码模式可以编辑并保存
- 支持 Markdown 常用语法、代码高亮、表格、引用块、任务列表、脚注等
- 支持 KaTeX 数学公式和 Mermaid 图表
- 支持当前文档查找和文件夹全文搜索
- 支持大纲跳转、文件树、侧栏显示隐藏
- 支持日间/夜间主题和中英文界面切换
- 支持记录窗口大小、窗口位置和阅读进度
- 支持导出为 PDF、HTML、TXT
- 支持底部状态栏显示当前文件路径、大小和字数信息

## 快捷键

| 快捷键 | 作用 |
| --- | --- |
| `Ctrl+F` | 查找当前文档 |
| `Ctrl+Shift+F` | 在当前文件夹里全文搜索 |
| `Ctrl+,` | 打开阅读设置 |
| `Ctrl+S` | 在源码模式下保存当前文件 |
| `Ctrl+E` | 打开或关闭导出菜单 |
| `Ctrl+R` | 重新读取当前文件 |
| `Ctrl+P` | 打印 |
| `Esc` | 关闭当前弹出的查找框、设置面板或右键菜单 |

## 右键菜单

正文区域右键菜单提供：

- 查找当前文档
- 重新读取文件
- 复制选中文字
- 复制文件路径
- 导出 PDF、TXT、HTML
- 打印

标签页右键菜单提供：

- 刷新切页
- 关闭当前切页
- 关闭其他切页
- 复制文件路径

空白区域不会弹出自定义右键菜单。

## 运行和开发

安装依赖：

```bash
pnpm install
```

开发运行：

```bash
pnpm tauri dev
```

生成正式版：

```bash
pnpm tauri build
```

编译完成后，绿色版程序一般在：

```text
src-tauri/target/release/md-reader.exe
```

安装包一般在：

```text
src-tauri/target/release/bundle/nsis/
```

## 开发环境

需要提前准备：

- Node.js
- pnpm
- Rust
- Windows WebView2 运行环境
- Visual Studio 生成工具，安装“使用 C++ 的桌面开发”相关组件

Windows 10/11 通常已经自带 WebView2。如果缺失，可以安装 Microsoft WebView2 Evergreen Runtime。

## 技术栈

- 桌面壳：Tauri 2
- 前端：Vue 3、TypeScript、Vite
- Markdown 渲染：markdown-it
- 代码高亮：highlight.js
- 数学公式：KaTeX
- 图表：Mermaid
- 国际化：vue-i18n

## 项目来源

本项目根据 [Neilooo/md-reader](https://github.com/Neilooo/md-reader) 修改维护。

原项目采用 MIT 许可证，本项目继续保留 MIT 许可证。

## 许可证

MIT
