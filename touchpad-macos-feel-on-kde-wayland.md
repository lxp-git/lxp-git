# Making the Touchpad Feel Like macOS on KDE Plasma (Wayland)

> Arch · Plasma/KWin 6.7.1 · Wayland. My actual setup — the config, scripts, and values I run.
> Everything here lives inside KWin (stable Plasma exposes no GUI for it), so each piece must be **rebuilt after every KWin update**.

## 1. Gestures — InputActions

Two KWin built-ins have no setting to tune: 3-finger horizontal = switch desktop (so a fast 3-finger drag mis-switches), and 4-finger switch that flies through several desktops at once. [InputActions](https://github.com/taj-ny/InputActions) is a KWin input-filter plugin whose triggers *swallow* the built-in gesture, so I block the 3-finger one (drag stays, done by patched libinput) and replace the 4-finger one with one-shot-per-swipe.

**Install** (AUR, no helper; `cli11` is the only extra build dep):

```bash
sudo pacman -S --needed cli11
for p in inputactions-ctl inputactions-kwin; do
  git clone https://aur.archlinux.org/$p.git && (cd $p && makepkg -si)
done
```

**Enable:** System Settings → Desktop Effects → InputActions → tick → **Apply**. It must be *saved* or it silently won't load — verify with `qdbus6 org.kde.KWin /Effects org.kde.kwin.Effects.isEffectLoaded kwin_gestures` → `true`.

**My `~/.config/inputactions/config.yaml`** (auto-reloads on save):

```yaml
touchpad:
  gestures:
    # Block KWin's built-in 3-finger desktop switch (3 fingers = drag only).
    # The libinput drag is button-down+motion, not a swipe, so it's unaffected.
    - type: swipe
      fingers: 3
      direction: left_right
      threshold: 5
      actions: []
    # 4-finger switch -> one desktop per swipe. Swap the two shortcuts if reversed.
    - type: swipe
      fingers: 4
      direction: left
      threshold: 20
      actions:
        - on: begin
          plasma_shortcut: kwin,Switch One Desktop to the Right
    - type: swipe
      fingers: 4
      direction: right
      threshold: 20
      actions:
        - on: begin
          plasma_shortcut: kwin,Switch One Desktop to the Left
```

## 2. Pointer acceleration curve + scroll

Stable KWin/Wayland has no custom pointer curve; native support ([kwin !6937](https://invent.kde.org/plasma/kwin/-/merge_requests/6937)) is still unmerged. I backport that one MR onto the current kwin and rebuild. A `pacman -Syu` reverts kwin to stock and breaks the curve, so this is a one-command redo:

```bash
#!/usr/bin/env bash
# rebuild-kwin-accel.sh — rebuild kwin with kwin!6937 (custom accel-curve D-Bus support).
# Fetches the matching official PKGBUILD, injects the patch, builds.
#   ./rebuild-kwin-accel.sh           # auto-detect installed kwin version
#   ./rebuild-kwin-accel.sh 6.7.2 1   # explicit version + pkgrel
set -euo pipefail
BASE="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PATCH="$BASE/patches/kwin-mr6937-on-6.7.0.patch"   # only touches device.{cpp,h}; applies across 6.7.x
[ -f "$PATCH" ] || { echo "missing patch: $PATCH"; exit 1; }

VER="${1:-$(pacman -Q kwin | awk '{print $2}' | cut -d- -f1)}"
REL="${2:-1}"
BUILD="$BASE/build/kwin-${VER}"; rm -rf "$BUILD"; mkdir -p "$BUILD"; cd "$BUILD"

curl -fsSL "https://gitlab.archlinux.org/archlinux/packaging/packages/kwin/-/raw/${VER}-${REL}/PKGBUILD" \
  -o PKGBUILD || { echo "fetch failed — check tag at .../kwin/-/tags"; exit 1; }

# Don't clobber an official prepare() (means it already carries its own patches).
grep -q '^prepare()' PKGBUILD && { echo "official PKGBUILD has prepare(); merge by hand"; exit 2; }

cp "$PATCH" kwin-6937.patch
cat >> PKGBUILD <<'EOF'

# --- LOCAL: backport kwin!6937 custom acceleration profile ---
source+=(kwin-6937.patch)
sha256sums+=('SKIP')
prepare() { cd "$pkgname-$pkgver"; patch -Np1 -i "$srcdir/kwin-6937.patch"; }
EOF

export MAKEFLAGS="-j$(nproc)" CMAKE_BUILD_PARALLEL_LEVEL="$(nproc)"   # else makepkg builds single-core
makepkg -C -f --nocheck --skippgpcheck --noconfirm

echo "Install, relog (Wayland), then apply the curve:"
echo "  sudo pacman -U $BUILD/kwin-${VER}-${REL}-x86_64.pkg.tar.zst"
echo "  source $BASE/trackpad-curves.sh && tpcurve macos && tpscroll 0.3"
```

The curve itself is set over the KWin session D-Bus (no root; applies live and persists to `~/.config/kcminputrc`, keyed per device). My values are a fufexan-style curve capped at ~1.9× on fast flicks (macOS-style accel that doesn't run away), and `scrollFactor 0.3` (default `1` is too fast for a hi-res trackpad, and the Touchpad-KCM slider doesn't affect this device):

```bash
#!/usr/bin/env bash
# trackpad-curves.sh — source it, then: tpcurve macos && tpscroll 0.3
TP_NODES=()
TP_MACOS="0.5:0,0.05,0.11,0.195,0.32,0.494,0.735,1.06,1.488,2.05,2.775,3.69,4.83,6.235,7.95,9.994,12.4,15.2,17.1,18.05,19,19.95"

# Find the live Magic Trackpad node(s); the number changes across USB/BT reconnects.
tp_find() {
  TP_NODES=( $(busctl --user tree org.kde.KWin \
    | grep -oE '/org/kde/KWin/InputDevice/event[0-9]+' \
    | while read -r p; do
        n=$(busctl --user get-property org.kde.KWin "$p" org.kde.KWin.InputDevice name 2>/dev/null)
        case "$n" in *Magic*Trackpad*) echo "${p##*/event}" ;; esac
      done) )
  echo "TP_NODES=${TP_NODES[*]}"
}

# tpcurve macos   (or pass a raw "step:p0,p1,..." string)
tpcurve() {
  local c="${1:-macos}" e p
  [ "$c" = macos ] && c="$TP_MACOS"
  [ ${#TP_NODES[@]} -eq 0 ] && tp_find >/dev/null
  for e in "${TP_NODES[@]}"; do
    p=/org/kde/KWin/InputDevice/event$e
    busctl --user set-property org.kde.KWin "$p" org.kde.KWin.InputDevice pointerAccelerationCustomPointsMotion s "$c"
    busctl --user set-property org.kde.KWin "$p" org.kde.KWin.InputDevice pointerAccelerationProfileCustom b true
  done
  echo "-> curve $c"
}

# tpscroll 0.3
tpscroll() {
  local e
  [ ${#TP_NODES[@]} -eq 0 ] && tp_find >/dev/null
  for e in "${TP_NODES[@]}"; do
    busctl --user set-property org.kde.KWin /org/kde/KWin/InputDevice/event$e org.kde.KWin.InputDevice scrollFactor d "$1"
  done
  echo "-> scrollFactor=$1"
}

tp_find >/dev/null 2>&1 || true
```

Settings are keyed per device id, so USB `[1452]` and Bluetooth `[76]` don't share — `tpcurve macos && tpscroll 0.3` again after switching transport.

## Maintenance

After every KWin update, rebuild both: `inputactions-kwin` (`makepkg -si`, restart KWin) and `./rebuild-kwin-accel.sh`. **Don't `IgnorePkg = kwin`** — it's tightly coupled to the rest of Plasma, and pinning it risks `kwin_wayland` failing to start (no desktop).

## References

- InputActions: <https://github.com/taj-ny/InputActions> · AUR `inputactions-kwin`
- Patched libinput (3-finger drag): <https://github.com/marsqing/libinput-three-finger-drag>
- Custom-curve MR: kwin [!6937](https://invent.kde.org/plasma/kwin/-/merge_requests/6937)
