# SolidOS Panes

An unofficial draft describing how SolidOS shows a Solid pod through panes: what a pane is, how one is chosen, how panes are registered, and how to write one. Written down from the running code and the SolidOS team's notes.

Read it: https://melvincarvalho.github.io/solidos-panes/

Corrections welcome as issues or pull requests. The document may move under the SolidOS organisation later; this repository will then redirect.

## Files

- `source.html`: the ReSpec source. Edit this one.
- `index.html`: the published page, a static build of `source.html` so it reads without any script. Rebuild after editing the source:

```sh
chromium --headless=new --virtual-time-budget=40000 --dump-dom http://localhost:8010/specs/solidos-panes/source.html > index.html
```
