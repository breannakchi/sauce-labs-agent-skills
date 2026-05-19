---
name: sauce-config-translator
description: >
  Translate plain English instructions into updated Sauce Labs capability
  blocks. Use when the user wants to modify, reconfigure, or retarget an
  existing Sauce Labs capability set.
when_to_use: >
  Use when the user provides an existing capability block and a natural
  language instruction to change it, OR when the user gives only a natural
  language target and needs a ready-to-use capability block generated from
  scratch.
---

# Sauce Labs Config Translator

## Purpose
This skill translates plain English test-run instructions into correct, validated
Sauce Labs capability blocks. It handles both:
- **Modification**: user pastes existing caps + describes what to change
- **Generation**: user describes a target environment from scratch

## Critical Rules
- Follow ONLY the capability rules defined in this skill. Do not add capabilities from outside knowledge.
- Do not invent capability names. Every capability you output must appear in this skill's templates or rules.
- When switching to RDC: ALWAYS remove appium:platformVersion and ALWAYS add resigningEnabled: true. No exceptions.

## Step 1 — Detect Intent

Read the user's request and classify it:

| Signal | Action |
|---|---|
| User pastes a YAML/JSON block AND describes a change | Modify the existing block |
| User describes a target with no existing block | Generate from scratch using canonical templates |
| Request is ambiguous | Ask one clarifying question: real device or virtual? |
| Request contains "real device", "real iPhone", "real Android", or "physical device" | Classify as RDC immediately — no clarifying question needed |

## Step 2 — Identify the Target Platform

Map common plain English phrases to the correct platform type:

| User says | Platform | Skill context |
|---|---|---|
| "real device", "physical phone", "RDC" | RDC | Apply RDC rules |
| "emulator", "simulator", "virtual", "VDC" | VDC | Apply VDC rules |
| "desktop browser", "Chrome on Windows", "Firefox", "Safari on Mac" | VDC Desktop | Apply VDC desktop rules |
| "Android Chrome", "mobile browser" | VDC or RDC browser | Clarify if real or virtual unless context is clear |
| No device type mentioned, app test implied | Ask: real device or emulator/simulator? | |

## Step 3 — Parse the Change Request

Extract the user's intent from natural language. Common patterns:

### Platform / OS changes
- "latest Chrome" → `browserName: chrome`, `browserVersion: latest`
- "Firefox on Windows 11" → `browserName: firefox`, `platformName: Windows 11`
- "Safari on macOS Sequoia" → `browserName: safari`, `platformName: macOS 15`
- "run on Android" → `platformName: Android`, remove `browserVersion`
- "run on iOS 17" → `platformName: iOS`, `appium:platformVersion: 17`
- "switch to emulator / simulator" → move to VDC, add `appium:platformVersion` and `appium:deviceName`
- "switch to real device" → move to RDC, add `resigningEnabled: true`, remove `appium:platformVersion`

### Device targeting
- "any iPhone" → `appium:deviceName: iPhone.*` (RDC dynamic allocation)
- "any Android" → `appium:deviceName: Google.*` (RDC dynamic allocation)
- "iPad only" → add `sauce:options.tabletOnly: true`
- "phone only" → add `sauce:options.phoneOnly: true`
- "private device" → add `sauce:options.privateDevicesOnly: true`

### Browser changes
- "latest browser" → `browserVersion: latest`
- "stable browser" → `browserVersion: stable`
- "pin to Chrome 123" → `browserVersion: "123"`
- "add ChromeDriver version X" → `sauce:options.chromedriverVersion: X`

### Feature additions
- "add biometrics" → `sauce:options.biometricsInterception: true` + `resigningEnabled: true` (RDC only)
- "add image injection" → `sauce:options.imageInjection: true` + `resigningEnabled: true` (RDC only)
- "slow network / 3G" → `sauce:options.networkProfile: 3G-fast`
- "no network" → `sauce:options.networkProfile: no-network`
- "capture performance" → `sauce:options.extendedDebugging: true` + `sauce:options.capturePerformance: true`
- "record video" → `sauce:options.recordVideo: true`
- "hide sensitive input" → `sauce:options.filterSendKeys: true`
- "add tunnel" → `sauce:options.tunnelName: <ask user for tunnel name>`

