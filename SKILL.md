---
name: aliyun-oidc-cert-renew
description: 用 GitHub Actions + 阿里云 RAM OIDC 自动续期 Let's Encrypt 通配证书并分发到 OSS/FC/CDN，仓库零长期 AK。覆盖 OIDC 提供商/角色/权限策略搭建、workflow 手动兑换临时 STS、ACME DNS-01 签发、CAS 上传复用、新子域 DNS 自动发现绑定、60 天 keepalive、状态持久化，以及 IMS/RAM/CAS/oss2 SDK 的全部命名坑。当用户说「自动续期证书」「不想存 AK」「GitHub Actions 操作阿里云」「SSL 证书过期」「新增子域自动上 HTTPS」「免密钥访问阿里云」时使用。
version: 1.0.0
license: MIT
agent_created: true
---

# 阿里云证书自动续期（GitHub Actions + RAM OIDC，免长期 AK）

## 何时用

- 想让**定时任务**自动续期 Let's Encrypt 证书并分发到阿里云（OSS / FC / CDN），但**不想把长期 AK/SK 塞进仓库 Secrets**。
- 想让**新子域加一条 CNAME 就自动上 HTTPS**，不用改代码。
- 更广义：任何「GitHub Actions（或别的 OIDC IdP）免密钥操作阿里云」的场景，本文第一、二节可直接复用。

## 架构

```
GitHub Actions (id-token: write)
   │ 1. 请求 OIDC JWT（显式 audience）
   ▼
RAM OIDC 提供商 (token.actions.githubusercontent.com)
   │ 2. AssumeRoleWithOIDC
   ▼
RAM 角色 → 临时 STS 凭证（1h）
   │ 3. 写入 $GITHUB_ENV
   ▼
cert_manager.py（ACME DNS-01 签发 → CAS 上传 → OSS/FC 分发）
   │ 4. 状态回写
   ▼
OSS 私有桶（account.key / privkey.pem / state.json）
```

**仓库里只需要两个非敏感 Secret**：`ALI_ROLE_ARN`、`ALI_OIDC_ARN`。

## 一、RAM 侧搭建（一次性，用永久 AK 调 API，AK 不落盘）

账号 ID 用 `sts.GetCallerIdentity` 拿，无需用户名。

### 1. OIDC 提供商 —— 属 **IMS API**，不是 `ram20150501`

```bash
pip install alibabacloud_ims20190815
```

- 方法 `create_oidcprovider`（小写 p），模型 `CreateOIDCProviderRequest`
- 字段全部 **snake_case**：`oidc_provider_name`、`issuer_url`、`client_ids`、`fingerprints`、`description`
- `issuer_url` = `https://token.actions.githubusercontent.com`
- `client_ids` = audience，如 `https://github.com/<owner>/<repo>`
- **指纹必须纯 hex、无冒号**：

```bash
openssl s_client -connect token.actions.githubusercontent.com:443 </dev/null 2>/dev/null \
  | openssl x509 -pubkey -noout \
  | openssl rsa -pubin -outform der 2>/dev/null \
  | openssl dgst -sha1        # 取输出里的 hex，去掉冒号
```

### 2. 角色（`alibabacloud_ram20150501`）

```python
ram.create_role(CreateRoleRequest(
    role_name=ROLE, assume_role_policy_document=json.dumps(trust), description=...))
```

信任策略 —— **`oidc:sub` 一定要 `StringLike` + 通配**（原因见「坑 3」）：

```json
{"Statement":[{"Action":"sts:AssumeRole","Effect":"Allow",
  "Principal":{"Federated":"acs:ram::<ACCT>:oidc-provider/github"},
  "Condition":{"StringEquals":{"oidc:iss":"https://token.actions.githubusercontent.com",
                                "oidc:aud":"https://github.com/<owner>/<repo>"},
               "StringLike":{"oidc:sub":"repo:<owner>*/<repo>*:ref:refs/heads/main"}}}],
 "Version":"1"}
```

改信任范围用 `update_role(UpdateRoleRequest(role_name=..., new_assume_role_policy_document=...))`
—— 字段名是 `new_assume_role_policy_document`。

### 3. 权限策略

