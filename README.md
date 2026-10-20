# Dragonfly 设计文档

模型平台"模型存储与下载加速"方案设计文档及配套图。

## 内容

| 路径 | 说明 |
|---|---|
| [docs/model-storage-acceleration-design.md](docs/model-storage-acceleration-design.md) | 设计文档：整体挑战、业界参考、10 条关键设计决策（ADR）、运行时视图、部署模型、量化收益 |
| [docs/multipart-upload-deep-dive.md](docs/multipart-upload-deep-dive.md) | 分片上传深度解析：从 git push 到华为云 OBS 的全链路组件与关键考量点 |
| [docs/git-client-deep-dive.md](docs/git-client-deep-dive.md) | Git Client 侧深度解析：从 .gitattributes 到后端 API（filter driver / hook 协议 / API 交互） |
| [docs/server-side-key-designs.md](docs/server-side-key-designs.md) | 服务端关键设计解析：任务归一化 / 续传状态外置 / etag 透传 / verify 幂等 |
| [docs/git-lfs-registration-checklist.md](docs/git-lfs-registration-checklist.md) | Git LFS 注册清单：4 组函数 + pre-push 钩子 + .gitattributes 触发配置 |
| [images/whimsical/](images/whimsical/) | Whimsical 图的 PNG 渲染截图（文档内嵌使用） |
| [diagrams/](diagrams/) | 架构图的可编辑 spec（`.json`）与交互式 HTML（archify 格式，备选版本） |

## Whimsical 在线图（可编辑）

| 图 | 链接 |
|---|---|
| 01 总体架构 | <https://whimsical.com/ALQXAJfBYgokJCd3ZbJTeZ> |
| 02 multipart 上传时序 | <https://whimsical.com/ACw6GL1ggeAvGhdeoQCrMb> |
| 03 集群内 P2P 下载时序 | <https://whimsical.com/DGc7bHNenoM4hjV41ihHYK> |
| 04 预热时序 | <https://whimsical.com/7QRsY6z1R7smVPPVowztvz> |
| 05 生产部署拓扑 | <https://whimsical.com/2GtouQzSR6ZMLAiJVc7LLY> |
| 06 分片上传全链路（git push → OBS） | <https://whimsical.com/98uDharcatoKyG3ELkrZAw> |
| 07 四个自动的机制归属 | <https://whimsical.com/qH4xLdMwtYQM8MJRqenHC> |
| 08 上传 API 交互时序 | <https://whimsical.com/mmMHWpYZ3VunAxsh7iABi> |

## 关联仓库

- [openmerlin/gitea](https://github.com/openmerlin/gitea) — LFS multipart / CDN 实现
- [TommyLike/dragonfly-config](https://github.com/TommyLike/dragonfly-config) — Dragonfly 集群配置
- [openmerlin/dragonfly-deploy](https://github.com/openmerlin/dragonfly-deploy) — Dragonfly 生产部署脚本
