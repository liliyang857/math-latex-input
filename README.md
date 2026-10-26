# math-latex-input

Vue 3 + TypeScript + Vite 项目，使用本地 TinyMCE **6.0.0**，接入从 `../math/src` 构建的公式编辑器 SDK **0.2.31**。

**[在线演示](https://yjt151.github.io/math-latex-input/)**

## 演示部署

推送到 `main` 后，GitHub Actions 会自动构建并部署到 GitHub Pages。仓库的 Settings → Pages → Source 应选择 **GitHub Actions**。部署进度可在仓库 Actions 中查看。

## 启动

需要 Node.js 22.12+ 和 pnpm。

```sh
pnpm install --frozen-lockfile
pnpm dev
```

```sh
pnpm typecheck
pnpm build
pnpm preview
```

## 使用

- 选择“SDK 返回形式”：LaTeX 公式或 IMG 图片（PNG），再点击工具栏“插入公式”。图片模式使用 SDK 的 `outputType: 'img'`，将返回的 `result.data`（含 PNG data URL 的 `<img>` HTML）插入正文。
- 在可视化或 LaTeX 模式编辑，确认后插入正文；双击已有公式可修改。
- TinyMCE 管理插入位置及撤销/重做，SDK 负责公式的 MathJax SVG 展示。
- 双击修改保留原公式形式，PNG 图片的 `data-latex` 用于回填源码。
- “导出格式”默认 HTML，保留公式节点和 PNG 图片；选择 LaTeX 时，将两种公式均转为 `$...$`。目前仅展示导出结果，未连接存储接口。

入口为 `src/App.vue`。TinyMCE 使用 inline 模式，与公式 SDK 共用 document；工具栏、主题、图标、插件和样式均由 npm 包本地加载，无需 CDN 或 Tiny Cloud API Key。不可编辑节点由 TinyMCE 6 核心支持，无需加载旧版 noneditable 插件。

## 公式包

`vendor/formula-editor-modal-0.2.31.tgz` 由源项目的 `src/sdk/index.ts` 入口构建，组件、渲染引擎与字体均包含在包内。本项目依赖本地 tgz，无需复制源码，安装后也无需保留相邻的 `math` 目录。

更新 SDK 时，在源项目构建后将新版本包输出到本项目（以下命令在 `math` 目录执行）：

```sh
npm run build:sdk
npm pack --pack-destination ../math-latex-input/vendor
```

源项目先提升版本，随后在本项目运行 `pnpm add ./vendor/formula-editor-modal-新版本.tgz`，提交新包和锁文件，再重启开发服务。请部署整个 `dist` 目录，保留公式引擎和字体的动态加载文件。

## 手工回归

1. 打开页面，检查示例方程正常显示。
2. 在正文末尾插入 `x^2+1`，选择 LaTeX 导出，结果应包含 `$x^2+1$`。
3. 双击新公式改为 `x^3+2`，导出只保留新公式，原示例方程仍然存在。
4. 点击 TinyMCE 的 Undo / Redo，确认公式回退和恢复；打开公式弹窗后取消，应保持正文不变。
5. 返回形式选择 IMG，插入公式后应生成含 `data-latex` 的 PNG 图片；双击修改仍为图片。HTML 导出保留图片数据，LaTeX 导出转换成源码。

构建会提示较大的 chunk：TinyMCE、MathLive、MathJax 为本地完整依赖，其中公式引擎和字体按需加载。
