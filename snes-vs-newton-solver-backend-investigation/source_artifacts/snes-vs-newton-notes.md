# SNES vs Newton Investigation Notes

## 1) Scope and Goal
The branch investigation isolates why coupled CH-Biot runs diverge from expectations when switching solver backends. We compare solver backend/path selection and PETSc option combinations across Task 2 exact/coupled probes and the fixed2 disk probe, using only outputs and code in this worktree.

## 2) Task 2 Exact-Limit CH Fix Already Applied
- `src/chb/models/chb_g_large_strain.py` now applies CH-SNES defaults in `_snes_options_with_defaults(...)` when `solver_backend == "snes"` and explicitly includes linear defaults `ksp_type=preonly`, `pc_type=lu` for CH.
- `docs/issues-surfaced-and-fixed-during-verification.md` records that Task 2 CH-only exact-limit SNES accuracy regressions were attributed to missing explicit CH linear solver defaults (non-`snes` defaults and fallback behavior), and that forcing SNES defaults fixed that lane.
- `tests/vv/test_ch_transport_without_mechanics.py` includes a dedicated forced-SNES CH-only exact-limit test entry point.

## 3) Task 2 Coupled Evidence
- `output/task2-solver-matrix/2026-03-25/exact_matrix.json`: all 12 cases pass for both CH-only and hydraulic-only exact-limit checks.
- `output/task2-solver-matrix/2026-03-25/coupled_matrix.json`: only `snes_default` and `snes_biot_bt_only` pass; 10 fail with `ok=False`, `mismatch=True`, and short records (2/8 expected in most fails).
- `output/task2-solver-matrix/2026-03-25/coupled_monitor_matrix.json`: same paired result on reduced set; pass set is `snes_default`, `snes_biot_bt_only`; fails are `snes_biot_lu` and `snes_biot_demo_like`.
- `output/task2-solver-matrix/2026-03-25/coupled_linear_split_matrix.json`: narrowed Biot linear-stack split shows
  - Passing: `snes_default`, `snes_biot_bt_only`, `snes_biot_gmres_ilu_explicit`, `snes_biot_preonly_ilu`
  - Failing: `snes_biot_gmres_lu`, `snes_biot_lu`
  - This means LU is the consistent failure ingredient on the Task 2 coupled strip, while `preonly` by itself is not enough to break the lane there.

## 4) Fixed2 Disk Probe Evidence
- `output/fixed2-disk-solver-matrix/2026-03-25/fixed2_smoke_matrix.json`: smoke run shows
  - `snes_default`: pass (`accepted_evolved_steps=2`, `final_time=0.002`)
  - `snes_biot_lu`: fail with repeated retries and final stop before advancing accepted trajectory (`accepted_evolved_steps=0`, `final_time=0.0`, return code 1, RuntimeError Nonconvergence at step 1).
- `output/fixed2-disk-solver-matrix/2026-03-25/phase1/*` run summaries show:
  - Passing: `snes_default`, `snes_biot_gmres_ilu_explicit`, `snes_biot_newtontr_gmres_ilu` (`final_time=0.002`, 2 evolved accepted steps).
  - Non-advancing (`final_time=0.0`): `snes_biot_demo_like`, `snes_biot_fgmres_ilu`, `snes_biot_lu`, `snes_biot_newtontr_lu`, `newton_default`, `newton_max_it_40`, `newton_relax_0p5_max_it_40`.
- `output/fixed2-disk-solver-matrix/2026-03-25/fixed2_snes_linear_split.json`: narrowed Biot linear-stack split on fixed2 shows
  - Passing: `snes_default`, `snes_biot_gmres_ilu_explicit`
  - Failing: `snes_biot_gmres_lu`, `snes_biot_preonly_ilu`, `snes_biot_lu`
  - So LU is again a failure ingredient, but fixed2 is stricter than Task 2 because `preonly+ilu` also fails there.
- `output/fixed2-disk-solver-matrix/2026-03-25/fixed2_snes_timing_refine.json`: refined non-LU timing sweep shows all tested GMRES+ILU-family cases pass on fixed2.
  - Fastest total wall time in this small sweep: `snes_biot_gmres_ilu_restart50` (~30.07 s total, ~15.07 s run).
  - `snes_default` and explicit `snes_biot_gmres_ilu_explicit` are essentially tied (~30.31 s and ~30.75 s total).
  - `snes_biot_newtontr_gmres_ilu` also passes but is slightly slower (~31.06 s total).
  - Tightening the inner Krylov solve (`snes_biot_gmres_ilu_ksp_rtol_1e_8`) still passes but is the slowest of the passing family (~32.28 s total).
- `output/fixed2-disk-solver-matrix/2026-03-25/newton_phase2/*` clean rerun confirms all tested Newton variants are non-advancing on fixed2:
  - `newton_max_it_40`: fails with `Nonconvergence at step 1 (ch=False, biot=False, split=nan)`.
  - `newton_relax_0p5_max_it_40`: fails with `Nonconvergence at step 1 (ch=True, biot=False, split=1.6908408013360833e-06)`.
  - `newton_biot_lu`: fails with `Nonconvergence at step 1 (ch=False, biot=False, split=nan)`.
- The rerun logs for `newton_max_it_40` and `newton_relax_0p5_max_it_40` contain repeated `Newton solver did not converge.` warnings before the dt-retry ladder starts, which means the slow Newton cases are active inner-solve failures rather than silent hangs.

