# Subagent Audit - P2

Source: `task/general-purpose` read-only audit on commits `e0a24448`, `abdb96da`, `6d380bec`, `e3bf93c2`.

## Findings

1. **High** — settings refresh path can keep stale anti-hotlink rules after external storage writes due snapshot cache reuse.
   - `webclipper/src/viewmodels/settings/useSettingsSceneController.ts`
   - `webclipper/src/services/integrations/anti-hotlink/anti-hotlink-settings.ts`

2. **Low** — tests miss controller sync and full editor callback flow.
   - `webclipper/tests/unit/settings-sections.test.ts`
   - `webclipper/tests/unit/anti-hotlink-rules-store.test.ts`
