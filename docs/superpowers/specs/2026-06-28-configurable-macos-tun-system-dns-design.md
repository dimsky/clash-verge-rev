# Configurable macOS TUN System DNS

## Goal

Replace the hard-coded `114.114.114.114` address used when Clash Verge Rev
overwrites macOS system DNS in TUN mode with a user-configurable single IPv4
address. The default and compatibility fallback is `8.8.8.8`.

This setting controls only the address written to macOS network services. It
does not change the upstream resolvers configured in `dns_config.yaml`.

## Configuration

Add `tun_system_dns: Option<String>` to `IVerge`.

- New installations default to `8.8.8.8`.
- Existing configurations without the field use `8.8.8.8`.
- Valid custom IPv4 addresses are persisted in `verge.yaml`.
- Invalid or empty values are not passed to the system DNS command; the backend
  falls back to `8.8.8.8`.

The field must participate in the existing `IVerge::patch_config` flow.

## Backend Data Flow

`get_config_values()` reads and validates `tun_system_dns`, then includes the
resolved value in `ConfigValues`. `enhance()` passes it to `use_tun()`.

When macOS TUN DNS overwrite runs, `use_tun()` restores the previous DNS and
then calls `set_public_dns()` with the resolved configurable address instead of
the current hard-coded `114.114.114.114`.

Validation uses Rust's IPv4 address parser. Only a plain IPv4 address is valid;
IPv6, CIDR notation, ports, multiple addresses, empty strings, and malformed
values are rejected.

## Settings UI

On macOS, add a compact “System DNS” text field next to the existing DNS
overwrite setting.

- Initial value comes from `verge.tun_system_dns`, with `8.8.8.8` as fallback.
- Commit on blur or Enter.
- Validate before saving.
- On invalid input, keep the last valid persisted value and display the
  project's standard error notice.
- Other operating systems do not display this field because this system DNS
  overwrite path is macOS-specific.

Add localized labels and validation messages following the existing settings
translation structure.

## Tests and Verification

Backend unit tests cover:

1. Missing configuration resolves to `8.8.8.8`.
2. A valid custom IPv4 address is preserved.
3. Empty and invalid values resolve to `8.8.8.8`.
4. IPv6, CIDR, port-qualified, and multiple-address inputs are rejected.

Verification also includes Rust formatting and targeted tests, frontend type
checking, and linting of affected files where supported by the repository.

## Scope

The change does not:

- Modify Clash/Mihomo upstream DNS resolver configuration.
- Add support for multiple DNS addresses or IPv6.
- Change behavior on Windows or Linux.
- Automatically rewrite an already-running system DNS value until the normal
  TUN configuration apply path runs.
