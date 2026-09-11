# firefox-android-overscroll

Working notes for a proposed Gecko contribution: give Firefox for Android the
elastic (rubber-band) overscroll animation that desktop Firefox already ships,
instead of the legacy Android edge glow it uses today.

**Status:** research done, nothing implemented yet. Current direction: match Android 12's
stretch effect (as Chrome does), implemented in Gecko's compositor rather than delegated
to the platform.

See [RESEARCH.md](RESEARCH.md) for the full findings — what the code does today on
each platform, why the Android path is broken on Android 12+, confirmation that
nobody else is working on this, and a recommended plan.