## 4b) Public Canonical-Disk and Growth-Only Confirmation Evidence
- `output/canonical-disk-solver-matrix/2026-03-25/canonical_disk_phase1.json`: longer public `canonical_disk.toml` sweep on a hydration-coupled four-seed disk shows
  - Passing: `snes_default` (~98.36 s), `snes_biot_gmres_ilu_explicit` (~96.79 s), `snes_biot_gmres_ilu_restart50` (~91.51 s), `snes_biot_newtontr_gmres_ilu` (~88.93 s).
  - Failing with no advancement: `snes_biot_preonly_ilu` (~42.87 s), `snes_biot_gmres_lu` (~104.58 s).
  - All passing cases reached `t = 0.025` with 41 accepted evolved steps; both failing cases stayed at `final_time = 0.0` and exhausted the dt-retry ladder.
- `output/chb-growth-only-solver-matrix/2026-03-25/medium_confirmation.json`: medium growth-only four-seed confirmation round on `configs/vv_reference/chb_growth_only_canonical_disk_medium.toml` shows
  - Passing: `snes_default` (~155.30 s), `snes_biot_gmres_ilu_explicit` (~158.14 s), `snes_biot_gmres_ilu_restart50` (~154.60 s), `snes_biot_newtontr_gmres_ilu` (~161.80 s).
  - Every case reached `t = 0.01` with 30 accepted evolved steps and zero `step_failed` rows.
  - Wall-time spread is small inside the surviving family: `restart50` is fastest, `snes_default` is only ~0.69 s slower (~0.4%), explicit `gmres+ilu` is ~2.3% slower, and `newtontr+gmres+ilu` is ~4.7% slower.

## 5) Working Hypotheses
- The observed divergence is less likely a pure “Newton vs SNES math” difference and more likely backend/config plumbing and solver stack effects, especially option packages applied to Biot/CH subsolvers.
- In current evidence, Biot-only LU forcing is a repeated failure pattern (`snes_biot_lu`, `snes_biot_demo_like`, `snes_biot_newtontr_lu`, `newton_biot_lu`) while default SNES and GMRES+ILU variants are robust.
- Across both Task 2 and fixed2, LU is the most consistent failure ingredient in the Biot stack. `preonly` without LU passes on Task 2 (`preonly+ilu`) but still fails on fixed2, so `preonly` is not universally toxic in the way LU appears to be.
- The fixed2 evidence now also shows that increasing Newton iterations to 40 and damping Newton (`biot_relaxation=0.5`) do not recover the coupled run.
- Within the surviving SNES family, modest GMRES+ILU variations do not change correctness on fixed2, canonical disk, or the medium growth-only confirmation lane; they mainly move runtime by a small amount, with tighter or alternative choices not yet showing a compelling correctness advantage.
- The CH-only exact fix aligns with an initialization/default-injection gap rather than a model-equation change: adding explicit CH linear defaults restored SNES convergence behavior in exact-limit harnesses.

## 6) Fact vs Inference
- Fact: Case pass/fail states above are directly from JSON results and summaries in `output/...` artifacts.
- Fact: SNES path handling is `solver_backend == "snes"` for CH and `solver_backend in {"snes", "hybrid"}` for Biot in `CHBGLargeStrain`.
- Fact: CH SNES linear defaults now include `preonly+lu` in core model setup.
- Inference: The coupled regression is primarily caused by Biot-stack option combinations and nonlinear trajectory differences, not by a deliberate physics equation change.
- Inference: The most robust tested family is still SNES with Biot on a non-LU iterative stack, especially GMRES+ILU. Fixed2 suggests that `preonly` may also matter on the stricter coupled disk probe, but Task 2 shows LU is the clearer cross-probe red flag.
- Inference: On current evidence, the narrowest “strong default” choice is still PETSc SNES with the public `newtonls + bt` nonlinear path and an explicit non-LU Biot iterative stack. `ksp_gmres_restart=50` is the fastest tested member of that family, but the canonical growth-only confirmation suggests the speed gain over the current public default is small enough that supportability and simplicity still matter.
- Inference: “hybrid” remains present in core code but is not part of public lean normalization and should be treated as non-default/legacy.

## 7) Next Steps
- Re-run `Task 2` and `fixed2` with a narrow option sweep that isolates only Biot SNES linear-stack knobs (KSP/PC vs globalization) while keeping startup/mesh/restart fixed.
- Extend the matrix to the public `configs/canonical_disk.toml` lane as the next 2D four-seed growth-adjacent probe.
  - Rationale: it is the public hydration-coupled disk lane, it already emits the default visual artifacts we use elsewhere (`frames`, montage, movie), and it is closer to the runs we actually want to support than `fixed2`.
  - Caveat: startup still uses `growth_mode = "no-growth"` here, so growth is neutral during bootstrap and then advances during the runtime solve via `g_rate`; this is “growth-capable after startup”, not “growth active from the very first bootstrap state.”
- Use the `chb_growth_only_reference` ladder as the later confirmation lane once the canonical-disk wall-time winner is known.
  - Rationale: it is the only tracked 2D four-seed reference surface that is explicitly about retained growth and symmetry observables.
  - Caveat: `hydration_on = false`, so it is growth-rich but not the closest water-coupled public lane.
- Add one more SNES-only refinement round around the passing non-LU family before broadening again: compare explicit GMRES restart/relative tolerance and, if needed, neighboring Krylov choices rather than jumping back to LU/direct stacks.
- Promote the current evidence into a study-ready comparison bundle under `studies/` once the final report shape is decided; the fixed2 and Task 2 artifacts now support a coherent “LU vs non-LU iterative Biot stack” narrative.
- Add explicit diagnostics for final convergence signatures (`ch`/`biot` convergence reasons) into a small, reproducible diff report so future regressions can distinguish “true fail” vs “did not advance.”
- Keep Task 4 and AMR work out of scope on this branch; continue SNES-first policy for public `chb-dolfinx-snes` lanes.
