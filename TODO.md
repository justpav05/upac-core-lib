<h1 align="center">✅ TODO</h1> 

Near-term, concrete items. See `ROADMAP.md` for the bigger picture.

## upac-cli

- `user/upac-cli/data/` (`.desktop`, `upac-mime.xml`, `.policy`) reference `Icon=upac`/`icon_name=upac`,
  but there's no actual icon asset (SVG/PNG) yet, and no install step wiring it into
  `/usr/share/icons/hicolor/...`. Needs real artwork before packaging.

## upac-lib

Test-coverage pass in progress. The entire non-command core is covered (`errors.rs`/`lock.rs`/
`search.rs`/`fs.rs`/`orchestrator/*`/`database/*`/`deploy/*`/`scripts/*`/`composefs/*`/`config/*`/
`boot/*`/`plugin/decoder/{error,manifest,triggers}.rs`/`plugin/boot/{error,manifest}.rs`), except
`plugin/decoder/unpack.rs`/`plugin/decoder/mod.rs`/`plugin/boot/mod.rs` (need a real dlopen'd/
`builtin-*` plugin) and `deploy/esp.rs` (real mount table) — both explicit, justified skips. Every
`mutated`/`unmutated` command's own `<Command>Error` enum is also now covered (inline tests next to
each `error.rs`, since `mutated`/`unmutated` aren't `pub`) — only each variant's own logic, not the
macro-generated `Common(...)` delegation shared with `errors.rs`'s already-tested `CommonError`.
Remaining: the `Stage::run()` bodies themselves — each needs a real composefs `Repository`/`Deploy`/
database in context, likely out of scope for unit tests unless a pure-logic helper turns out to be
extractable.

**Two standalone boot-time services still need to be built** — neither is upac-lib/upac-cli code,
both run on the installed system itself, outside anything `up`/`up-sp` invokes:

- **composefs-mount boot hook**: nothing yet resolves `composefs=<hash>` (the kernel cmdline param
  `write_boot_entry` already writes via `ComposefsCmdline::new_v2`) against the on-disk repository,
  mounts the erofs image with fs-verity, and overlays `state/deploy/<digest>/etc/` — without this, a
  genesis-produced disk's firmware boots the kernel, but the initramfs has no way to actually
  assemble the root. The upstream tool for this already exists (`composefs-setup-root`, crates.io,
  same `composefs-rs` project as our `composefs`/`composefs-boot` deps) — confirmed by reading its
  `main.rs`: it does NOT ship any systemd unit itself (only the binary — `Makefile`'s
  `install-setup-root` target installs nothing else), so the unit is ours to write. Confirmed its
  hardcoded expectations already match our on-disk layout exactly, no restructuring needed:
  `Repository::open_path(sysroot, "composefs")` ↔ `lib.toml`'s `repo_dir = "composefs"`;
  `state/deploy/<hex>/{etc,var}` ↔ `deploys_dir = "state/deploy"` +
  `TargetSysroot::deploy_dir(prefix_digest)`; `composefs=<hex>` karg ↔
  `ComposefsCmdline::new_v2(...).to_cmdline_arg()`. Remaining work: (1) write the actual `.service`
  unit (`After=sysroot.mount`, `Before=initrd-root-fs.target`/`initrd-switch-root.target`, same role
  as ostree's `ostree-prepare-root.service`), (2) have genesis embed it directly into `PrefixTree`
  before `commit_tree()` — same mechanism `EmbedDatabaseStage` already uses to insert the database
  file, not a new one — plus create its `*.wants/` enablement symlink since there's no live systemd
  to `systemctl enable` against on an unbooted target. Still unresolved: whether upac ships/packages
  the `composefs-setup-root` binary itself or expects it to already exist on the source distro (same
  open question as systemd-boot/rEFInd's own binaries).
- **UKI A/B confirm-boot service**: after a successful boot, something needs to confirm once, swap
  `to`↔`from`, and set the normal persistent boot order. Nothing calls `Booter::confirm_boot`
  anywhere yet. Open design question: how does the service know it just booted the `to` slot
  specifically (from `/proc/cmdline`? from the loaded UKI's own filename?) — needs deciding before
  writing any code.
