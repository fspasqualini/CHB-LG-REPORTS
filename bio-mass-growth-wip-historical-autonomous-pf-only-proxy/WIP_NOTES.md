# Bio-Mass Growth WIP

This WIP surface packages the autonomous `pf-only` no-pressure short-horizon matrix together with the later `t_end = 0.12` continuation attempts, including the aborted lumen continuations.

## Why clean outward expansion is blocked under the current setup

- The current lane uses growth_mode = "no-growth" and source_theta = 0, so it does not contain a constitutive relaxed-body growth law or a direct storage-mass source. Autonomous forcing acts only through pf.
- The outer boundary carries a wall penalty with wall_strength = 8.0 and wall_pref = -1.0, which energetically favors tissue-like phase at the rim and makes lumen-like edge takeover expensive.
- The hydraulic mode is closed, so pressure cannot relax through leakage; source forcing can therefore accumulate into internal compression instead of pressure-relieved enlargement.
- The disk edge is not mechanically clamped in displacement. The blockage to clean outward expansion is constitutive and energetic, not a hard u Dirichlet boundary condition.

## Included comparison bundles

- `bio-mass-growth-wip-short-horizon-all`
- `bio-mass-growth-wip-long-horizon-all`
- `bio-mass-growth-wip-s20-lumen-horizon-ladder`
- `bio-mass-growth-wip-s20-tissue-horizon-ladder`
- `bio-mass-growth-wip-s30-lumen-horizon-ladder`
- `bio-mass-growth-wip-s30-tissue-horizon-ladder`
- `bio-mass-growth-wip-s20-long-horizon-sign-contrast`
- `bio-mass-growth-wip-s30-long-horizon-sign-contrast`

## Included packaged runs

- `mig-source-s10-pf-only-autonomous-lumen-t020`
- `mig-source-s10-pf-only-autonomous-tissue-t020`
- `mig-source-s10-pf-only-autonomous-lumen-t030`
- `mig-source-s10-pf-only-autonomous-tissue-t030`
- `mig-source-s10-pf-only-autonomous-lumen-t040`
- `mig-source-s10-pf-only-autonomous-tissue-t040`
- `mig-source-s20-pf-only-autonomous-lumen-t020`
- `mig-source-s20-pf-only-autonomous-tissue-t020`
- `mig-source-s20-pf-only-autonomous-lumen-t030`
- `mig-source-s20-pf-only-autonomous-tissue-t030`
- `mig-source-s20-pf-only-autonomous-lumen-t040`
- `mig-source-s20-pf-only-autonomous-tissue-t040`
- `mig-source-s30-pf-only-autonomous-lumen-t020`
- `mig-source-s30-pf-only-autonomous-tissue-t020`
- `mig-source-s30-pf-only-autonomous-lumen-t030`
- `mig-source-s30-pf-only-autonomous-tissue-t030`
- `mig-source-s30-pf-only-autonomous-lumen-t040`
- `mig-source-s30-pf-only-autonomous-tissue-t040`
- `mig-source-s20-pf-only-autonomous-tissue-t120`
- `mig-source-s30-pf-only-autonomous-tissue-t120`
- `mig-source-s20-pf-only-autonomous-lumen-t120-aborted`
- `mig-source-s30-pf-only-autonomous-lumen-t120-aborted`
