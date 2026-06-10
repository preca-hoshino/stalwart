# Stalwart Enterprise Patched — 完整构建与部署方案

> 基于 Stalwart v0.16.8 (2026-06-06) | 私有 Fork + GitHub Actions 自动编译 + Docker 部署
> 仅限个人使用，不分发

---

## 一、快速操作流程

### Phase 1: Fork + 私有化（2 分钟）

```bash
gh repo fork stalwartlabs/stalwart --clone=false --fork-name stalwart
gh repo edit stalwart --visibility private --accept-visibility-change-consequences
gh repo clone stalwart && cd stalwart
git checkout v0.16.8
git checkout -b v0.16.8-patch-1
mkdir -p scripts .github/workflows
```

将下方「补丁脚本」「CI/CD 工作流」分别写入对应文件，然后：

```bash
git add -A && git commit -m "Enterprise: unlock enterprise features for personal use"
git push origin v0.16.8-patch-1
```

### Phase 2: Actions 自动编译（~45 分钟）

推送后自动触发，产出：

| 产物 | 说明 |
|------|------|
| `stalwart-x86_64-unknown-linux-gnu.tar.gz` | Linux amd64 二进制 |
| `stalwart-x86_64-unknown-linux-musl.tar.gz` | Linux amd64 静态链接 |
| `stalwart-aarch64-unknown-linux-gnu.tar.gz` | Linux arm64 二进制 |
| `stalwart-aarch64-unknown-linux-musl.tar.gz` | Linux arm64 静态链接 |
| `ghcr.io/<你>/stalwart:v0.16.8-patch-1` | Docker 镜像 (amd64+arm64) |
| GitHub Release | 自动发版，附带二进制 + sha256 |

### Phase 3: 服务器部署（5 分钟）

```bash
# 登录 GHCR（需 PAT，权限 read:packages）
echo $CR_PAT | docker login ghcr.io -u YOUR_USERNAME --password-stdin

# ⚠️ 先编辑下方「部署配置」中的 image 地址
cd /docker/stalwart && docker compose up -d
docker logs -f stalwart-mail
# 访问 https://d-mail.restart.org.cn 完成初始向导
```

### Phase 4: 邮件迁移（可选，30-60 分钟）

```bash
# Poste.io 仍在运行时逐邮箱迁移
imapsync --host1 127.0.0.1 --port1 993 --ssl1 \
         --user1 user@domain --password1 'xxx' \
         --host2 127.0.0.1 --port2 993 --ssl2 \
         --user2 user@domain --password2 'xxx'
# MX/SPF/DMARC 不变，DKIM 在 Stalwart 重新生成后更新 DNS
# 确认正常后：cd /docker/poste.io && docker compose down
```

### Phase 5: 后续升级

```bash
gh repo sync OWNER/stalwart -b v0.16.8-patch-1 --source stalwartlabs/stalwart
# Actions 自动重编译
docker compose pull && docker compose up -d
```

---

## 二、补丁脚本 `scripts/apply-patches.sh`

