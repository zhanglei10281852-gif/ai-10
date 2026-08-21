# Bug Reproduction

## 包的性质

当前 test_model_fix 保存的是被测模型修复后的结果源码，不是初始含 Bug 源码。要复现原始缺陷，必须检出下面固定的 parent SHA；不要在当前修复结果源码上期待重新出现修复前失败。生成系统使用的可信验证补丁和完整验证日志仅在本地留存，不提交到结果分支。

## 问题现象

ML 工程师和数据工程师登录后拿到两个不同 token，它们长度相同。先认证 ML token，再认证数据工程师 token，第二次返回的却还是 ML 身份。这次先诊断，不要修改代码；生产代码、测试和配置保持不变。请检查缓存命中条件、数据库查询是否被绕过以及最终 principal，给出跨用户串身份的完整原因。

## 含 Bug 版本

- 仓库：zhanglei10281852-gif/ai-10
- 仓库地址：https://github.com/zhanglei10281852-gif/ai-10.git
- parent SHA：0aba8b71d8785e5930383d067641d1ec0d42f46f

## 复现步骤

```bash
git clone -- https://github.com/zhanglei10281852-gif/ai-10.git bug-repro
cd bug-repro
git checkout --detach 0aba8b71d8785e5930383d067641d1ec0d42f46f
go test ./internal/service -run ^TestAuthenticationCacheIsScopedByToken$ -count=1
```

## 双架构完整错误信息

### linux/amd64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/service -run ^TestAuthenticationCacheIsScopedByToken$ -count=1
--- FAIL: TestAuthenticationCacheIsScopedByToken (0.61s)
    annotation_core_behavior_test.go:254: data principal = {UserID:usr_476f45cc0e17ff55dce53385 Email:ops@example.test DisplayName:Ops Role:ml_engineer SessionID:ses_e1a3ea4e21c03bc12ed4c65a}
FAIL
FAIL	github.com/zhanglei10281852-gif/ai/internal/service	0.611s
FAIL

```

stderr：

```text
(empty)
```

### linux/arm64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/service -run ^TestAuthenticationCacheIsScopedByToken$ -count=1
--- FAIL: TestAuthenticationCacheIsScopedByToken (1.92s)
    annotation_core_behavior_test.go:254: data principal = {UserID:usr_33b768f4822413e6fa001103 Email:ops@example.test DisplayName:Ops Role:ml_engineer SessionID:ses_b184997ffb55100e705c5717}
FAIL
FAIL	github.com/zhanglei10281852-gif/ai/internal/service	2.155s
FAIL

```

stderr：

```text
(empty)
```

## 通过条件

诊断必须定位 internal/service/auth.go 的 AuthService principalCache，证明 Authenticate 以 len(token) 作为 sync.Map 缓存键，固定长度的不同 token 因而命中同一缓存项并绕过数据库查询，最终返回首个 token 的 principal；证据需覆盖两次认证、缓存命中和身份角色污染。定向复现完成且仓库保持零改动。
