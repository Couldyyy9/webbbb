# 部署到 GitHub Pages（改进版：在线做题 + 错题本）

静态产物已经导出好了：**`out/` 里有 `index.html`（61 KB）+ 1560 个文件，共 682 MB**，`out/.nojekyll`、`out/CNAME`、`out/404.html` 都在。
但要上 GitHub，**不要把 `out/` 推上去**——仓库里的 `deploy.yml` 会在 GitHub 的机器上自动 `npm run build` 再发布，你只需要推源码。

## 一、先选一个线上地址

| 方案 | 线上地址 | 需要做的设置 | 备注 |
| --- | --- | --- | --- |
| **A. 自定义域名（推荐）** | `https://www.cettong.cn` | 什么都不用设 | 站点原本就是这么发布的：`public/CNAME` 里就是 `www.cettong.cn`；DNS 把 `www` 用 CNAME 指到 `<用户名>.github.io` |
| B. GitHub 自带地址（项目站） | `https://<用户名>.github.io/<仓库名>/` | 仓库 → Settings → Secrets and variables → **Actions → Variables** 加 `PAGES_BASE_PATH = /<仓库名>` | 不设的话页面会**样式全丢 + 题库 404**，因为静态产物里是 `/xxx` 这种根路径 |
| C. 用户站 | `https://<用户名>.github.io/` | 什么都不用设 | 仓库名必须正好是 `<用户名>.github.io` |

> 本次已经给代码加了 `PAGES_BASE_PATH` 支持（`next.config.ts` + `src/lib/base-path.ts`）：题库 JSON 的 fetch、真题 PDF/MP3 的地址都会跟着加前缀，`next/link` 的页面跳转由 Next 自动处理。**本地开发什么都不用设**，仍然是 `http://localhost:3000`。
>
> **选 B / C（不要自定义域名）时多做两步**：删掉 `.github/workflows/deploy.yml` 里 `cname: www.cettong.cn` 这一行，并删掉 `public/CNAME`。否则发布出来的 `gh-pages` 分支里会带着 `CNAME`，Pages 会一直认为你的站要绑定 `www.cettong.cn`，在 Settings → Pages 里显示域名没验证/证书报错。
> 选 A（用自己的域名）时，把 `deploy.yml` 的 `cname:`、`public/CNAME` 两处改成你自己的域名即可。

## 二、要推上去的文件（只有 ~1 MB）

原仓库里那 667 MB 的 `public/library`（241 个 PDF/MP3）本次**一个字节都没改**，所以：

- 如果你**基于原仓库**（fork 或直接用自己的仓库）→ 只需补上改动的文件，媒体文件不用重新上传。
- 如果你**新建空仓库** → 667 MB 媒体也得推上去（GitHub 单个文件上限 100 MB，最大文件 13 MB，没问题；但仓库会很大、上传很慢）。

改动文件的完整清单和一份可直接覆盖的副本已经生成好了：

```
C:\Users\15963\Desktop\deepseek h\push-to-github\
├── FILELIST.txt          # 本次新增/修改的每个文件（相对仓库根目录）
└── <按仓库目录结构摆放的所有改动文件>
```

覆盖方式（PowerShell，`$repo` 改成你 clone 下来的仓库目录）：

```powershell
$repo = "C:\Users\15963\Desktop\deepseek h\cet-website"   # git clone 出来的目录
Copy-Item -Path "C:\Users\15963\Desktop\deepseek h\push-to-github\*" -Destination $repo -Recurse -Force
cd $repo
git add -A
git commit -m "feat: 在线做题 + 错题本（24 套 / 1119 题）"
git push
```

> 注意：`push-to-github\` 里**不含** `node_modules`、`.next`、`out`、`.banktest`（这些都不该入库，`.gitignore` 已经忽略）。

## 三、安装 git（本机还没装）

```powershell
winget install --id Git.Git -e --source winget
# 装完重开一个终端，git --version 应该能输出版本号
```

不想装 git 也行：GitHub 网页版支持**拖拽文件夹上传**，把 `push-to-github\` 里的内容按目录结构拖进仓库即可（先删掉网页上的同名旧文件，或让 GitHub 显示 diff 后提交）。

## 四、首次部署的一次性设置

1. 仓库 → **Settings → Pages** → Source 选 **Deploy from a branch** → Branch 选 **`gh-pages`** / **`/ (root)`** → Save。
   （`deploy.yml` 用 `peaceiris/actions-gh-pages` 把 `out/` 推到 `gh-pages` 分支，所以这里要选它，不是 `main`。）
2. 仓库 → **Settings → Pages → Custom domain** 填 `www.cettong.cn` → Save → 勾上 **Enforce HTTPS**。
3. DNS（在你买域名的服务商那里）：`www` 记录 CNAME → `<用户名>.github.io`。
4. 仓库 → **Actions** 标签页，确认 `Deploy to GitHub Pages` 跑成功（约 2–4 分钟，第一次要装依赖）。

之后每次 `git push` 到 `main`/`master`，Actions 都会自动重新构建并发布。

## 五、部署完怎么自检

| 访问 | 期望 |
| --- | --- |
| `/` | 首页出现「在线做题」和「错题本」入口 |
| `/practice` | 24 套题库列表（四级 12 + 六级 12） |
| `/practice/cet4/2024_12_1` | 55 道题、答题卡、听力播放器、0 秒开始计时 |
| `/mistakes` | 空状态「还没有错题记录」；做一套题后自动出现错题 |
| `/bank/manifest.json` | JSON，`"totalQuestions":1119` |
| `/library/cet4/2024_12_1/test.pdf` | 能直接看 PDF（带 Range 206 就是正常） |

## 六、常见坑

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 页面打开是纯文字、样式全丢 | 少了 `.nojekyll`，Jekyll 把 `/_next` 吃掉了 | `public/.nojekyll` 已存在，确认它被推上去了（peaceiris 也会自动加） |
| 本地好好的，线上样式/题库 404 | 用了 github.io 子路径但没设 `PAGES_BASE_PATH` | 按方案 B 加仓库变量后重新跑 Actions |
| 答题页提示「题库加载失败」 | `public/bank/*.json` 没推上去 | 确认仓库里有 `public/bank/` 25 个 JSON |
| Actions 在 `npm ci` 失败 | `package-lock.json` 里写的是国内镜像地址 | workflow 里已有 `sed` 把 `registry.npmmirror.com` 换回官方源 |
| Actions 构建报 Node 版本 | 旧 workflow 用的是 Node 20 | 已改为 Node 22 |
| 想发布到自己的域名 | | 改 `deploy.yml` 里的 `cname:` 和 `public/CNAME` |

## 七、备选：不用 Actions，直接把导出产物推上 gh-pages

```powershell
cd "C:\Users\15963\Desktop\deepseek h\cet做题"
$env:NODE_ENV='production'; npm run build          # 生成 out\
cd out
git init; git checkout -b gh-pages
git add -A; git commit -m "deploy: static export"
git remote add origin https://github.com/<用户名>/<仓库名>.git
git push -f origin gh-pages
```

这样推的是**完整的 682 MB 静态产物**（含 PDF/MP3），适合"本地已经构建好、不想让云端跑构建"的情况。改动源码后要重新构建再推一次。