`create_policy` + `attach_policy_to_role`（snake_case：`policy_name`、`policy_document`、`policy_type`、`role_name`）。

Allow Action **至少**：`alidns:*`、`oss:*`、`fc:*`、**`yundun-cert:*`**（不是 `cas:*`，见坑 5）。

改已有策略：`create_policy_version(CreatePolicyVersionRequest(policy_name, policy_document, set_as_default=True))`；
版本上限 5，超了先 `delete_policy_version` 删最旧的非默认版。

直接跑 `scripts/setup_oidc.py` 可一步建完上述三者（AK 从环境变量读）。

## 二、workflow：手动 OIDC 兑换（不要依赖官方 action）

**不要用 `aliyun/configure-aliyun-credentials-action`** —— RAM 配置完全正确时它仍会报
`AuthenticationFail.NoPermission / ImplicitDeny / sts:AssumeRole / PolicyType: AssumeRolePolicy`，且难排查。
改为自己请求 JWT + `AssumeRoleWithOIDC`，并把 `iss/aud/sub` 打出来对照信任策略。

```yaml
permissions: { contents: read, id-token: write }
steps:
  - uses: actions/checkout@v4
  - uses: actions/setup-python@v5
    with: { python-version: "3.13" }
  - run: pip install alibabacloud_sts20150401 alibabacloud_alidns20150109 \
             alibabacloud_cas20200407 alibabacloud_fc_open20210406 oss2 acme cryptography
  - name: OIDC -> STS 临时凭证
    env:
      ALI_ROLE_ARN: ${{ secrets.ALI_ROLE_ARN }}
      ALI_OIDC_ARN: ${{ secrets.ALI_OIDC_ARN }}
      AUDIENCE: https://github.com/<owner>/<repo>
    run: python scripts/oidc_sts_env.py     # 本 skill 自带，等价于下面的内联版
```

内联等价实现（要点全在注释里）：

```python
# 1) 请求 OIDC JWT：URL 已带 ?job_id=...，必须「追加」&audience=
url = os.environ['ACTIONS_ID_TOKEN_REQUEST_URL'] + '&audience=' + os.environ['AUDIENCE']
req = urllib.request.Request(url, headers={
    'Authorization': 'Bearer ' + os.environ['ACTIONS_ID_TOKEN_REQUEST_TOKEN']})
oidc = json.load(urllib.request.urlopen(req, timeout=30))['value']

# 2) 解码 payload 打出 RAM 实际看到的 claim（定位 ImplicitDeny 的神器）
p = oidc.split('.')[1]; p += '=' * (-len(p) % 4)
cl = json.loads(base64.urlsafe_b64decode(p))
print('OIDC claims -> iss=%r aud=%r sub=%r' % (cl.get('iss'), cl.get('aud'), cl.get('sub')))

# 3) AssumeRoleWithOIDC（请求字段全 snake_case，且要重试吸收偶发 500）
sts = StsClient(openapi_models.Config(endpoint='sts.aliyuncs.com'))
r = sts.assume_role_with_oidc(sts_models.AssumeRoleWithOIDCRequest(
    oidcprovider_arn=os.environ['ALI_OIDC_ARN'], role_arn=os.environ['ALI_ROLE_ARN'],
    oidctoken=oidc, role_session_name='cert-renew'))
c = r.body.credentials      # access_key_id / access_key_secret / security_token / expiration
with open(os.environ['GITHUB_ENV'], 'a') as f:
    f.write('ALIBABA_CLOUD_ACCESS_KEY_ID=%s\n' % c.access_key_id)
    f.write('ALIBABA_CLOUD_ACCESS_KEY_SECRET=%s\n' % c.access_key_secret)
    f.write('ALIBABA_CLOUD_SECURITY_TOKEN=%s\n' % c.security_token)
```

## 三、cert_manager.py 的三条硬要求

