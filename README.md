# Iconify API

该存储库包含 Iconify API 脚本。它是一个使用 Node.js 编写的 HTTP 服务器，具有以下功能：

- 提供图标数据，供按需加载图标数据的图标组件使用（而不是捆绑成千上万个图标）。
- 生成 SVG，您可以在 HTML 或样式表中链接到它。
- 提供托管图标的搜索引擎，可供图标选择器使用。

## Docker

要构建 Docker 镜像，请运行 `./docker.sh`。

如果您想自定义配置，请先 fork 该存储库，自定义源代码，然后构建 Docker 镜像并部署 API。

要运行 Docker 镜像，请运行 `docker run -d -p 3000:3000 iconify/api`（将第一个 3000 更改为您要运行 API 的端口）。

与 Docker 镜像一起使用的 NPM 命令：

- `npm run docker:build` - 构建 Docker 镜像。
- `npm run docker:start` - 在端口 3000 上启动 Docker 容器。
- `npm run docker:stop` - 停止所有 Iconify API Docker 容器。
- `npm run docker:cleanup` - 删除所有未使用的 Iconify API Docker 容器。

由于 Docker 的限制，没有命令来删除未使用的镜像。您需要手动从 Docker Desktop 或命令行中进行删除。

## 如何使用

首先，您需要安装 NPM 依赖项并运行构建脚本：

```
npm install
npm run build
```

然后可以启动服务器：

```
npm run start
```

默认情况下，服务器将：

- 自动从 [`@iconify/json`](https://github.com/iconify/icon-sets) 加载最新的图标。
- 从 `icons` 目录加载自定义图标集。
- 在端口 3000 上提供数据。

您可以自定义 API 的行为：

- 提供自定义图标集，从各种来源加载。
- 在不同的端口上运行。
- 如果您不需要搜索引擎，可以禁用它以减少内存使用。

## 端口和 HTTPS

建议不要在端口 80 上运行 API。服务器可以处理几乎任何端口，但仍然不如专用解决方案（如 nginx）好。

在不为外界暴露的不常用端口上运行 API，使用防火墙规则隐藏，使用 nginx 作为反向代理。

不支持 HTTPS。这是一个非常占用资源的过程，最好由专用解决方案（如 nginx）处理。使用 nginx 作为 HTTP 和 HTTPS 服务器，在隐藏端口（如默认端口 3000）上转发查询到 API HTTP 服务器。

## 配置

有几种方式可以更改配置：

- 编辑 `src/config/` 目录中的文件，然后重新构建脚本。这对于一些高级选项（例如使用具有自定义图标的 API）是必需的。
- 使用环境变量，例如 `PORT=3100 npm run start`。
- 使用 `.env` 文件存储环境变量。

### 环境变量选项

可以使用环境变量更改的选项及其默认值（您可以在 `src/config/app.ts` 中找到所有选项）：

- `HOST=0.0.0.0`：HTTP 服务器监听的 IP 地址或主机名。
- `PORT=3000`：HTTP 服务器监听的端口。
- `ICONIFY_SOURCE=full`：Iconify 图标集的来源。设置为 `full` 使用 `@iconify/json` 包，`split` 使用 `@iconify-json/*` 包，`none` 仅使用自定义图标集。
- `REDIRECT_INDEX=https://iconify.design/`：`/` 路由的重定向。API 不提供任何页面，因此索引页面重定向到主网站。
- `STATUS_REGION=`：要添加到 `/version` 路由响应的自定义文本。Iconify API 在服务器网络上运行，访问者被路由到最近的服务器。用于告诉用户连接到的服务器。
- `ENABLE_ICON_LISTS=true`：启用列出图标集的 `/collections` 路由和 `/collection?prefix=whatever` 路由以获取图标列表。用于图标选择器。如果仅使用 API 提供图标数据，则禁用它。
- `ENABLE_SEARCH_ENGINE=true`：启用 `/search` 路由。需要启用 `ENABLE_ICON_LISTS`。
- `ALLOW_FILTER_ICONS_BY_STYLE=true`：允许根据填充或描边搜索图标，例如在搜索查询中添加 `style=fill`。此功能会使用一些内存，因此可以禁用它。需要启用 `ENABLE_SEARCH_ENGINE`。

### 内存管理

默认情况下，API 将使用内存管理功能。它仅在内存中存储最近使用的图标，从而减少内存使用。

如果您的 API 流量很大（每分钟超过 1,000 个请求），最好不要使用内存管理。在如此高的查询数量下，磁盘读/写操作可能会导致性能下降。如果所有内容都在内存中并可以立即访问，API 可以在基本 VPS 上轻松处理多达 10 倍的流量。

有关完整 API 文档中的内存管理，请参阅[内存管理](https://docs.iconify.design/api/hosting-js/config.html)。

### 更新图标

服务器启动时会自动更新图标。

除此之外，API 还可以在不重新启动服务器的情况下更新图标集。

要启用自动更新，您必须设置 `APP_UPDATE_SECRET` 环境变量。如果没有设置，更新将无法工作。

- `ALLOW_UPDATE=true`：启用 `/update` 路由。
- `UPDATE_REQUIRED_PARAM=secret`：来自密钥/值对的密钥。不能为空。
- `APP_UPDATE_SECRET=`：来自密钥/值对的值。不能为空。
- `UPDATE_THROTTLE=60`：运行更新之前等待的秒数。

要触发图标集更新，请打开 `/update?foo=bar`，其中 `foo` 是 `UPDATE_REQUIRED_PARAM` 的值，`bar` 是 `APP_UPDATE_SECRET` 的值。

更新不会立即触发，而是在 `UPDATE_THROTTLE` 秒后运行。这样做是为了防止在连续多次触发更新（例如 GitHub 钩子）时进行多次检查。

如果在更新过程已经运行时（即已检查源以进行更新，但下载仍在进行中），则在当前运行的更新结束后将运行另一个更新检查。

`/update` 路由的响应始终相同，无论结果如何。这样做是为了防止尝试猜测密钥/值对，甚至查看路由是否启用。要查看实际结果，您需要检查控制台。成功的请求和更新过程将被记录。

### HTTP 头

默认情况下，服务器发送以下 HTTP 头：

- 各种 CORS 头，允许从任何地方访问。
- 缓存头，将响应缓存 604800 秒（7 天）。

要更改头，请编辑 `src/config/app.ts` 中的 `httpHeaders` 变量，然后重新构建脚本。

## Node vs PHP

先前版本的 API 也作为 PHP 脚本提供。这已经停止使用。Node 应用程序执行速度更快，可以处理每秒数千个查询，并且使用的内存更少。

## 完整文档

此文件是基本的。

完整的文档可在[Iconify 文档网站](https://docs.iconify.design/api/)上找到。

## 赞助商

<p align="center">
  <a href="https://github.com/sponsors/cyberalien">
    <img src='https://cyberalien.github.io/static/sponsors.svg'/>
  </a>
</p>

## 许可证

Iconify API 使用 MIT 许可证。

`SPDX-License-Identifier: MIT`

此许可证不适用于托管在 API 上的图标和 API 生成的文件。您可以以任何许可证托管图标，没有任何限制。请遵守公共道德，不要托管盗版商业图标集（虽然不确定为什么有人会使用商业图标集，因为有许多优秀的开源图标集可用）。
