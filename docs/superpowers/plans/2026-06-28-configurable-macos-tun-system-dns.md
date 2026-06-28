# Configurable macOS TUN System DNS Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let macOS users configure the single IPv4 address written to system DNS during TUN DNS overwrite, defaulting safely to `8.8.8.8`.

**Architecture:** Add a persisted optional Verge setting and normalize it in Rust before it reaches the macOS TUN path. Expose the setting through the existing Clash settings screen, reuse the shared IPv4 validator, and retain backend fallback protection for manually edited configuration files.

**Tech Stack:** Rust 2024, Tauri 2, React 19, TypeScript, Material UI, i18next, Cargo tests, pnpm.

---

## File Map

- Modify `src-tauri/src/config/verge.rs`: define, default, and patch the persisted setting.
- Modify `src-tauri/src/enhance/mod.rs`: normalize the setting and carry it through enhancement state.
- Modify `src-tauri/src/enhance/tun.rs`: consume the resolved DNS instead of a hard-coded address and host unit tests.
- Modify `src/types/global.d.ts`: expose the persisted field to TypeScript.
- Modify `src/components/setting/setting-clash.tsx`: render and save the macOS-only input.
- Modify `src/locales/*/settings.json`: add labels and validation messages for all shipped locales.

### Task 1: Add backend DNS normalization with tests

**Files:**
- Modify: `src-tauri/src/enhance/tun.rs`

- [ ] **Step 1: Write failing normalization tests**

Add a pure helper contract and tests at the bottom of `tun.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::resolve_tun_system_dns;

    #[test]
    fn defaults_missing_tun_system_dns_to_google_dns() {
        assert_eq!(resolve_tun_system_dns(None), "8.8.8.8");
    }

    #[test]
    fn preserves_valid_custom_ipv4_tun_system_dns() {
        assert_eq!(resolve_tun_system_dns(Some("1.1.1.1")), "1.1.1.1");
    }

    #[test]
    fn rejects_invalid_tun_system_dns_values() {
        for value in [
            "",
            "not-an-ip",
            "2001:4860:4860::8888",
            "8.8.8.8/32",
            "8.8.8.8:53",
            "8.8.8.8,1.1.1.1",
        ] {
            assert_eq!(resolve_tun_system_dns(Some(value)), "8.8.8.8");
        }
    }
}
```

- [ ] **Step 2: Run the targeted test and verify RED**

Run:

```bash
cargo test --manifest-path src-tauri/Cargo.toml enhance::tun::tests
```

Expected: compilation fails because `resolve_tun_system_dns` does not exist.

- [ ] **Step 3: Implement minimal normalization**

Add:

```rust
const DEFAULT_TUN_SYSTEM_DNS: &str = "8.8.8.8";

pub(crate) fn resolve_tun_system_dns(value: Option<&str>) -> String {
    value
        .and_then(|value| value.parse::<std::net::Ipv4Addr>().ok())
        .map(|value| value.to_string())
        .unwrap_or_else(|| DEFAULT_TUN_SYSTEM_DNS.to_string())
}
```

- [ ] **Step 4: Run the targeted test and verify GREEN**

Run the command from Step 2.

Expected: all three `enhance::tun::tests` pass.

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/enhance/tun.rs
git commit -m "test: define macOS TUN system DNS validation"
```

### Task 2: Persist and pass the configurable DNS

**Files:**
- Modify: `src-tauri/src/config/verge.rs`
- Modify: `src-tauri/src/enhance/mod.rs`
- Modify: `src-tauri/src/enhance/tun.rs`

- [ ] **Step 1: Add a failing configuration-default test**

Add a test module at the bottom of `config/verge.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::IVerge;

#[test]
fn default_tun_system_dns_is_google_dns() {
    assert_eq!(
        IVerge::new().tun_system_dns.as_deref(),
        Some("8.8.8.8")
    );
}
}
```

- [ ] **Step 2: Run the test and verify RED**

Run:

```bash
cargo test --manifest-path src-tauri/Cargo.toml default_tun_system_dns_is_google_dns
```

Expected: compilation fails because `IVerge::tun_system_dns` does not exist.

- [ ] **Step 3: Add the persisted field**

Add to `IVerge` near `enable_dns_settings`:

```rust
/// IPv4 address written to macOS system DNS while TUN DNS overwrite is active
pub tun_system_dns: Option<String>,
```

Add to `IVerge::new()`:

```rust
tun_system_dns: Some("8.8.8.8".into()),
```

Add to `patch_config()`:

```rust
patch!(tun_system_dns);
```

- [ ] **Step 4: Carry the normalized value through enhancement**

Add `tun_system_dns: String` to `ConfigValues`. In `get_config_values()`, destructure the optional field and resolve it:

```rust
ref tun_system_dns,
```

```rust
let tun_system_dns =
    tun::resolve_tun_system_dns(tun_system_dns.as_deref());
