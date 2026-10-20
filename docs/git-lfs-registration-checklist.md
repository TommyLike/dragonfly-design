# Git LFS 注册清单

> 整套机制在 git 上需要注册的"函数"共 **4 组 + 1 个触发配置**，全部由 `git lfs install` 一次性完成。
> 来源：上游官方 git-lfs 客户端（已对照源码验证：`commands/command_env.go`、`t/t-install.sh`），**非本方案服务端代码**——服务端（gitea fork）的 multipart 扩展不注册到 git。

## 注册清单

| # | 注册物 | 注册位置 | 具体内容 | 作用 |
|---|---|---|---|---|
| 1 | **filter 过滤器对**（两个方向） | `git config filter.lfs.*` | `clean = git-lfs clean -- %f`<br>`smudge = git-lfs smudge -- %f`<br>`process = git-lfs filter-process`（性能优化，长驻进程）<br>`required = true` | 指针 ↔ 真实内容的转换（add 时 clean、checkout 时 smudge） |
| 2 | **pre-push 钩子函数** | `.git/hooks/pre-push` | `git-lfs pre-push "$@"` | push 时收集指针、上传实体、失败中止 push |
| 3 | **diff 驱动函数** | `git config diff.lfs.*` | `textconv = git-lfs textconv`<br>`cachetextconv = true` | `git diff` 时二进制转可读文本再比较 |
| 4 | **merge 驱动函数** | `git config merge.lfs.*` | `driver = git-lfs merge -- %O %A %B %L %P`<br>（%O=祖先 %A=当前 %B=对方 %L=冲突标记长度 %P=输出路径） | 合并冲突时不按文本三路合并二进制 |
| 5 | **触发配置**（非函数，但缺一不可） | 仓库 `.gitattributes` | `*.bin filter=lfs diff=lfs merge=lfs -text` | 决定哪些文件命中上述 4 组注册 |

## 注册后的完整 git 配置形态

```ini
[filter "lfs"]     # 过滤器对：指针 ↔ 真实内容的转换
    clean = git-lfs clean -- %f
    smudge = git-lfs smudge -- %f
    process = git-lfs filter-process
    required = true
[diff "lfs"]       # diff 驱动：二进制内容先转可读文本再比较
    textconv = git-lfs textconv
    cachetextconv = true
[merge "lfs"]      # merge 驱动：二进制冲突不按文本三路合并
    name = Git LFS merge
    driver = git-lfs merge -- %O %A %B %L %P
```

```
# .git/hooks/pre-push
git-lfs pre-push "$@"
```

## 关键细节

1. **`git lfs install --local` 会把 1/3/4 写进仓库的 `.git/config`**——配置随仓库分发，clone 的人自动获得（这正是 Pro Git 提醒的"driver 不会随 `.gitattributes` 分发"问题的解法）；全局 install 则写 `~/.gitconfig` + 全局 hooks 路径；
2. **唯一的钩子是 pre-push**：现代 git-lfs 只注册这一个钩子（clean/smudge 过滤器已覆盖其他时机）；
3. **服务端配套命令**（不在 git 配置里注册，但协议需要）：`git-lfs-authenticate`——ssh 远程下客户端通过 `ssh git@host git-lfs-authenticate <repo> <op>` 换取认证 header；
4. **本方案服务端的补充**（不在 git 注册范围内）：
   - Gitea 侧：`access_token` 认证方案 + multipart 协议扩展（`/objects/batch` 分流 + `/multipart-verify` 路由）；
   - 集群内加速接入（git config，非函数注册）：`http.proxy` / `url.insteadOf` / `lfs.transfer.enablehrefrewrite`。

## 一句话总结

**git 上注册的就 4 个函数（clean/smudge/textconv/merge driver）+ 1 个钩子（pre-push），外加 `.gitattributes` 一行触发条件；其余全部发生在服务端和云平台。**

## 关联文档

- [Git Client 侧深度解析：从 .gitattributes 到后端 API](git-client-deep-dive.md)
- [分片上传深度解析：从 git push 到华为云 OBS](multipart-upload-deep-dive.md)
- [模型存储与下载加速方案设计](model-storage-acceleration-design.md)
