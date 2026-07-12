# Moving From KDE to COSMIC for a Faster Desktop

> Arch Linux · July 2026

In the LLM era, any small utility I'm missing is just a prompt away — so the desktop itself is the thing worth optimizing. I switched from KDE Plasma to **COSMIC** (cosmic-comp):

- **Apps launch much faster** than on KDE.
- **Resizing an IntelliJ IDEA window stays smooth**, where KWin stuttered badly — a quick benchmark script confirmed it.

Then I ported the gadgets I actually rely on:

- **Three-finger drag** — via patched [libinput-three-finger-drag](https://github.com/marsqing/libinput-three-finger-drag).
- **"What's keeping my machine awake?"** — a panel applet I wrote that lists every app blocking sleep, with a keep-awake toggle: [cosmic-ext-applet-inhibit-status](https://github.com/lxp-git/cosmic-ext-applet-inhibit-status).
