# ONLYOFFICE AI Plugin Fix — writeMacro no-ops on web Document Server

Patched build of the official ONLYOFFICE AI plugin fixing **PR #596**  
`fix(ai-plugin): writeMacro family no-ops on web Document Server` — commit `f27b8f4`.

## The bug
On web Document Server, `writeMacro` (and `generateDocx` / `changeParagraphStyle` same pattern) silently fails:

* **WRITE** requests report success but never modify the document
* **READ** requests return no data

**Root cause — `sdkjs-plugins/content/ai/scripts/helpers/helpers.js:1192`:**

```js
// before (broken)
Asc.Editor.callCommand(function(){ var __result = eval(Asc.scope.macroCode) })
```

`Asc.plugin.callCommand` serializes via `toString()` and the editor core executes through `_safe_eval_closure` (`sdk-all-min.js`) which wraps with `Function.apply(... return eval(...))` and shadows `window` — sandbox. Nested `eval()` inside body returns `undefined` and never runs.

Verified: `new Function(code)` and `executeCommand("command",code)` mutate; nested `eval(code)` and nested `new Function(code)()` do not.

## The fix
* ** `sdkjs-plugins/content/ai/scripts/helpers/helpers.js:1192,4193,8115` (word/slide/cell)** — dual-form `new Function` wrapper:
  * Expression form: `new Function("try { var __r = (" + code + "); ... }")` captures IIFE/single expression
  * If `new Function` throws (multi-statement or has `return`), fall back to statement form: `new Function("try { var __result = (function(){ "+code+" }).call(this); ... }")`
  * Preserves `try/catch` + `onlyoffice_id_result` / `onlyoffice_id_error_message` contract
* **`sdkjs-plugins/content/ai/scripts/helpers/helperFuncs.js:104` + `helpers.js`** — align prompts/tool descriptions to new model: READ examples use explicit `return` (`return Api.GetDocument().GetElement(0).GetText()`), add READ/WRITE/CHAT decision rule, GENERATIVE rule (ask first on ambiguous "write content"), SELF-CHECK rule (verify top-level `return` on READ).

**Source fix for reproducibility — `sdkjs-plugins/content/ai/.dev/helpers/{word,slide,cell}/write-macro.js:39,49,142`:**

Patched ` .dev` sources then regenerated via `helpers.py:1`:

```
cd sdkjs-plugins/content/ai/.dev/helpers && python3 helpers.py
# regenerates ../../scripts/helpers/helpers.js
```

Previously `deploy/ai.plugin` was stale and `helpers.py` would revert the patch — now synced.

No core rebuild needed. Works with existing ONLYOFFICE 9.4.0.129 + patched plugin.

## Build from PR branch

```bash
git clone https://github.com/ONLYOFFICE/onlyoffice.github.io.git
cd onlyoffice.github.io
git fetch origin pull/596/head:fix-ai-writemacro
git checkout fix-ai-writemacro
# optional: sync .dev sources if testing regeneration
# then in sdkjs-plugins/content/ai/.dev/helpers: python3 helpers.py

# package (zip not always present on Arch)
cd sdkjs-plugins/content/ai
python3 -c "import zipfile,pathlib; src=pathlib.Path('.'); out=pathlib.Path('/tmp/ai-fix.plugin'); ..."
# or: zip -r /tmp/ai.plugin config.json *.html scripts resources translations vendor components
```

Prebuilt artifact in this repo: [`ai.plugin`](./ai.plugin) (2.1 MB, 663 files, `grep -c "new Function" => 6`).

## Install (Arch)

**Plugin Manager (recommended):** ONLYOFFICE -> `Plugins` -> `Settings` (gear) -> `Install plugin from file` -> select `ai.plugin` -> Enable -> Restart.

**Manual DesktopEditors:**
```bash
mkdir -p ~/.local/share/onlyoffice/desktopeditors/sdkjs-plugins/
unzip -o ai.plugin -d ~/.local/share/onlyoffice/desktopeditors/sdkjs-plugins/ai/
# system: /opt/onlyoffice/desktopeditors/sdkjs-plugins/
```

**Document Server (web):**
```bash
/var/www/onlyoffice/documentserver/sdkjs-plugins/
# docker: /usr/share/documentserver/sdkjs-plugins/
sudo systemctl restart ds-converter ds-docservice
```

## Test

```js
// WRITE — should now mutate (pre-patch no-op)
var oDoc = Api.GetDocument(); var oPar = Api.CreateParagraph(); oPar.AddText("hello patched"); oDoc.Push(oPar);

// READ — must use explicit return (pre-patch returned nothing)
return Api.GetDocument().GetElementsCount()
return Api.GetActiveSheet().GetRange("A1").GetValue()
return Api.GetPresentation().GetSlidesCount()
```

Upstream: https://github.com/ONLYOFFICE/onlyoffice.github.io/pull/596

License: AGPL-3.0 (same as ONLYOFFICE plugins).
