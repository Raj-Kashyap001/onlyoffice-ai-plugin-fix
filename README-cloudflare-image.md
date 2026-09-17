# Cloudflare Image Generator – Standalone ONLYOFFICE Plugin

Independent plugin for **ONLYOFFICE Docs / Desktop Editors** that generates images via **Cloudflare Workers AI** (`@cf/black-forest-labs/flux-1-schnell`) using **Account ID + API Token** (no Worker needed).

## Features
- Works in **Word, Spreadsheet, Presentation** (insert at cursor/current cell/slide)
- Direct Cloudflare API: `POST https://api.cloudflare.com/client/v4/accounts/{ACCOUNT_ID}/ai/run/@cf/black-forest-labs/flux-1-schnell`
- Also supports Worker proxy via `Worker URL` field (optional)
- CORS-safe on Desktop (`AscSimpleRequest` / `fetchExternal`) and Web
- Settings persisted in `localStorage`
- Preview + Insert

## Install
1. Download `cloudflare-image.plugin` (or `ai.plugin` for the patched AI plugin)
2. ONLYOFFICE → Plugins → Settings (gear) → Install plugin from file → select `cloudflare-image.plugin` → Enable
3. Restart Editors

**Manual:**
```bash
mkdir -p ~/.local/share/onlyoffice/desktopeditors/sdkjs-plugins/
unzip -o cloudflare-image.plugin -d ~/.local/share/onlyoffice/desktopeditors/sdkjs-plugins/cloudflare-image/
```

## Configure
Open plugin (Plugins → Cloudflare Image Generator):
- **Prompt** – description (e.g. `A realistic red soccer ball on green grass, centered, soft studio lighting, highly detailed, square composition`)
- **Width/Height** – mm, **Style** – realistic/cartoon/etc.
- **Settings** → `Account ID` (32 hex, Cloudflare dashboard → Workers & Pages → Overview → right sidebar) + `API Token` (My Profile → API Tokens → Create with Workers AI) – saved locally.  
  Alternatively set `Worker URL` (`https://your-worker.workers.dev/generate`) to keep token server-side.

## Usage
1. Enter prompt → **Generate image** → Preview
2. **Insert into document** → inserts at cursor (Word: new paragraph with image, Cell: at active cell, Slide: centered on current slide)

## Build from source
```bash
git clone https://github.com/ONLYOFFICE/onlyoffice.github.io.git
cd onlyoffice.github.io
# new plugin is sdkjs-plugins/content/cloudflare-image
cd sdkjs-plugins/content/cloudflare-image
python3 -c "import zipfile,pathlib; src=pathlib.Path('.'); out=pathlib.Path('/tmp/cloudflare-image.plugin'); import fnmatch; ..."
```

## License
AGPL-3.0 (same as ONLYOFFICE plugins). Model: Cloudflare Workers AI FLUX.1 Schnell – https://developers.cloudflare.com/workers-ai/models/flux-1-schnell/
