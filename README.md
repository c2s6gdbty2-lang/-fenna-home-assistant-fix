# Fenna Home Assistant Fixes

Small, documented Home Assistant fixes used by Fenna/Home Sweet Home.

## Included fixes

- Home Connect Local Siemens CM776GKB1 oven setup fix
  - Fixes greedy oven cavity parsing in `homeconnect_ws`
  - Prevents `ValueError: invalid literal for int() with base 10: '001.RemainingProgramTime'`
  - Verified after Home Assistant Core restart with 29 Siemens oven entities created

See:

- `docs/homeconnect-local-siemens-oven-cm776gkb1-fix.md`
- `custom_components/homeconnect_ws/entity_descriptions/cooking.py`

## Scope

This repository intentionally does not contain a full Home Assistant configuration.
It only contains small patches and documentation that are safe to share publicly.

