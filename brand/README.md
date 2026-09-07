# Brand assets

Every icon on this site is cut from the supplied Sengol logo package. Nothing
here is hand-drawn — if a mark needs to change, change it in the package and
re-cut, do not edit the derived files.

## What ships, and where it came from

| File in web root | Source in the logo package |
|---|---|
| `favicon.ico` (16/32/48) | `04-favicon/favicon.ico` |
| `favicon-32.png`, `favicon-16.png` | `04-favicon/favicon-{32,16}.png` |
| `apple-touch-icon.png` | `03-mobile-app-icons/app-icon-180.png` |
| `icon-192.png`, `icon-512.png` | `03-mobile-app-icons/app-icon-{192,512}.png` |
| `brand-mark.png` | symbol cropped out of `01-master/sengol-logo-light.png`, trimmed to its alpha bounds, 6% padding, resized to 120×120 |

`brand-mark.png` is the light (white) symbol, for dark backgrounds only. The
package also ships a dark symbol for light backgrounds — this site is dark
throughout, so it is not used here. Anywhere the mark lands on white (README,
PyPI, slides) needs `01-master/sengol-logo-dark.png` instead.

## Regenerating og-image.png

`og-card.html` is the source for the 1200×630 social card. It uses the same
tokens, fonts and copy as the page, so a hero rewrite means re-rendering the
card too.

```
chromium --headless --disable-gpu --hide-scrollbars \
  --force-device-scale-factor=1 --window-size=1200,630 \
  --screenshot=og-image.png brand/og-card.html
```

Then optimize losslessly. Do **not** quantize to a 256-colour palette: the
background is a navy gradient, and every palette method tried shifted it
visibly toward purple (max channel deviation 36–72) for a ~250 KB saving that
is not worth a wrong brand colour on the most-shared asset on the site.

## Known limits of the source art

- **The package is raster only** — no SVG, no editable vector. Scaling past
  1024 px will soften. A vector would have to be traced, which is a redraw, not
  a conversion.
- **The mark does not survive 16 px.** At favicon size the chain links merge
  into a grey blob with an amber pixel at the centre; it reads as "dark tile,
  amber core" and nothing more. That is what `favicon.ico` ships and it is
  acceptable, but a purpose-drawn 16 px cut — shield silhouette plus keyhole,
  links dropped — would read better if anyone wants to make one.
- **The logo amber is `#F7C666`; the site gold is `#C9A84C`.** They are not the
  same colour and the page has not been reconciled to the logo. Deliberate, and
  open — see the PR discussion.