### Feature removals
- "remove biometrics" → delete `biometricsInterception`, remove `resigningEnabled` only if no other RDC feature needs it
- "remove throttling" → delete `networkProfile` / `networkConditions`

## Step 4 — Apply Platform Rules

### If target is RDC (real device):
- Always include `sauce:options.resigningEnabled: true` when any advanced feature is present
- Always use `storage:filename=<name>` for app binaries
- Do NOT set `browserVersion`
- Do NOT set `appium:platformVersion` (real device = dynamic allocation)
- Prefer `appium:deviceName` patterns (`iPhone.*`, `Google.*`) unless user specifies a model

### If target is VDC desktop:
- Always include `browserVersion`
- Always include `platformName`
- Check for incompatible combinations:
  - `capturePerformance` requires `extendedDebugging: true`
  - `webSocketUrl: true` is NOT compatible with `extendedDebugging`
  - `devTools: true` is NOT compatible with `extendedDebugging`
- If user requests conflicting options, flag the conflict and ask which they need

### If target is VDC mobile (emulator/simulator):
- Always include `appium:platformVersion`
- Always include `appium:deviceName`
- Always include `appium:automationName` (UIAutomator2 for Android, XCUITest for iOS)
- Do NOT set `resigningEnabled`

## Step 5 — Preserve Existing Values

When modifying an existing block:
- Keep all fields the user did NOT ask to change
- Keep `sauce:options.build` and `sauce:options.name` unless user asks to change them
- Keep tunnel config unless user asks to change it
- Keep timeout values unless user asks to change them
- Only add or remove what was explicitly requested

## Step 6 — Validate Before Output

Run this checklist before returning the result:

- [ ] `platformName` is present and correctly cased (`Android`, `iOS`, `Windows 11`, `macOS 15`, etc.)
- [ ] `appium:automationName` matches platform (`UIAutomator2` for Android, `XCUITest` for iOS)
- [ ] Desktop sessions have `browserVersion`; mobile sessions do not
- [ ] RDC sessions have `resigningEnabled: true` if any advanced feature is enabled
- [ ] No conflicting VDC desktop options (see Step 4)
- [ ] `sauce:options.build` is present
- [ ] `sauce:options` block is present
- [ ] App source uses `storage:filename=...` unless user specified otherwise

## Step 7 — Output Format

- Return the full updated capability block in YAML by default
- If the user's original was JSON, return JSON
- Add a short comment above any non-obvious change, e.g.:
  `# resigningEnabled required for biometricsInterception`
- If you made an assumption (e.g. defaulted to `latest`), state it briefly after the block
- If the request was ambiguous and you had to guess, flag it and offer alternatives

## Supported Network Profiles (for reference)
- `no-throttling`
- `no-network`
- `2G-packet-loss`
- `2G`
- `3G-slow`
- `3G-fast`
- `4G-slow`
- `4G-fast`

## Example Translations

### Example 1 — Switch browser
**User:** "Change this to run on Firefox instead"
**Input caps include:** `browserName: chrome`, `platformName: Windows 11`
**Output change:** `browserName: firefox` — all other fields preserved

### Example 2 — Switch from emulator to real device
**User:** "Run this on a real iPhone instead of the simulator"
**Input caps include:** `platformName: iOS`, `appium:platformVersion: 17`, `appium:deviceName: iPhone 14`
**Output changes:**
- Remove `appium:platformVersion`
- Change `appium:deviceName: iPhone.*`
- Add `sauce:options.resigningEnabled: true`
- Add note: `# Switched to RDC dynamic allocation — real device does not use platformVersion`

### Example 3 — Add a feature
**User:** "Add network throttling to simulate 3G"
**Output change:** Add `sauce:options.networkProfile: 3G-fast` — all other fields preserved

### Example 4 — Conflicting options caught
**User:** "Add performance capture and BiDi websocket endpoint"
**Output:** Flag conflict — `capturePerformance` requires `extendedDebugging: true`, which is incompatible with `webSocketUrl`. Ask user which they actually need.
