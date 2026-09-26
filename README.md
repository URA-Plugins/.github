# URA plugin workflows

`plugin.yml` 构建单个插件、校验 ZIP、运行该仓库 `tests/` 下的 smoke 项目。

## 本地验证

Windows 安装 Git、PowerShell 7.6、Node.js 24、.NET SDK 10.0.303 和 [act](https://nektosact.com/installation/index.html)。在插件仓根目录执行：

```powershell
act workflow_dispatch --artifact-server-path "$env:TEMP/ura-act-artifacts"
```

仓库 `.actrc` 将 `windows-latest` 映射到 Windows 本机执行器。workflow 使用临时工作目录，构建关闭本机插件部署；`actions/upload-artifact` 将产物交给 act 的本地 artifact server。

修改共用 workflow 时，增加本地仓库映射，将 `<commit-sha>` 替换为调用仓固定的完整 40 位提交 SHA，路径填写本仓库的绝对路径：

```powershell
act workflow_dispatch --artifact-server-path "$env:TEMP/ura-act-artifacts" --local-repository 'URA-Plugins/.github@<commit-sha>=C:/src/ura-workflows'
```

本地和 GitHub 调用同一份 workflow，执行相同的构建、产物校验和测试步骤。[act 的限制](https://nektosact.com/not_supported.html) 包括 permissions、concurrency 等 GitHub 服务端语义；本地运行不能证明远端权限和 Release 发布成功。

## GitHub Release

22 个插件仓通过 `URA-Plugins/.github/.github/workflows/plugin.yml@<commit-sha>` 调用共用 workflow，每批发布固定到经过验证的完整 40 位提交 SHA；`v1` 标签保留。普通 push、pull request 和手动运行执行验证；推送 `v` 开头的 tag 才执行发布 job。

1. 在插件项目中更新数字版本、变更记录，完成验证。
2. 推送对应 tag，例如 `v1.2.3` 或 `v1.2.3-preview.1`；tag 的数字部分必须与 manifest `Version` 相等。
3. workflow 下载验证 job 的同一份 ZIP，创建 draft Release、附上唯一插件 ZIP，再发布。带 `-` 后缀的 tag 标记为 prerelease；其它 tag 标记为 stable。

插件通过 `Version="*"` 引用最新稳定版 Host NuGet 包。workflow 构建前刷新依赖解析，按实际包内的 repository commit 检出测试用 Host；编译或测试失败直接终止。不同时间运行可能解析到不同包版本，已发布的 ZIP 保持原内容。

ZIP 文件名、主 DLL、manifest `InternalName` 必须一致，大小上限为 64 MiB；manifest 由 Host NuGet 构建契约生成。发布使用调用仓库的 `GITHUB_TOKEN`，需要 `contents: write`。已有同名 Release 会明确失败，修正版本后使用新 tag。

URACloud 从 GitHub Release 读取插件包；首次接入时在插件中心选择仓库并同步。