```

Include it in the `ConfigValues` construction and `enhance()` destructuring, then change:

```rust
config = use_tun(config, enable_tun);
```

to:

```rust
config = use_tun(config, enable_tun, tun_system_dns);
```

- [ ] **Step 5: Use the configured value in the macOS TUN path**

Change the function signature:

```rust
pub fn use_tun(mut config: Mapping, enable: bool, tun_system_dns: String) -> Mapping
```

Replace:

```rust
crate::utils::resolve::dns::set_public_dns("114.114.114.114".to_string()).await;
```

with:

```rust
crate::utils::resolve::dns::set_public_dns(tun_system_dns).await;
```

- [ ] **Step 6: Run backend tests and formatting**

Run:

```bash
cargo fmt --manifest-path src-tauri/Cargo.toml --check
cargo test --manifest-path src-tauri/Cargo.toml enhance::tun::tests
cargo test --manifest-path src-tauri/Cargo.toml default_tun_system_dns_is_google_dns
```

Expected: formatting succeeds and all targeted tests pass.

- [ ] **Step 7: Commit**

```bash
git add src-tauri/src/config/verge.rs src-tauri/src/enhance/mod.rs src-tauri/src/enhance/tun.rs
git commit -m "feat: configure macOS TUN system DNS"
```

### Task 3: Add the macOS settings input

**Files:**
- Modify: `src/types/global.d.ts`
- Modify: `src/components/setting/setting-clash.tsx`
- Modify: `src/locales/ar/settings.json`
- Modify: `src/locales/de/settings.json`
- Modify: `src/locales/en/settings.json`
- Modify: `src/locales/es/settings.json`
- Modify: `src/locales/fa/settings.json`
- Modify: `src/locales/id/settings.json`
- Modify: `src/locales/jp/settings.json`
- Modify: `src/locales/ko/settings.json`
- Modify: `src/locales/ru/settings.json`
- Modify: `src/locales/tr/settings.json`
- Modify: `src/locales/tt/settings.json`
- Modify: `src/locales/zh/settings.json`
- Modify: `src/locales/zhtw/settings.json`

- [ ] **Step 1: Expose the field to TypeScript**

Add near `enable_dns_settings`:

```ts
tun_system_dns?: string
```

- [ ] **Step 2: Add localized strings**

Add these keys under `settings.sections.clash.form.fields` in every locale:

```json
"tunSystemDns": "System DNS",
"invalidTunSystemDns": "Enter a valid IPv4 address"
```

Use native translations for Chinese:

```json
"tunSystemDns": "系统 DNS",
"invalidTunSystemDns": "请输入有效的 IPv4 地址"
```

Use English fallbacks for locales without a verified native translation.

- [ ] **Step 3: Add macOS-only input state and validation**

Use the project's existing `validator` dependency:

```tsx
import validator from 'validator'
```

Add platform state:

```tsx
const isMAC = getSystem() === 'macos'
```

Track the draft:

```tsx
const [tunSystemDns, setTunSystemDns] = useState(
  () => verge?.tun_system_dns ?? '8.8.8.8',
)
```

Add the commit handler:

```tsx
const commitTunSystemDns = useLockFn(async () => {
  const value = tunSystemDns.trim()
  if (!validator.isIP(value, 4)) {
    setTunSystemDns(verge?.tun_system_dns ?? '8.8.8.8')
    showNotice.error(
      new Error(t('settings.sections.clash.form.fields.invalidTunSystemDns')),
    )
    return
  }
  await patchVerge({ tun_system_dns: value })
  setTunSystemDns(value)
})
```

- [ ] **Step 4: Render the input**

Directly after the DNS overwrite switch, render:

```tsx
{isMAC && (
  <SettingItem
    label={t('settings.sections.clash.form.fields.tunSystemDns')}
  >
    <TextField
      size="small"
      value={tunSystemDns}
      sx={{ width: 140, input: { py: '7.5px' } }}
      onChange={(event) => setTunSystemDns(event.target.value)}
      onBlur={commitTunSystemDns}
      onKeyDown={(event) => {
        if (event.key === 'Enter') {
          event.currentTarget.blur()
        }
      }}
    />
  </SettingItem>
)}
```

- [ ] **Step 5: Regenerate i18n key types and run frontend checks**

Run:

```bash
pnpm i18n:types
pnpm typecheck
pnpm lint
pnpm format:check
```

Expected: all commands exit successfully with no warnings.

- [ ] **Step 6: Commit**

```bash
git add src/types/global.d.ts src/components/setting/setting-clash.tsx src/locales
git commit -m "feat: add macOS TUN system DNS setting"
```

### Task 4: Final regression verification

**Files:**
- Verify all modified files.

- [ ] **Step 1: Confirm the old runtime default is gone**

Run:

```bash
rg -n '114\.114\.114\.114' src-tauri/src src
```

Expected: no matches.

- [ ] **Step 2: Run the complete relevant verification suite**

Run:

```bash
cargo fmt --manifest-path src-tauri/Cargo.toml --check
cargo test --manifest-path src-tauri/Cargo.toml
pnpm typecheck
pnpm lint
pnpm format:check
pnpm i18n:check
git diff --check HEAD~3
```

Expected: every command exits successfully.

- [ ] **Step 3: Review final diff and repository state**

Run:

```bash
git diff HEAD~3 -- src-tauri/src/config/verge.rs src-tauri/src/enhance/mod.rs src-tauri/src/enhance/tun.rs src/types/global.d.ts src/components/setting/setting-clash.tsx src/locales
git status --short
```

Expected: the diff matches the approved design and the worktree is clean.
