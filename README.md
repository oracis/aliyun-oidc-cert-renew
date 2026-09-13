# aliyun-oidc-cert-renew

用 **GitHub Actions + 阿里云 RAM OIDC** 自动续期 Let's Encrypt 通配证书并分发到 OSS / FC / CDN，
**仓库里不放任何长期 AK/SK**。

## 它解决什么

| 问题 | 做法 |
|---|---|
| 定时任务要 AK/SK，存 Secrets 有泄露风险 | OIDC 换 1 小时临时 STS，仓库只存两个非敏感 ARN |
| 每次跑都重签，撞 Let's Encrypt 每周 5 张限额 | ACME 状态持久化到 OSS 私有桶 + 指纹守卫 |
| 上次分发失败，证书卡在「已签发未分发」 | 签发与分发解耦，证书有效也照跑分发 |
| 新子域要改代码才能上 HTTPS | DNS 自动发现：加一条 CNAME 就自动绑证书 |
| 仓库 60 天没动静，GitHub 停掉 schedule | keepalive 每月自推送 |

## 结构

```
SKILL.md                  完整实施手册 + 10 条 SDK 命名坑
manifest.yaml             市场发布元数据
scripts/setup_oidc.py     一次性建 OIDC 提供商 / RAM 角色 / 权限策略
scripts/oidc_sts_env.py   workflow 用：OIDC JWT → AssumeRoleWithOIDC → $GITHUB_ENV
```

## 快速开始

```bash
# 1. 一次性搭 RAM（永久 AK 从环境变量读，不落盘）
export ALIBABA_CLOUD_ACCESS_KEY_ID=LTAI...
export ALIBABA_CLOUD_ACCESS_KEY_SECRET=...
pip install alibabacloud_ims20190815 alibabacloud_ram20150501 alibabacloud_sts20150401
python scripts/setup_oidc.py --owner <owner> --repo <repo>

# 2. 仓库 Secrets 填脚本输出的两个 ARN，workflow 加一步
python scripts/oidc_sts_env.py
```

workflow 需要 `permissions: { id-token: write }`，之后所有步骤的阿里云 SDK / oss2
都能从 `ALIBABA_CLOUD_ACCESS_KEY_ID` / `_SECRET` / `ALIBABA_CLOUD_SECURITY_TOKEN` 读到临时凭证。

## 许可

MIT
