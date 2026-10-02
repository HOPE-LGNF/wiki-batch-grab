# wiki-batch-grab

批量从 MediaWiki 导出页面源码为本地文件。

**为什么需要它**：[Wikitext](https://marketplace.visualstudio.com/items?itemName=RoweWilsonFrederiskHolme.wikitext) 只有单页的「Pull page to edit」，没有批量导出；而 MediaWiki 的 `action=query` 一次可以问几十个标题，批量抓取非常合适。

**和 Wikitext 的关系**：导出的文件默认带 `PAGE_INFO` 头，格式与 Wikitext 写入的**逐字节一致**。所以抓下来、改完，可以直接用 Wikitext 的「Post your edit to the website」推回去——它从头部读目标页面与冲突基准，并在上传前把整块剥掉。

## 要求

Node 18+（用到内置 `fetch`）。**零第三方依赖**，整个工具就是一个文件，可以直接拷到任何地方运行。

## 用法

```bash
# 先看要抓哪些（只解析标题，不发内容请求）
node batch-grab.mjs --site <你的站点> --prefix "模块:建筑/" --dry-run

# 真正导出
node batch-grab.mjs --site <你的站点> --prefix "模块:建筑/" --out ./wiki-export

# 混合来源，并放慢节奏
node batch-grab.mjs --site <你的站点> --category "分类:角色" --file 额外标题.txt \
  --out ./wiki-export --delay 500

# 被 Cloudflare 质询时，换成浏览器的 UA（必要时再补请求头）
node batch-grab.mjs --site <你的站点> --prefix "模块:" \
  --ua "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36" \
  --header "Accept-Language: zh-CN,zh;q=0.9"
```

> **必须跑在能访问该站点的网络里。** 它是个普通的 Node 脚本，不在编辑器里运行、也不走任何扩展宿主。站点在 Cloudflare 后面时，数据中心 IP 或非常规客户端容易被质询——此时在自己机器上跑才有意义。

## 参数

| 参数 | 作用 |
|---|---|
| `--site <host>` | 必填，例如 `www.huijiwiki.com` |
| `--scheme <s>` | 默认 `https://` |
| `--api-path <p>` | 默认 `/api.php`。**注意维基百科是 `/w/api.php`，灰机wiki 是 `/api.php`** |
| `--ua <string>` | User-Agent。MediaWiki 要求 UA 能标识工具与联系方式，默认值已符合 |
| `--header "K: V"` | 追加/覆盖任意请求头，可重复 |
| `--pages "A,B,C"` | 直接给标题 |
| `--file <path>` | 从文件读标题，每行一个，`#` 开头忽略 |
| `--prefix <p>` | 按标题前缀列举（含子页面），例如 `模块:建筑/` |
| `--category <c>` | 列举某分类的成员 |
| `--all` | 列举全部页面（慎用） |
| `--out <dir>` | 输出目录，默认 `./wiki-export` |
| `--name-style` | `dash`（默认）把 `:` 与 `/` 换成 `_`，平铺；`raw` 保留 `/`，按子页面分目录 |
| `--no-page-info` | 不写 `PAGE_INFO` 头，只要纯源码 |
| `--overwrite` | 覆盖已存在的文件；默认跳过，便于中断后续跑 |
| `--batch <n>` | 每次请求合并的标题数，默认 20，上限 50（API 限制） |
| `--delay <ms>` | 两次请求之间的等待，默认 300 |
| `--dry-run` | 只列出将要导出的页面，不请求内容、不写文件 |
| `--quiet` | 只打印汇总 |

## 工作方式

1. **枚举**：`list=allpages`（含 `apprefix`）/ `list=categorymembers` / 显式标题，翻页靠响应里的 `continue`，结果取并集去重。
2. **抓正文**：`action=query&prop=revisions&rvprop=content|ids|timestamp&rvslots=main&titles=A|B|C`，`formatversion=2` 拿数组式结构，默认 20 个标题一批。
3. **落盘**：文件名 = 标题按风格换字符 + 内容模型对应扩展名（`wikitext` → `.wikitext`、`Scribunto` → `.lua`、`json` → `.json`、`css` → `.css`、`javascript` → `.js`）；内容是 `PAGE_INFO` 头 + 空行 + 正文。
4. **健壮性**：整批请求失败会逐条重试（坏标题不牵连整批）；页面不存在或未返回正文会单独记账，最后逐条列出并以非零码退出；`--delay` 节流；默认跳过已存在文件，中断后重跑即续传。

请求只设两个头（`User-Agent` 与 `Accept: application/json`），其余交给 Node `fetch` 的默认值；不做任何写操作，读取走匿名即可，不需要账号。

## 输出格式

文件名：`模块:实体/信息框` → `模块_实体_信息框.lua`（`dash` 风格）

```text
<%-- [PAGE_INFO]
    comment = #Please do not remove this struct. It's record contains some important information of edit. This struct will be removed automatically after you push edits.#
    pageTitle = #模块:实体/信息框#
    pageID = #2001#
    revisionID = #15001#
    contentModel = #Scribunto#
    contentFormat = #text/plain#
[END_PAGE_INFO] --%>

local p = {}
```

这段字节是**跨仓库契约的锚点**：本仓库的测试断言「产出的就是这些字节」，[wiki-user-preview](https://github.com/HOPE-LGNF/wiki-user-preview) 的测试断言「能解析这些字节」。任一侧改了格式，自己那侧的测试就会红——契约靠固定样本 + 文档维持，不靠两个仓库共享代码。

## 测试

```bash
node --test
```

12 项断言：文件名映射、`PAGE_INFO` 格式与契约样本、分批合并、整批失败后逐条重试、续跑跳过、`--dry-run` 不发内容请求、list 接口分页续抓、客户端参数编码 / UA / 自定义头 / API 与 HTTP 错误处理。

## 许可

MIT
