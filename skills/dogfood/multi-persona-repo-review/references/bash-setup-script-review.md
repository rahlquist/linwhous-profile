# Bash system-setup / provisioning repo review checklist

Static review recipes for setup-script repos (install-*.sh, provisioning, dotfiles installers, run-all orchestrators). Use when shellcheck is unavailable or as a supplement. Every finding needs file:line.

## Bash traps that actually bite

- **`raise SystemExit('STRING')` in an embedded Python heredoc exits with code 1, not the string.** If the bash side does `case "$rc" in 3) ...duplicate... ;;` the branch is dead — idempotent re-runs look like hard failures. Verify with a one-liner subprocess probe before reporting. (Seen: fetch-model.sh-style config inserters.)
- **`CURRENT_USER="${SUDO_USER:-$USER}"` silently becomes `root` when the script is run as root directly** (sudo -v succeeds for root). Downstream: adds root to docker/libvirt groups instead of the human, writes systemd units with `User=root`, chowns user dirs to root. Flag any script that both `require_root_or_sudo` and uses SUDO_USER without refusing `EUID==0`.
- **`cmd | tee log` under `set -o pipefail`** propagates the left side's failure — good — but orchestrators that then `exit 1` mid-`pacman -Syu`/`apt upgrade` leave a half-upgraded system with no trap. Flag full-upgrade-in-pipeline without a preflight confirm.
- **Destructive networking without verification:** deleting the live NM connection (`nmcli con delete "$EXISTING"`) or writing networkd files *before* confirming the replacement has an IP = SSH lockout. Require: detect who manages the NIC (`nmcli device status` vs `networkctl`), verify new link has an address before removing old, warn "don't run over SSH".
- **`|| true` after `sudo nft ...` + `systemctl enable nftables`** persists a possibly-empty/partial ruleset to /etc/nftables.conf and enables the service — a boot-time lockout landmine when another firewall tool (ufw/iptables-nft) is mid-config. Persist only what the script itself created.
- **Re-run traps (idempotency):** `find ... -newer marker`, `mktemp` names with `$(date +%H%M%S)` collisions when run twice in the same second as different users (root-owned tee output → later user run gets Permission denied), `grep -o`-scraped defaults from configs whose *comments* contain example values (comment lines must be stripped before counting/scraping).
- **`2>/dev/null || warn "skipped"` install loops** hide real resolution errors; later `systemctl enable --now <svc>` fails opaquely. Distinguish "package not found" from real errors; fail hard when the core package didn't land.
- **`groups "$CURRENT_USER" | grep -q docker`** — fine, but adding a user to `docker` group with no security caveat ≈ passwordless root; note it.

## Python helpers in these repos

- CSV/summary writers that `ORDER BY id DESC` + `setdefault(test_name, value)` across *all* historical rows mix tonight's partial run with last night's results in one row. Flag unless filtered by current run id/epoch.
- Lockfile + flock watchers: check the in-flight-subprocess grace period (`timeout=budget+120`) vs the parent's own budget — mismatch = double-kill or orphaned GPU-hogging bench processes.
- SQLite "changed file" detection via (size, mtime) is fine; but re-marking `status='pending'` on mtime change must not delete old runs (append-only is correct).

## Doc-vs-code matrix (always build one)

README claims vs what scripts actually do — hardware targets, backend flags (CUDA vs ROCm), step descriptions in orchestrator tables, commit-message templates. A README saying "CUDA sm_120" while the script builds `-DGGML_HIP=ON -DAMDGPU_TARGETS=gfx1201` is a HIGH doc-vs-code mismatch that sends users down a broken path.

## Report shape

Severity-ranked, file:line for everything, then a "what works / verified clean" note (e.g. a bounds-checked GGUF parser) so the report reads as evidence not vibes. Verify cheap claims yourself with a subprocess probe before ranking CRITICAL.
