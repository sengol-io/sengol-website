# Self-hosted fonts

Both families are redistributed here under the **SIL Open Font License 1.1**,
which requires the copyright and licence notice to accompany any copy of the
font software. Those notices are alongside the files:

| Family | Files | Licence | Notice |
|---|---|---|---|
| Inter | `inter-latin*.woff2` | SIL OFL 1.1 | `Inter-LICENSE.txt` |
| JetBrains Mono | `jetbrains-mono-latin*.woff2` | SIL OFL 1.1 | `JetBrainsMono-LICENSE.txt` |

Copyright (c) 2016 The Inter Project Authors — https://github.com/rsms/inter
Copyright 2020 The JetBrains Mono Project Authors — https://github.com/JetBrains/JetBrainsMono

## Regenerating

Filenames carry a sha256 prefix so `_headers` can cache them `immutable` for a
year without ever serving a stale file. If you replace a font, recompute the
hash and update the `@font-face` `src` and the `<link rel="preload">` in
`index.html` to match.

Both were fetched as **variable** woff2 (one file per family per subset covers
every weight), latin and latin-ext only. `unicode-range` means latin-ext is
downloaded only if a visitor actually renders a character in that range.
