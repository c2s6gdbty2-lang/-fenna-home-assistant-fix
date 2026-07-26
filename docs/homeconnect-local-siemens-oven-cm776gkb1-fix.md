# Home Connect Local: Siemens CM776GKB1 oven setup fix

## Summary

Home Connect Local 1.0.4 failed while setting up a Siemens CM776GKB1 oven because the dynamic oven status parser treated a nested status field as a cavity id.

Affected integration:

- Domain: `homeconnect_ws`
- Integration: Home Connect Local
- Version: `1.0.4`
- Appliance: Siemens Oven `CM776GKB1`

## Symptom

During Home Assistant setup, the oven config entry failed while building available entities:

```text
custom_components/homeconnect_ws/__init__.py
custom_components/homeconnect_ws/entity_descriptions/__init__.py
custom_components/homeconnect_ws/entity_descriptions/cooking.py
generate_oven_status
```

The failing exception was:

```text
ValueError: invalid literal for int() with base 10: '001.RemainingProgramTime'
```

## Cause

The CM776GKB1 profile contains this status key:

```text
Cooking.Oven.Status.Cavity.001.RemainingProgramTime.AutoCounting
```

The old regular expression in `generate_oven_status` was greedy:

```python
pattern = re.compile(r"^Cooking\.Oven\.Status\.Cavity\.(.*)\..*$")
```

For the key above, it captured:

```text
001.RemainingProgramTime
```

That value was then passed to `int(group[0])`, causing setup to abort.

## Fix

Limit the cavity capture group to the next dot:

```diff
- pattern = re.compile(r"^Cooking\.Oven\.Status\.Cavity\.(.*)\..*$")
+ pattern = re.compile(r"^Cooking\.Oven\.Status\.Cavity\.([^.]+)\..*$")
```

This captures only:

```text
001
```

Nested status keys are still matched, but they no longer corrupt the cavity id.

## Validation

Before the fix:

```text
old_groups = ['001', '001.RemainingProgramTime']
old_int_fails = '001.RemainingProgramTime'
```

After the fix:

```text
new_groups = ['001']
new_int_ok = 001
```

After a full Home Assistant Core restart, the Siemens oven was created successfully:

- Device: `Siemens Oven`
- Model: `CM776GKB1`
- Entities created: `29`

Examples:

- `sensor.siemens_oven_current_temperature`
- `sensor.siemens_oven_bedrijfsstatus`
- `sensor.siemens_oven_power_state`
- `switch.siemens_oven_aan_uit`
- `number.siemens_oven_setpoint_temperature`
- `button.siemens_oven_start`

## Rollback

Restore the original regex in:

```text
custom_components/homeconnect_ws/entity_descriptions/cooking.py
```

Then restart Home Assistant Core.

