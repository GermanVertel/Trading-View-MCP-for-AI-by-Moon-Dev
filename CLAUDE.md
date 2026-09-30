# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

An MCP server (stdio, FastMCP) that drives a real, logged-in TradingView chart in Chrome over the Chrome DevTools Protocol: write Pine Script, compile-check it, paste it into the Pine Editor, add it to the chart, read results back. There is no official TradingView API for this; everything goes through the live page DOM.

## Commands

```bash
pip install -r requirements.txt          # or: pip install -e '.[dev]' for pytest
python -m tradingview_mcp.chrome_launcher # launch dedicated Chrome profile with CDP on :9222 (log in once)
python -m tradingview_mcp.server          # run the MCP server (stdio)
python mount_pine.py examples/<file>.pine # one-shot: validate -> write -> add to chart -> legend check -> screenshot
python mount_pine.py <file.pine> --shot out.png --keep-editor

pytest tests/test_smoke.py                # no Chrome needed; tool registry + selector module
pytest tests/test_smoke.py::test_server_registers_expected_tools
TV_MCP_CDP_PORT=9333 python tests/test_vwap_star_live.py   # needs a live logged-in TradingView tab
```

`tests/test_vwap_star_logic.py` reads OHLCV from a parent trading-bots repo and skips when that data is absent. There is no linter configured.

## Architecture

Two channels, deliberately separate:

- **`pine_facade.py`** — browser-free compile check via HTTPS POST to `pine-facade.tradingview.com/.../translate_light/`. Returns structured errors with line/column. Needs browser-like `Origin`/`Referer`/`User-Agent` headers or it 403s. Always validate here before touching the browser.
- **CDP** for everything that touches the chart:
  - `chrome_launcher.py` — launches Chrome with a dedicated profile (`~/.tradingview_mcp_chrome`) and remote debugging; no-op if already running.
  - `cdp_client.py` — raw websocket CDP client (`send`, `eval_js`, `press_key`, `type_text`, `screenshot`). `find_tradingview_tab` prefers tabs carrying the MCP marker, then `/chart` URLs. It also detects the "Moon Dev Code App" (Electron host) via a control endpoint (`MOONDEV_CTRL`, the app's `mcp-endpoint.json`, or `127.0.0.1:8765`); when present it does not launch its own Chrome and asks the app to focus the tab. Auto-accepts `javascriptDialogOpening` (TradingView's `beforeunload` would otherwise freeze the page on reload).
  - `tv_selectors.py` — every DOM selector in one file. When TradingView ships a UI change and something breaks, look here first; update `SELECTORS_VERIFIED_ON` when re-verifying.
- **`tools/`** — `chart.py`, `indicators.py`, `pine.py`, each exposing `register(mcp)`; wired up in `server.py:build_server`. Tool functions are decorated with `@with_cdp("tv_...")` from `tools/_helpers.py`, which injects a connected `CDPClient` as the first arg (hidden from the MCP schema), catches exceptions into `{"status": "error", "error": ...}` dicts, and logs to JSONL when `TV_MCP_LOG_FILE` is set. Tools return dicts with a `status` key rather than raising.
- **`run_*.py`** in the repo root are ad-hoc live drivers that import the private `_` helpers from `tools/pine.py` directly.

When adding a tool, register it in the relevant module's `register()` and add its name to the expected set in `tests/test_smoke.py`.

## TradingView/Monaco gotchas (measured, not guessed)

These drive much of the code in `tools/pine.py`; don't "simplify" them away:

- Non-text key events must include `code`, `windowsVirtualKeyCode`, and `nativeVirtualKeyCode` or Monaco ignores them.
- A synthetic `ClipboardEvent` paste updates Monaco's model but TradingView's change tracking never fires, so "Add to chart" stays disabled. One real key event (`_nudge_dirty`) arms it.
- Synthetic `.click()` does not reach some TradingView handlers (e.g. pane selection); use `Input.dispatchMouseEvent` at real coordinates.
- Monaco's hidden textarea mirrors only ~200 chars, so long scripts can't be read back. Verify writes by: buffer empty before paste, and caret line after paste within `abs(got - expected) <= 1` (not `>=`, which lets stale code above survive).
- The Pine console is append-only: snapshot the line count before compiling and read only new lines for errors.
- Legend `data-name` attributes are gone; the legend uses hashed CSS module classes, so match stable prefixes (`[class*="sources-"]`, `[class*="titlesWrapper"]`).
- TradingView allows one active session per account; another open TradingView tab/window triggers "Session disconnected".
- Electron `<webview>` hosts: `Page.captureScreenshot` hangs and the Pine Editor may not mount unless the webview has real size (~1000px+ wide) and focus.

## Config

Optional `.env` (see `.env.example`): `TV_MCP_CDP_PORT` (9222), `TV_MCP_CHROME_PROFILE_DIR`, `TV_MCP_START_URL`, `TV_MCP_CHROME_BINARY`, `TV_MCP_SKIP_CHROME_LAUNCH`, `TV_MCP_LOG_FILE`.

## Conventions

Code, log messages, and docstrings carry a "🌙 Moon Dev" voice/prefix; keep it consistent with surrounding code. Example Pine scripts in `examples/` are expected to compile.
