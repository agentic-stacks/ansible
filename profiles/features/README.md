# Features Profile

## Overview
Configuration profile for teams adopting bleeding-edge ansible-core 2.17 features and experimental collection versions. Use this profile to stay current, evaluate upcoming capabilities, and prepare for migration before features reach stable.

## ansible-core 2.17 Highlights
- **Improved `ansible.builtin.uri` module** — native retry and backoff parameters for HTTP calls.
- **Template comment stripping** — `jinja2_native` mode now supports `comment_start_string`/`comment_end_string` overrides.
- **Declarative loop controls** — new `loop_control.extended_allitems` toggle for memory optimization in large loops.
- **Module-level timeout** — per-task `timeout:` keyword respected by all connection plugins.
- **Playbook-level `vars_plugins`** — load custom vars plugins directly from `vars_plugins/` without global config changes.

## Experimental Collection Versions
Pin experimental releases in `requirements.yml` with an upper bound to avoid surprise breakage:
```yaml
collections:
  - name: community.general
    version: ">=9.0.0,<10.0.0"
  - name: ansible.utils
    version: ">=4.0.0,<5.0.0"
  - name: ansible.netcommon
    version: ">=7.0.0,<8.0.0"
```

## Known Issues
- `jinja2_native` combined with `ansible.builtin.template` may produce unexpected type coercion for boolean strings — test with `assert` after rendering.
- The `free` strategy in 2.17 has a known race condition when combined with `serial` and `throttle` on the same play — avoid mixing until the fix lands in 2.17.1.
- Collections pinned to pre-release versions (`rc`, `beta`) may not resolve correctly with older `ansible-galaxy` — use `pip install ansible-galaxy-ng` as a workaround.

## Applicable Skills
- `skills/operations/upgrades` — step-by-step version migration and deprecation handling
- `skills/reference/known-issues` — full catalogue of ansible-core 2.16 and 2.17 issues
- `skills/reference/compatibility` — Python, OS, and collection compatibility matrices
- `skills/operations/testing` — CI validation against multiple ansible-core versions
