# proton-pass-biometrics

Source-built Arch package of [Proton Pass](https://proton.me/pass) desktop
(`ProtonMail/WebClients`, tag `proton-pass@1.42.0`) with Linux fingerprint
unlock added. The fork is carried entirely by two patches on top of the
upstream tarball; nothing else is vendored.

Replaces `proton-pass-bin` (same binary name `proton-pass`, same desktop
entry, bundled Electron — no system `electron` dependency).

## How it works

Upstream ships the full biometrics machinery for Windows and macOS and stubs
it out on Linux in three places. The patches replace those stubs:

- `0001-rust-linux-biometrics-backend.patch` — real implementation for
  `applications/pass-desktop/native/shared`:
  - `can_check_presence`: true iff fprintd (`net.reactivated.Fprint`) has a
    device with at least one enrolled finger for the current user **and**
    `pkcheck` is on PATH. Any failure is reported as `false`, never an error
    (upstream semantics: false hides the option).
  - `check_presence`: `pkcheck --allow-user-interaction --action-id
    ch.proton.pass.unlock --process <pid of the Electron main process>`.
    The action is `auth_self` for active local sessions, so polkit asks PAM,
    and `/etc/pam.d/polkit-1` puts `pam_fprintd.so` first with a password
    fallback — the desktop agent (Omarchy) shows the fingerprint prompt.
  - `get_secret`/`set_secret`/`delete_secret`: Secret Service (gnome-keyring
    login collection) via zbus, matching macOS keychain / Windows DPAPI
    semantics upstream uses. No app-level PIN prompt: the polkit/PAM path is
    the authentication, and it already has a password fallback.
- `0002-ts-enable-linux-biometrics.patch` — the renderer gates:
  `supportsBiometrics` in `src/app/App.tsx`, the platform factory in
  `src/lib/biometrics/index.ts` (new `biometrics.linux.ts`, same shape as
  the Windows factory), and the `LockSetup.tsx` settings toggle.

The vault unlock key (`offlineKey_biometrics::<localID>` attribute `key`)
is stored in the keyring when the biometrics lock is enabled; unlocking
verifies presence through polkit and decrypts with that key.

## Layout

```
PKGBUILD
proton-pass.desktop             desktop entry (same as proton-pass-bin)
ch.proton.pass.unlock.policy     polkit action, installed to /usr/share/polkit-1/actions
0001-rust-linux-biometrics-backend.patch
0002-ts-enable-linux-biometrics.patch
```

## Build & install

```
makepkg -sf
sudo pacman -U proton-pass-biometrics-1.42.0-1-x86_64.pkg.tar.zst
```

pacman will replace `proton-pass-bin` (`conflicts` + `provides`). Build
requirements: `nodejs-lts-jod` (Node 24 — **required**, see below) and
`rust`; pnpm 12.8.1 is fetched automatically by corepack from the tree's
`packageManager` field. Runtime: `fprintd` + `gnome-keyring` + `polkit`,
with at least one enrolled finger (`fprintd-enroll`).

> Node ≥25 must not drive the build: its zip stack hangs silently inside
> electron-forge's extraction, so the build "succeeds" without ever
> producing `out/`. If your default `node` is newer, prefix makepkg with
> the Node 24 one, e.g. `mise exec node@24 -- makepkg` or use the
> system `nodejs-lts-jod`.

## Assumptions

- The gnome-keyring login collection is unlocked at session start (PAM).
  If it is not, `get_secret` fails and the app falls back to password
  unlock — see `gnome-keyring-pam` in `/etc/pam.d/system-auth`.
- The fingerprint prompt comes from the session's polkit authentication
  agent; without one, `pkcheck --allow-user-interaction` cannot prompt and
  verification fails closed (password unlock still works).

## Remote builds on daemon

The heavy compile (cargo release with LTO, webpack, forge packaging) can
run on the `daemon` host over the tailnet instead of the workstation:

```
# one-time on daemon (x86_64 required): node 24 LTS (nodejs-lts-jod), rust
rsync -a --exclude=src --exclude=pkg --exclude='*.tar.zst' ./ daemon:proton-pass-biometrics/
ssh daemon 'cd proton-pass-biometrics && makepkg -sf'   # SOURCES via ~/.cache, no sudo if makedeps present
scp daemon:proton-pass-biometrics/proton-pass-biometrics-*-x86_64.pkg.tar.zst .
sudo pacman -U proton-pass-biometrics-*-x86_64.pkg.tar.zst
```

daemon's Tailscale key expiry must be disabled (`tailscale up --advertise-tags`
or key expiry off in the admin console) for this to be reliable; it was
offline with an expired node key when this repo was first built.

The package produced on daemon is identical: PKGBUILD only consumes the
upstream tag tarball plus the repo files, and `sha512sums` pins both.