```bash
#!/usr/bin/env bash
# ============================================================
# apply-patches.sh — 最小化解锁 Stalwart Enterprise 功能
# ============================================================
# 用法: bash scripts/apply-patches.sh
# 在 Stalwart 源码根目录执行
# ============================================================
set -euo pipefail

echo "[patch] 开始应用 Enterprise 补丁..."

# ── 注: Cargo.toml 无需修改 ─────────────────────────────────
# 当前 v0.16.8 的 default features 已包含 enterprise:
#   default = ["rocks", "enterprise"]
# CI 使用 --no-default-features --features 覆盖，无需改动。

# ── Patch 1: license.rs — 跳过签名 + 永不过期 ──────────────
FILE="crates/common/src/enterprise/license.rs"
echo "[1/3] $FILE"
python3 -c "
import sys
p='$FILE'
c=open(p).read()

# 1a: 注释签名验证，直接返回 Ok
c=c.replace(
    '''        // Validate signature
        self.public_key
            .verify(
                &key[..(U64_LEN * 2) + (U32_LEN * 2) + domain_len],
                signature,
            )
            .map_err(|_| LicenseError::Validation)?;

        let key = LicenseKey {
            valid_from,
            valid_to,
            domain,
            accounts,
        };

        if !key.is_expired() {
            Ok(key)
        } else {
            Err(LicenseError::Expired)
        }''',
    '''        let key = LicenseKey {
            valid_from,
            valid_to,
            domain,
            accounts,
        };
        Ok(key) // PATCHED: signature + expiration bypassed'''
)

# 1b: is_expired 恒为 false
c=c.replace(
    '''    pub fn is_expired(&self) -> bool {
        let now = now();
        now >= self.valid_to || now < self.valid_from
    }''',
    '''    pub fn is_expired(&self) -> bool { false } // PATCHED'''
)

# 1c: 放宽参数校验（去掉 accounts==0 检查）
c=c.replace(
    '''        if valid_from == 0
            || valid_to == 0
            || valid_from >= valid_to
            || accounts == 0
            || domain.is_empty()''',
    '''        if valid_from == 0
            || valid_to == 0
            || valid_from >= valid_to
            || domain.is_empty() // PATCHED: allow accounts=0'''
)

open(p,'w').write(c)
print('  ✓ license.rs')
"

# ── Patch 2: mod.rs — 账户数无限制 ─────────────────────────
FILE="crates/common/src/enterprise/mod.rs"
echo "[2/3] $FILE"
python3 -c "
import sys
p='$FILE'
c=open(p).read()
c=c.replace(
    '''    pub async fn can_create_account(&self) -> trc::Result<bool> {
        if let Some(enterprise) = &self.core.enterprise {
            let total_accounts = self.total_accounts().await.caused_by(trc::location!())?;

            if total_accounts + 1 > enterprise.license.accounts as usize {
                trc::event!(
                    Server(trc::ServerEvent::Licensing),
                    Details = \"Account creation not possible: license key account limit reached\",
                    Domain = enterprise.license.domain.clone(),
                    Total = total_accounts,
                    Limit = enterprise.license.accounts,
                );

                return Ok(false);
            }
        }

        Ok(true)
    }''',
    '''    pub async fn can_create_account(&self) -> trc::Result<bool> {
        Ok(true) // PATCHED: no account limit
    }'''
)
open(p,'w').write(c)
print('  ✓ mod.rs')
"

# ── Patch 3: config.rs — 无 license 时生成默认有效 license ──
FILE="crates/common/src/enterprise/config.rs"
echo "[3/3] $FILE"
python3 -c "
p='$FILE'
c=open(p).read()

# 3a: 替换整个 license_result match 块
OLD = '''        let license_result = match (
            enterprise.license_key.secret().await,
            enterprise.api_key.secret().await,
        ) {
            (Ok(Some(license_key)), Ok(maybe_api_key)) => {
                match (
                    LicenseKey::new(license_key, &server_hostname),
                    maybe_api_key,
                ) {
                    (Ok(license), Some(api_key)) if license.is_near_expiration() => Ok(license
                        .try_renew(api_key.as_ref())
                        .await
                        .map(|result| {
                            update_license = Some(result.encoded_key);
                            result.key
                        })
                        .unwrap_or(license)),
                    (Ok(license), None) => Ok(license),
                    (Err(_), Some(api_key)) => LicenseKey::invalid(&server_hostname)
                        .try_renew(api_key.as_ref())
                        .await
                        .map(|result| {
                            update_license = Some(result.encoded_key);
                            result.key
                        }),
                    (maybe_license, _) => maybe_license,
                }
            }
            (Ok(None), Ok(Some(api_key))) => LicenseKey::invalid(&server_hostname)
                .try_renew(api_key.as_ref())
                .await
                .map(|result| {
                    update_license = Some(result.encoded_key);
                    result.key
                }),
            (Ok(None), Ok(None)) => {
                #[cfg(not(feature = \"test_mode\"))]
                return None;

                #[cfg(feature = \"test_mode\")]
                Ok(LicenseKey {
                    valid_to: store::write::now() + (86400 * 365),
                    valid_from: store::write::now() - 3600,
                    domain: server_hostname.to_string(),
                    accounts: 100,
                })
            }
            (Err(err), _) => {
                bp.build_error(ObjectType::Enterprise.singleton(), err);
                return None;
            }
            (_, Err(err)) => {
                bp.build_error(ObjectType::Enterprise.singleton(), err);
                return None;
            }
        };'''

NEW = '''        // PATCHED: bypass license validation, always generate valid license
        let license_result: Result<LicenseKey, _> = match enterprise.license_key.secret().await {
            Ok(Some(license_key)) => {
                use super::license::LicenseValidator;
                LicenseValidator::new().try_parse(&license_key).or_else(|_| {
                    Ok(LicenseKey {
                        valid_to: store::write::now() + (86400 * 365 * 100),
                        valid_from: store::write::now() - 3600,
                        domain: server_hostname.to_string(),
                        accounts: u32::MAX,
                    })
                })
            }
            _ => Ok(LicenseKey {
                valid_to: store::write::now() + (86400 * 365 * 100),
                valid_from: store::write::now() - 3600,
                domain: server_hostname.to_string(),
                accounts: u32::MAX,
            }),
        };'''
c=c.replace(OLD,NEW)

# 3b: 注释账户配额检查
OLD_Q='''        match bp
            .registry
            .query::<RoaringBitmap>(RegistryQuery::new(ObjectType::Account))
            .await
        {
            Ok(total) if total.len() > license.accounts as u64 => {
                bp.build_warning(
                    ObjectType::Enterprise.singleton(),
                    format!(
                        \"License key is valid but only \\\\
                        allows {} accounts, found {}.\",
                        license.accounts,
                        total.len()
                    ),
                );
                return None;
            }
            Err(e) => {
                trc::error!(
                    e.caused_by(trc::location!())
                        .details(\"Failed to count total individual principals\")
                );
                return None;
            }
            _ => (),
        }'''
NEW_Q='        // PATCHED: skip account quota check'
c=c.replace(OLD_Q,NEW_Q)

open(p,'w').write(c)
print('  ✓ config.rs')
"

echo ""
echo "[patch] ✅ 所有补丁应用完成"
```

