# 服务端关键设计解析

> **口径**：本文聚焦**我们补的服务端设计**（Gitea fork 的实现 + Dragonfly 适配配置），不展开上游组件内部机制（git-lfs 客户端机制见 [git-client-deep-dive.md](git-client-deep-dive.md)，Dragonfly 内部调度不在本文范围）。
> **关联实现**：[openmerlin/gitea@dc8744072](https://github.com/openmerlin/gitea/commit/dc8744072)（Support multipart）、[@97ef16488](https://github.com/openmerlin/gitea/commit/97ef16488)（NoSuchUpload 幂等）、openmerlin/dragonfly-deploy（代理配置）

## 1. 关键度梯队

按"做错了会怎样"衡量：

| 梯队 | 设计 | 错了会怎样 |
|---|---|---|
| 第一梯队 | ① 签名 URL 任务归一化 | **静默失效**：P2P 缓存命中率恒为 0，系统"看起来正常"（能下载，只是都回源），只有成本账单会暴露 |
| 第一梯队 | ② 断点续传状态外置 | 客户端零修改做不到，或服务端被迫引入分布式上传状态（状态存储、过期、多实例一致性问题爆炸） |
| 第二梯队 | ③ etag 透传（协议定制） | 合并直接失败 → 已传分片全废、全量重传；报错是不直观的 `NoSuchUpload` |
| 第二梯队 | ④ CommitAndVerify 幂等 | 网络抖动下重试产生假失败（数据其实好了，用户看到 push 失败）或脏数据 |

---

## 2. ① 签名 URL 任务归一化

**问题**：dfdaemon 代理按 **URL 生成任务 ID**——同一 URL = 同一任务 = 缓存可复用。而 LFS 下载 URL 是动态签名的（`Expires`/`Signature` 每次生成都变），CDN 化后还可能多一层 `auth_key`。结果：同一模型每次下载都是"新任务"，缓存永远 miss，且**无任何报错信号**。

**设计**（`seed-peer.yaml` / `dfget.yaml`）：

```yaml
proxy:
  proxies:
    - regx: /lfs/\w{2}/\w{2}/\w{60}\?      # 1) 只代理 LFS 对象流量
  defaultFilter: 'response-content-disposition&AWSAccessKeyId&Signature&Expires'
                                            # 2) 任务 ID 剥离签名参数
```

- **正则过滤（一层收口）**：`/lfs/\w{2}/\w{2}/\w{60}\?` 精确匹配 Gitea 的 LFS 对象路径格式（`lfs/{oid前2位}/{oid第3-4位}/{oid其余60位}` + 必有查询参数）。命中才走 P2P 代理，其他流量完全不进缓存体系；
- **参数剥离（二层归一）**：`defaultFilter` 声明"生成任务 ID 时忽略这些查询参数"——`AWSAccessKeyId&Signature&Expires` 是 OBS 签名三件套（v2 签名形式），`response-content-disposition` 每次生成也会变。剥离后同一 oid 的任何两次签名 URL 映射到**同一个任务 ID**；
- **逃生门**：代理支持 `X-Dragonfly-Filter` 请求头按请求覆盖默认剥离策略。

**两个要点**：

1. 正则与 defaultFilter 是**两层独立防线**：正则决定"什么进缓存"，filter 决定"什么算同一个东西"——缺任何一个都失效，但失效方式不同（前者缓存了不该缓存的，后者该命中的不命中）；
2. ⚠️ **需要确认的联动点**：若 CDN 的 `auth_key`（方式A，含随机数）进入 URL，它同样随每次生成变化，**必须同步加入 defaultFilter**。当前配置快照未包含 `auth_key`（main 分支代码中亦未见其生成逻辑），说明该机制可能尚未上线或以其他方式配置——代码 review 时应显式确认一次。

---

## 3. ② 断点续传状态外置

**约束**：Gitea 多实例无状态（不能自己记进度）× 客户端零修改 × batch 协议幂等。**解：把"传到哪了"存在 OBS 的 multipart 任务状态机里，每次协商都查。**

**设计**（`modules/storage/hwcloud.go` 的 `GenerateMultipartParts`）：

```
GenerateMultipartParts(objectKey, size):
  1. ListMultipartUploads(Prefix=objectKey)     # 查该对象有几个未完成任务
       >1 个 → 全部 AbortMultipartUpload         # 脏状态（协商半途失败留下的），重建
       =1 个 → ListParts(uploadId)               # 拿到已传分片清单（PartNumber→ETag/Size）
       =0 个 → 跳过
  2. uploadID 为空 → InitiateMultipartUpload
  3. 按 20MB/片 循环生成分片：
       partSize = min(20MB, size - offset)
       已传 且 Size == partSize → 只返回 Etag（不带上传 URL）  # 客户端跳过 = 续传
       否则 → CreateSignedUrl(PUT, ?partNumber=N&uploadId=...)  # 签名 URL，默认 24h
  4. 返回 parts[] + verify + abort
```

**三个"为什么"**：

- **>1 个任务全 abort 再重建**：多任务是状态分裂信号（上一次协商失败后客户端又重试过）。保留任何一个，客户端的分片认知都可能与 OBS 侧不一致；重建（重传部分分片）的成本远低于"合并出损坏对象"；
- **已传分片必须 Size 匹配才复用**：TCP 中断时最后一个分片的 PUT 可能未写完，OBS 任务里留下"已登记但不完整"的分片。只按 partNumber 判断会把它带进合并——必须 size 完全一致才认；
- **签名 URL 给 24h**（`default_expire = 86400`）：GB 级上传 + 重试窗口跨小时级，30 分钟（早期值）会"文件没传完 URL 先过期"。过期无害——重试一次 batch 协商（本函数幂等）即可拿到新 URL。

---

## 4. ③ etag 透传（协议定制）

**约束**：华为云 OBS 的 `CompleteMultipartUpload` **必须携带每段 ETag 做合并前校验**。ETag 只出现在分片 PUT 的响应头里——服务端看不到，只有客户端能看到。社区 multipart 提议无此需求，是厂商约束逼出的协议定制。

**设计**（`modules/structs/multipart.go` + `services/lfs/server.go`）：

1. **batch 响应声明聚合结构**：

```json
"verify": {
  "href": ".../info/lfs/multipart-verify?oid=..&size=..",
  "params": { "uploadId": "..." },
  "aggregation_params": {
    "key": "part_ids", "type": "array",
    "item": { "type": "object", "attributes": ["part_id", "etag"] }
  }
}
```

2. **客户端捕获并聚合**：每个分片 PUT 完成时从响应头取 `ETag`，与 `part_id` 配对收集，全部完成后调用 verify：

```json
POST /info/lfs/multipart-verify?oid=..&size=..
{ "upload_id": "...",
  "part_ids": [ {"etag": "abc...", "part_id": 1}, {"etag": "def...", "part_id": 2} ] }
```

3. **服务端消费**（`HWCloudStorage.CommitUpload`）：解析 `part_ids` → 组装 `obs.Part{ETag, PartNumber}` → `CompleteMultipartUpload`。数据结构对应：`MultipartPartID{Etag, Index}` → `MultiPartCommitUpload{UploadID, PartIDs}`。

**采纳决策**：Giftless 为 Azure 的 commit 需求新增了独立的 `commit` action；官方 multipart 提议建议**把 commit 语义合入 `verify`**。我们采纳官方建议——verify 即 commit，少一个 action、少一次客户端状态机转换，代价是 verify 语义比社区规范略重（合并+校验一体）。

---

## 5. ④ CommitAndVerify 幂等（重试安全）

**问题**：网络抖动下客户端会**无脑重试 verify**。若 verify 非幂等，第一次已成功、第二次重试报错 → 用户看到 push 失败（假失败）→ 全量重传或人工介入。

**设计**（`MultiPartVerifyHandler` → `CommitAndVerify`，含 commit `97ef16488`）：

```
MultiPartVerifyHandler:
  1. 解析 oid/size（query）+ body（upload_id + part_ids）
  2. 仓库鉴权
  3. contentStore.Exists(p)?
       已存在 → LFSObjectAccessible 校验 → 直接 200       ← 幂等兜底①
  4. CommitAndVerify(p, body):
       a. CommitUpload → CompleteMultipartUpload
          （NoSuchUpload 错误被忽略）                       ← 幂等兜底②
       b. Stat(p)：不存在 / size 不匹配 → false → 404     ← 正确性兜底
  5. NewLFSMetaObject 入库（对象对仓库可见的开关）
```

**幂等三件套**：

| 兜底 | 防什么 |
|---|---|
| ① 对象已存在 → 直接 200 | 重复 verify：第一次已成功，重试不再走合并路径 |
| ② `NoSuchUpload` 忽略 | 任务已被合并/被脏任务清理 abort 后，重试的 Complete 不报错 |
| ③ 合并后 `Stat` 校验 size | 分片缺失/错序——合并前校验漏掉的错误，以"字节数对不上"暴露，宁失败不静默 |

**设计理念**：把"客户端重试"从错误路径变成正常路径——协议允许客户端在任意环节断开后无条件重试，服务端保证收敛到同一终态。

---

## 6. 四者关系与关联文档

- ②③④ 构成"上传的正确性与可用性"：状态外置 → etag 保证合并成功 → 幂等保证重试安全；
- ① 独立守护"下载加速的真实有效性"，且与 CDN 鉴权（主文档 ADR-4）交织。

- [模型存储与下载加速方案设计](model-storage-acceleration-design.md)（ADR-2/3/4/9）
- [分片上传深度解析：从 git push 到华为云 OBS](multipart-upload-deep-dive.md)
- [Git Client 侧深度解析：从 .gitattributes 到后端 API](git-client-deep-dive.md)
- [Git LFS 注册清单](git-lfs-registration-checklist.md)