```python
STOKEN = os.environ.get('ALIBABA_CLOUD_SECURITY_TOKEN', '')

# 1) oss2 必须按有无 token 选 Auth 实现（坑 6）
auth = oss2.StsAuth(AK, SK, STOKEN) if STOKEN else oss2.Auth(AK, SK)

# 2) 阿里云 SDK Config 透传 token
cfg.security_token = STOKEN          # 长期 AK 场景为空，可忽略

# 3) 签发与分发解耦 + 指纹守卫（坑 7、8）
need_issue = DO('cert') and (left <= THRESHOLD or FORCE)   # 证书有效只跳过签发，仍跑分发
cert_fp = hashlib.sha256(pem).hexdigest()
if cert_fp != state.get('cert_fp'):                        # 指纹变了才重传 CAS
    old = state.get('cert_id')
    if old:
        cas.delete_user_certificate(DeleteUserCertificateRequest(cert_id=old))  # 注意是 cert_id
    cert_id = cas.upload_user_certificate(UploadUserCertificateRequest(
        name=CAS_NAME, cert=pem, key=privkey_pem)).body.cert_id
```

## 四、状态持久化（不做就会撞 LE 限额）

runner 是临时容器，`account.key` / `privkey.pem` / `state.json` 必须存外部。
用 OSS 私有桶：跑前 `get_object_to_file`、跑后 `put_object_from_file`，桶不存在自动建（建桶写法见坑 9）。
否则每次运行都当首次签发，**撞 Let's Encrypt「每域名每周 5 张重复证书」限额**。

## 五、keepalive：别让 GitHub 停掉 schedule

GitHub 对**连续 60 天无活动**的仓库会禁用 `schedule`，且 workflow 自己跑不算活动。
专用仓加 `keepalive.yml`，每月 1 日用 `GITHUB_TOKEN`（`permissions: contents: write`）改一个时间戳文件并 push：

```yaml
on:
  schedule: [{ cron: "0 3 1 * *" }]
  workflow_dispatch:
permissions: { contents: write }
```

## 六、新子域「无感」接入（DNS 自动发现）

- **证书侧本来就无感**：签的是通配 `example.com + *.example.com`，`*.` 覆盖任意**一级**子域，
  新建子域无需重新签发、不消耗配额。注意 `*.` **不覆盖多级子域**（`a.b.example.com`）。
- 真正的手工成本在「绑定」。加 `discover_targets()` 扫根域 CNAME（分页 100/页）：
  - `<bucket>.oss-<region>.aliyuncs.com` → 正则反解 bucket + region，自动生成 OSS target
  - `<uid>.<region>.fcapp.run` → FC target（service/function 无法从 DNS 推导，取 `FC_TARGETS[0]` 兜底）
  - 非阿里云 CNAME（Vercel / 其他 CDN）与 `_` 开头记录（`_dnsauth` / `_acme-challenge`）跳过
- `merge_targets(manual, auto)`：**手工优先**，自动发现按 `domain` 去重追加；分发循环遍历合并结果；
  提供 `--no-discover` 退回纯手工模式。
- 效果：新建子域**只需加一条 CNAME**，下次定时任务自动写 `_dnsauth` TXT 验归属 + 绑 `cert_id` + 上 HTTPS。

## 七、必坑清单（按踩中概率排序）

1. **迁移仓库 / 改 audience 必须三处同步**，否则 `AuthenticationFail.OIDCToken.AudienceNotMatch` / `Invalid audience.`：
   ① workflow 的 `audience`；② 角色信任策略的 `oidc:aud`（`update_role`）；
   ③ **OIDC 提供商的 `ClientIds`**（IMS `add_client_id_to_oidcprovider` / `remove_client_id_from_oidcprovider`）。
   只改前两处是典型漏网。

2. **官方 action 莫名 `ImplicitDeny`**：`aliyun/configure-aliyun-credentials-action` 在 RAM 全对时仍报
   `NoPermission / ImplicitDeny / PolicyType: AssumeRolePolicy`。改用第二节的手动兑换。
   `get_role` 返回的 `Principal.Federated` 是数组，那是 RAM 规范化结果，**不是 bug**，别去改它。

3. **`oidc:sub` 别用 `StringEquals` 写死 `repo:<owner>/<repo>:*`**：GitHub 在 fork / 重跑 / 某些事件下会把
   **repo ID 与 owner ID 嵌进 sub**，变成 `repo:oracis@4110185/myrepo@1368066874:ref:refs/heads/main`。
   用 `StringLike` + `repo:<owner>*/<repo>*:ref:refs/heads/main` 一套覆盖两种格式。
   **打印 JWT 的 `sub` 是定位此类问题的唯一快路径。**

