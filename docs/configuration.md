# Configuration Reference

Everything the gate reads from disk lives in `<agentDir>/config/pi-verdict.json` (first run generates a template there; `PI_CODING_AGENT_DIR` overrides the location). This page is the full reference; the README keeps a quick-start subset.

## User rules

```json
{
  "allow": ["^ls\\b", "^git (status|log|diff)\\b"],
  "deny":  ["rm ", "docker ", "^/etc/"],
  "denyPaths": ["~/Documents/private", "~/work/company"],
  "ignoreTools": ["todo", "web_search"],
  "builtinDenyFloor": true,
  "classifierModel": null,
  "toggleShortcut": "ctrl+shift+a",
  "enabledByDefault": true,
  "tamperResponse": "warn"
}
```

- `allow`/`deny` are JS regex arrays; **`deny` wins over `allow`**, both beat the classifier. Match targets: bash/powershell = the full command string; file tools = the resolved absolute path; other tools (e.g. MCP) are not covered by rules and land in the gray zone.
- `denyPaths` are plain paths (not regexes) you declare **protected**: any tool call touching them — file tools via their path, bash via path tokens extracted from the command string — triggers a **terminal ask** you adjudicate (non-interactive sessions degrade to deny). Not affected by `builtinDenyFloor: false`.
  The classifier only ever learns that protected paths *exist*; the paths themselves never leave your machine, and a matched path shows **only** in the local confirm dialog.
- `ignoreTools` is a plain list of tool names **outside** the command/file families (`todo`, `web_search`, MCP/custom tools, …) that skip adjudication entirely — verdict allow, zero model calls. Entries naming covered tools (`bash`/`read`/`write`/`edit`/`grep`/`find`/`ls`/`powershell`) are inert: those stay governed by the deny floor and your allow/deny rules, and the self-protection layer runs before any passthrough. An exempted tool that touches paths also drops the classifier's `denyPaths` existence-hint vigilance (uncovered tools never hit the path extractor).
- `builtinDenyFloor: false` turns the built-in danger/path floor off entirely (risk accepted by you; the classifier and your rules remain — the self-protection layer always stays on).
- `classifierModel: "provider/model-id"` sets the classifier model (e.g. a fast flash-class model); precedence is flag > env > config > session model (self-reflection); an invalid value falls back to the session model with a one-time warning.
- The spec accepts pi's native `--model` thinking suffix: `"zai/glm-5.3-flash:low"` sets classifier thinking to effort low (default without suffix: thinking explicitly off).
- `toggleShortcut` rebinds the master-switch toggle key (`null` or empty disables it, not persisted).
- `enabledByDefault` (fork) seeds the master switch at session start when neither `--auto-mode` nor `--no-auto-mode` was passed. `true` (the default, and upstream's behaviour) starts gated; `false` starts **ungated** — the footer shows `auto mode off` and every tool call runs directly until `/automode on` or the toggle key. Only a literal `false` turns it off; a missing, `null` or mistyped value keeps the gate on. Read once per session, like `toggleShortcut`; the config reload at `session_start` never reverts a runtime `/automode` choice.
- `tamperResponse` (fork) is what a **live** session does when this file or the installed extension copy changes underneath it. `"warn"` (the fork default): notify once, take a new baseline, keep the build and rules already loaded, and carry on — nothing is written back, nobody is asked, nothing fails closed; the change applies to new sessions. So installing a new copy or hand-editing the config never wedges a running session or overwrites your file. `"fail-closed"` is upstream's response: restore the session snapshot (extension always; config only when you decline the dialog) and deny every call until restart. Only the literal `"fail-closed"` selects it.

## Why no built-in allowlist?

Bypass testing of the rule layer ([writeup](../research/rule-layer-security-audit.md)) showed that allowlist robustness is very limited. The built-in layer only makes **deny** claims (the sound direction); allow claims are yours.

## Host notes (pi and oh-my-pi)

pi-verdict runs on both [pi](https://github.com/badlogic/pi-mono) and [oh-my-pi](https://github.com/can1357/oh-my-pi) (omp); the extension self-anchors to whichever agent tree it is installed in. On a dual-install machine the gate follows the extension copy's own location — the mere presence of `~/.omp` never redirects a pi run (and vice versa).

Implementation details worth knowing if you hack on the extension:

- omp 18's `ModelRegistry` has no `complete` method, so the classifier resolves completion through the pi-ai compat module at first gray-zone verdict (`@earendil-works/pi-ai/compat`, which omp's legacy compat layer rewrites to its bundled pi-ai). Resolution or call failures follow the usual fail-closed deny.
- Thinking control is sent in both hosts' native dialects (`thinkingEnabled`/`effort` for pi, `reasoning`/`disableReasoning` for omp); each host reads its own fields and ignores the other's.
- omp installs npm plugins under `plugins/node_modules/<pkg>/` in its config root — under `<agentDir>/` up to omp 18.0, a sibling of `agent/` since omp 18.1; both layouts get the same whole-package-dir self-protection as the pi forms (ADR-0001). The user-rules config always lives under `<agentDir>/config/` on both layouts.