---

## 三、GitHub Actions 工作流 `.github/workflows/release.yml`

```yaml
# ============================================================
# Stalwart v0.16.8-patch-1 — GitHub Actions Release Workflow
# ============================================================
# 触发: push tag v* 时自动构建
# 产出: 二进制 (4平台) + Docker镜像 (GHCR) + GitHub Release
# ============================================================

name: Release

on:
  push:
    tags: ['v*']
  workflow_dispatch:

env:
  CARGO_TERM_COLOR: always
  FEATURES: "sqlite,rocks,enterprise"

permissions:
  contents: write
  packages: write

jobs:
  patch:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Checkout upstream Stalwart
        uses: actions/checkout@v4
        with:
          repository: stalwartlabs/stalwart
          ref: v0.16.8
          path: upstream

      - name: Apply patches
        run: |
          cp -r scripts upstream/scripts 2>/dev/null || true
          cd upstream
          bash scripts/apply-patches.sh

      - name: Upload patched source
        uses: actions/upload-artifact@v4
        with:
          name: patched-source
          path: upstream
          retention-days: 1

  build-binaries:
    needs: patch
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        include:
          - target: x86_64-unknown-linux-gnu
            artifact: stalwart-x86_64-unknown-linux-gnu
          - target: x86_64-unknown-linux-musl
            artifact: stalwart-x86_64-unknown-linux-musl
          - target: aarch64-unknown-linux-gnu
            artifact: stalwart-aarch64-unknown-linux-gnu
          - target: aarch64-unknown-linux-musl
            artifact: stalwart-aarch64-unknown-linux-musl
    steps:
      - name: Download patched source
        uses: actions/download-artifact@v4
        with:
          name: patched-source
          path: .

      - uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}

      - name: Install cross-compilation tools
        run: |
          sudo apt-get update
          case "${{ matrix.target }}" in
            x86_64-unknown-linux-gnu)
              echo "LINKER=gcc" >> $GITHUB_ENV ;;
            x86_64-unknown-linux-musl)
              sudo apt-get install -y musl-tools
              echo "LINKER=musl-gcc" >> $GITHUB_ENV ;;
            aarch64-unknown-linux-gnu)
              sudo apt-get install -y gcc-aarch64-linux-gnu
              echo "LINKER=aarch64-linux-gnu-gcc" >> $GITHUB_ENV ;;
            aarch64-unknown-linux-musl)
              sudo apt-get install -y gcc-aarch64-linux-gnu musl-tools
              echo "LINKER=aarch64-linux-gnu-gcc" >> $GITHUB_ENV ;;
          esac

      - name: Cache cargo
        uses: actions/cache@v4
        with:
          path: |
            ~/.cargo/registry
            ~/.cargo/git
            target
          key: ${{ matrix.target }}-cargo-${{ hashFiles('**/Cargo.lock') }}

      - name: Build
        run: |
          cargo build --release --target ${{ matrix.target }} -p stalwart \
            --no-default-features --features "${{ env.FEATURES }}"

      - name: Package binary
        run: |
          cd target/${{ matrix.target }}/release
          tar czf ../../../${{ matrix.artifact }}.tar.gz stalwart
          cd ../../..
          sha256sum ${{ matrix.artifact }}.tar.gz > ${{ matrix.artifact }}.tar.gz.sha256

      - name: Upload binary artifact
        uses: actions/upload-artifact@v4
        with:
          name: ${{ matrix.artifact }}
          path: |
            ${{ matrix.artifact }}.tar.gz
            ${{ matrix.artifact }}.tar.gz.sha256

  docker:
    needs: patch
    runs-on: ubuntu-latest
    steps:
      - name: Download patched source
        uses: actions/download-artifact@v4
        with:
          name: patched-source
          path: .

      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=semver,pattern={{version}}
            type=raw,value=v0.16.8-patch-1
            type=raw,value=latest

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  release:
    needs: [build-binaries, docker]
    runs-on: ubuntu-latest
    steps:
      - name: Download all binary artifacts
        uses: actions/download-artifact@v4
        with:
          path: artifacts
          pattern: stalwart-*
          merge-multiple: true

      - name: Create Release
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          ASSETS=""
          for f in artifacts/*.tar.gz artifacts/*.sha256; do
            [ -f "$f" ] && ASSETS="$ASSETS $f"
          done
          gh release create "${{ github.ref_name }}" \
            --repo "${{ github.repository }}" \
            --title "Stalwart ${{ github.ref_name }}" \
            --generate-notes \
            $ASSETS
```

