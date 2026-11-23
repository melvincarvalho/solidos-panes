---
name: solidos-panes
description: Build, test, load and list a SolidOS pane (one JavaScript module that draws a kind of data in the SolidOS data browser), or a Solid app that a person's tool index offers as "Open with". Use when asked to make a new pane, a new app for Solid data, or to make one discoverable through a tool index or the store.
---

# Building panes and apps for SolidOS

This is the short, practical guide for a person or an LLM arriving cold. The full specification is
[SolidOS Panes](https://melvincarvalho.github.io/solidos-panes/) (this repository); each section below links into it
for detail.

## The idea in five lines

1. A **pane** is one JavaScript file that draws one kind of thing (a note, a chess game, a poll) inside the
   SolidOS data browser. ([§4](https://melvincarvalho.github.io/solidos-panes/#the-pane))
2. An **app** is a web page that opens a person's data in its own tab, through a link pattern such as
   `https://example.org/viewer?url={url}`.
3. A person's pod holds a **tool index**: their own list of panes and apps.
   ([§7.4.1](https://melvincarvalho.github.io/solidos-panes/#dynamic-tool-index))
4. The **SolidOS Browser** reads that index after login, loads the panes, and offers the apps as "Open with" links.
   Nothing is rebuilt or released to add one.
5. The **store** is a public catalogue in the same format, so adding a tool is copying its entry into your index.

Try it first: open the [SolidOS Browser](https://solidos.org/docs/browser/), and a sample chess game with the chess pane:
<https://solidos.org/docs/browser/?uri=https%3A%2F%2Fsolid-panes.github.io%2Fchess%2Fgames%2Fopera.ttl>

## Pane or app?

| Build a pane when | Build an app when |
|---|---|
| It shows or edits one kind of thing, in place, next to SolidOS's own views | It is a full program with its own layout and navigation |
| It should share the logged-in session and the data already loaded | It can log in on its own and load its own data |
| One file is enough | It needs a build step, a framework, or many files |

A pane can be small: a button, a theme change, a new view of a file type. A whole suite can also be one pane that
loads others (see Commons below).

## A pane in 30 lines

```js
const pane = {
  name: 'markdown-reader',                   // unique; a second pane with the same name is ignored
  icon: 'data:image/svg+xml,...',            // a data: URI, so the pane stays one file
  label (subject, context) {                 // a short tab name if this pane fits the subject, else null
    return /\.md$/i.test(subject.uri) ? 'Notes' : null
  },
  shouldGetFocus (subject) {                 // optional: true to open first, ahead of the built-in panes
    return !!pane.label(subject)
  },
  render (subject, context) {                // return one DOM element; it may fill in later
    const dom = context.dom
    const store = context.session.store      // the rdflib store SolidOS shares
    const root = dom.createElement('div')
    root.className = 'mdr'
    const style = root.appendChild(dom.createElement('style'))
    style.textContent = CSS
    const text = root.appendChild(dom.createElement('pre'))
    text.textContent = 'Loading…'
    const doFetch = (store.fetcher && store.fetcher._fetch) || fetch   // the logged-in fetch
    doFetch(subject.uri, { headers: { Accept: 'text/markdown, text/plain' } })
      .then(r => r.text()).then(t => { text.textContent = t })
      .catch(e => { text.textContent = 'Could not load: ' + e.message })
    return root
  }
}
const CSS = `.mdr { padding: 16px; color: #1d1d1f; background: #fff } .mdr pre { white-space: pre-wrap }`
export default pane
```

The members, the context and how SolidOS picks a pane are in
[§4.1](https://melvincarvalho.github.io/solidos-panes/#required-members) to
[§5](https://melvincarvalho.github.io/solidos-panes/#selection).

Useful inside `render`:

- Data already loaded: `store.statementsMatching(s, p, o, subject.doc())`, `store.each(...)`, `store.any(...)`.
- Load a document: `store.fetcher.load(node).then(...)`.
- Go somewhere in place: `context.getOutliner(dom).GotoSubject(node, true, undefined, true, undefined)`, with a real
  `href` as fallback.
- Folders: `subject.uri` ends in `/`; children are `store.each(subject, $rdf.sym('http://www.w3.org/ns/ldp#contains'))`.
- `shouldGetFocus` is called with the subject only. To reach the store there, use the global `SolidLogic.store`.
- Match terms by local name where you can, so data in other vocabularies still works.

### Creating new things

A pane that can make a new instance (a new poll, a new address book) adds `mintClass` and
`mintNew(context, options)`, which returns `{ ...options, newInstance }`. The Add menu of a folder then offers it.
Details: [§8](https://melvincarvalho.github.io/solidos-panes/#creating).

## Rules that keep panes working together

- **One file, no imports, no dependencies.** It must load from any URL. (A pane may load more files itself at run
  time, but say so in its README.)
- **Build the DOM with `createElement` and `textContent`. Never `innerHTML`**, and never `eval` or `new Function`.
  Notes, messages and file names can carry hostile text.
- **All styles in one CSS string**, appended as a `<style>` inside the root, with **one class prefix** per pane.
- **Light theme.** It sits inside the SolidOS page.
- **Works at 400 px** wide with no sideways page scroll.
- **A read pane never writes.** A pane that writes says so in its page and README.
- **A pane runs as the logged-in person**: it can read and change whatever they can. Treat that seriously.
- Pick a licence. If you bundle GPL code (for example a chess engine), the pane is GPL.

More in [§9.2](https://melvincarvalho.github.io/solidos-panes/#rules-that-keep-panes-working-together).

## Loading a pane

| Where | How |
|---|---|
| By link, no server change | `https://solidos.org/docs/browser/?pane=<pane URL>&uri=<resource URL>` (both URL-encoded). The page asks once before running a pane from a site it has not allowed before. |
| In a person's tool index | Add a registration (below). It loads for them on every page, after login. |
| On a JavaScript Solid Server | `jss start ... --panes https://.../a-pane.js,https://.../b-pane.js` (JSS 0.0.227 or later) |
| In code, on any SolidOS page | `panes.register((await import(url)).default)` |

How loading works, and its security model: [§7.5](https://melvincarvalho.github.io/solidos-panes/#dynamic-loading),
[§7.6](https://melvincarvalho.github.io/solidos-panes/#dynamic-security).

## The tool index: discovery

The tool index uses the PodOS tools vocabulary, `https://pod-os.org/vocab/tools#`, so one file serves both PodOS and
SolidOS. The preferences file points to it:

```turtle
<WebID> tools:toolIndex <toolIndex.jsonld> .
```

A pane and an app:

```turtle
<> a tools:ToolIndex ; tools:registration <#chess>, <#umai> .

<#chess> a tools:Registration ; schema:name "Chess" ; ac:capability <#chess-cap> .
<#chess-cap> a ac:Capability ; ac:action odrl:display ; ac:invocation <#chess-inv> .
<#chess-inv> a panes:PaneModuleInvocation ; schema:url "https://solid-panes.github.io/chess/chess-pane.js" .

<#umai> a tools:AppRegistration ; schema:name "Umai" ; ac:capability <#umai-cap> .
<#umai-cap> a ac:Capability ; ac:action odrl:display ; ac:resourceType schema:Recipe ; ac:invocation <#umai-inv> .
<#umai-inv> a ac:UriTemplateInvocation ; hydra:template "https://umai.noeldemartin.com/viewer?url={url}" ;
    hydra:mapping [ hydra:variable "url" ; hydra:property ac:open ] .
```

Things to know:

- `panes:PaneModuleInvocation` is `https://mashlib.com/ns/panes#PaneModuleInvocation`, a **placeholder** until PodOS
  has a kind for a JavaScript module.
- To switch an entry off, remove the `tools:registration` link; the entry can stay in the file.
- App link patterns use plain `{variable}` replacement. Write `?uri={url}`, not RFC 6570 forms like `{?uri}`.
- An app with no `ac:resourceType` is offered for anything; one with a type only for things of that type.
- Panes are not chosen by the entry's type: every registered pane loads and decides with its own `label`.
- `.jsonld` is the safest format on a pod; `.ttl` also works.

Tools that work with it:

- [Tools app](https://solid-apps.github.io/tools/): finds or creates a person's tool index.
- [Tool index pane](https://github.com/solid-panes/tool-index): a dashboard that lists, switches on and off, and adds
  from the store.
- [Store](https://solid-panes.github.io/store/): the public catalogue, [`catalogue.jsonld`](https://solid-panes.github.io/store/catalogue.jsonld).

## Testing

Test in a real browser, headless, before saying a pane works:

1. It loads by link (`?pane=...&uri=...`) on the SolidOS Browser, and the root element renders.
2. No page errors in the console.
3. No sideways overflow at 1300 px and at 400 px: `document.documentElement.scrollWidth <= innerWidth`.
4. The content you expect is there.
5. Take a screenshot and look at it.

The store's automatic check is a good floor. It fails a pane that is not served over https as JavaScript, does not
parse, has static imports, uses `innerHTML` or `eval`, is not a pane, or scrolls sideways at 400 px:

```sh
git clone https://github.com/solid-panes/store && cd store
node tests/panes.mjs https://example.org/my-pane.js https://example.org/something-it-draws.ttl
```

It is a floor, not a review.

## Getting listed in the store

Open an issue or a pull request on [solid-panes/store](https://github.com/solid-panes/store) with the pane's
address, a sample it draws, and the output of the check above. The lists are at the top of `build.mjs`;
`node build.mjs` writes `catalogue.jsonld`. Entries are not signed yet; signing and ratings are planned.

## Publishing a pane

A pane in its own repository, served by GitHub Pages or any static host, with:

- the pane file;
- a sample data file or two it can draw;
- a `README.md` with a try-it link (`https://solidos.org/docs/browser/?pane=...&uri=...`);
- a small `index.html`, a `LICENSE`, and a share picture (`og.png`);
- `.nojekyll` on GitHub Pages.

Static hosts and long `Accept` headers: rdflib sends a long `Accept` header that GitHub Pages rejects on cross-site
reads. A pane that fetches for itself should send a short one, as in the example above.

## Examples to read

| Pane | What it shows | Try it |
|---|---|---|
| [notes](https://github.com/solid-panes/notes) | Markdown notes with links between them | <https://solid-panes.github.io/notes/> |
| [chess](https://github.com/solid-panes/chess) | A chess game stored as RDF; bundles GPL code | [opera game](https://solidos.org/docs/browser/?uri=https%3A%2F%2Fsolid-panes.github.io%2Fchess%2Fgames%2Fopera.ttl) |
| [poll](https://github.com/solid-panes/poll) | "Find a time": writes answers, creates new polls with `mintNew` | <https://solid-panes.github.io/poll/> |
| [tool-index](https://github.com/solid-panes/tool-index) | Edits a tool index | <https://github.com/solid-panes/tool-index> |
| [Commons](https://github.com/JavaScriptSolidServer/commons) | A Nextcloud-style suite: one shell pane loading many app panes | <https://jss.live/commons/> |

## Where to go next

- The full specification: <https://melvincarvalho.github.io/solidos-panes/>
- Open questions in the design: [§7.7](https://melvincarvalho.github.io/solidos-panes/#dynamic-open)
- Signing and trust for panes: [JavaScriptSolidServer#651](https://github.com/JavaScriptSolidServer/JavaScriptSolidServer/issues/651)