4. **STS 偶发 `InternalError 500`**：凭证指纹核对无误仍 500 时属瞬时故障，`AssumeRoleWithOIDC` 加
   3~5 次指数退避重试即可。

5. **CAS 的 RAM 命名空间是 `yundun-cert`，不是 `cas`**：策略写 `cas:*` **覆盖不到**
   `UploadUserCertificate`，报 `NoPermission / 403 / yundun-cert:UploadUserCertificate / AccountLevelIdentityBasedPolicy`。
   必须加 `yundun-cert:*`。

6. **oss2 用 STS 必须 `oss2.StsAuth`，不能把 token 当 `oss2.Auth` 第 3 个参数**：
   `oss2.Auth.__init__(access_key_id, access_key_secret)` 只收 2 个参数，传第 3 个直接
   `TypeError: Auth.__init__() takes 3 positional arguments but 4 were given`。
   正确：`oss2.StsAuth(ak, sk, stoken) if stoken else oss2.Auth(ak, sk)`。

7. **CAS 删除参数是 `cert_id`，不是 `certificate_id`；且 SDK 查不到上传型证书**：
   写成 `certificate_id=` 会静默不删（被 `except` 吞掉），旧证书**长期占用固定名称**，
   下次上传必报 `NameRepeat (code:400, 名称重复)`。`cas20200407` 只有
   `list_user_certificate_order`（仅购买订单），**没有** `DescribeUserCertificateList`，
   所以 `state.json` 里的 `cert_id` 是定位旧证书的唯一途径。
   稳健写法：上传前按 state 的 `cert_id` 删除（失败仅告警）；若仍 `NameRepeat`，用「固定名+时间戳」重传兜底。

8. **签发与分发必须解耦，证书有效不能整体 `sys.exit(0)`**：否则上次分发失败后，
   有效证书会永远卡在「已签发未分发」。用 `need_issue` 只控制签发，CAS 上传加 `sha256(pem)` 指纹守卫，
   指纹一致就复用旧 `cert_id`，避免每天重复创建 CAS 证书。

9. **oss2 `create_bucket` 第一个位置参数是 `permission`（ACL 字符串），不是 `input`（config 对象）**：
   签名 `create_bucket(self, permission=None, input=None, headers=None)`。
   写 `create_bucket(oss2.models.BucketCreateConfig(oss2.BUCKET_ACL_PRIVATE))` 会把 config 塞进
   `x-oss-acl` header，签名时 `TypeError: can't concat BucketCreateConfig to bytes`。
   正确：`b.create_bucket(oss2.BUCKET_ACL_PRIVATE)`（默认即私有桶）。
   `BucketCreateConfig` 构造器也按 snake_case：`storage_class` / `data_redundancy_type` / `acl`。

10. **cron 时区是 UTC**：北京 03:30 = `30 19 * * *`。schedule 有 ±15 分钟漂移，对续期无影响。
    `wait_txt()` 等 DNS 轮询在 Linux runner 用 `dig`（需 `dnsutils`），Windows 无 dig 回退 `nslookup`。

## 八、验收清单

跑一次 workflow，日志应依次出现：

```
OIDC claims -> iss=... aud=... sub=repo:<owner>/<repo>:ref:refs/heads/main
STS 临时凭证已获取
新证书已签发，有效期至 ...
已上传 CAS，cert_id = ...
处理 OSS <domain> -> 已绑定自定义域名
FC 自定义域名 <domain> 证书已更新
cert state saved
```

再去浏览器核对三个站点的证书颁发者（Let's Encrypt R11/R12）与到期时间即可。

## 附：脚本

- `scripts/setup_oidc.py` —— 一次性建 OIDC 提供商 + 角色 + 权限策略（读环境变量里的永久 AK）。
- `scripts/oidc_sts_env.py` —— workflow 用：OIDC JWT → `AssumeRoleWithOIDC` → 写 `$GITHUB_ENV`。