---

## 四、服务器部署配置 `docker-compose.yml`

```yaml
# ⚠️ 使用前修改 image 地址为你的 GHCR 镜像
# mkdir -p /docker/stalwart/data/{config,store}

services:
  mailserver:
    image: ghcr.io/YOUR_USERNAME/stalwart:v0.16.8-patch-1
    container_name: stalwart-mail
    hostname: d-mail.restart.org.cn
    ports:
      - "25:25"
      - "8080:8080"
      - "443:443"
      - "110:110"
      - "143:143"
      - "465:465"
      - "587:587"
      - "993:993"
      - "995:995"
    environment:
      - TZ=Asia/Shanghai
    volumes:
      - ./data/config:/etc/stalwart
      - ./data/store:/var/lib/stalwart
    networks:
      - 1panel-network
    restart: always
    deploy:
      resources:
        limits:
          memory: 512m
          cpus: '1.0'
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

networks:
  1panel-network:
    external: true
```

---

## 五、补丁原理（3 文件，~28 行改动）

| 文件 | 改动 | 作用 |
|------|------|------|
| `crates/common/src/enterprise/license.rs` | ~8 行 | 注释 Ed25519 `.verify()` + `is_expired()` 恒返回 `false` + 去掉 `accounts==0` 检查 |
| `crates/common/src/enterprise/mod.rs` | ~5 行 | `can_create_account()` 恒返回 `Ok(true)` |
| `crates/common/src/enterprise/config.rs` | ~15 行 | 无 license 时生成 100 年有效默认 license（`u32::MAX` 账户）+ 注释配额检查 |

> **注**：`crates/main/Cargo.toml` 的 `default` features 已包含 `enterprise`（`default = ["rocks", "enterprise"]`），且 CI 使用 `--no-default-features --features` 覆盖，无需修改。

**企业名自定义**：默认 license 的 `domain` 字段取自服务器 hostname。修改 `apply-patches.sh` 中 `server_hostname.to_string()` 为固定字符串即可自定义。

---

## 六、时间线

| 阶段 | 耗时 | 说明 |
|------|------|------|
| Phase 1: Fork + 补丁 | 2min | 一次性 |
| Phase 2: CI/CD 编译 | 45min | 首次，后续自动 |
| Phase 3: 部署 | 5min | docker compose up |
| Phase 4: 迁移 | 30-60min | 取决于邮箱数 |
| Phase 5: 后续升级 | 2min/次 | sync fork + pull |
