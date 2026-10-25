# Changes from upstream

Upstream: https://github.com/HydroSAR/HydroSAR, branch `develop` (the default branch) at commit 479fa65c91619629c486371cb49bcb5712288d27 (2025-10-03), 35 commits after the v1.0.0 release (47d9648). HydroSAR is a NASA Applied Sciences project led by the University of Alaska Fairbanks with the Alaska Satellite Facility (ASF), MSFC, UAH, GSFC and JPL; the package is maintained mainly by Joseph H. Kennedy and others at ASF. BSD 3-Clause License, Copyright (c) 2020, Alaska Satellite Facility (see `LICENSE`, unchanged).

This copy starts from upstream's file tree at 479fa65 without its git history. Left out: `.github/` (upstream's CI, release and PyPI publishing workflows, dependabot config and issue templates). Everything else is as upstream has it, apart from the changes below.

## 2026-10-24: threshold works with NumPy 2

`src/hydrosar/threshold.py`, `expectation_maximization_threshold`: the posterior loop stored the 1 by 1 array returned by `_make_distribution` into a single array element. NumPy 1.25 deprecated that and NumPy 2 raises `ValueError: setting an array element with a sequence`, so `make_HYDRO30` stopped there whenever NumPy 2 was installed, which is what a plain `conda install -c conda-forge hydrosar` gives today. The value is now taken out with `.item()` first.

Results do not change: it is the same float64, and the posterior array does not feed the returned threshold. `environment.yml` keeps its `numpy<2` pin.

## 2026-10-24: the nmad flood depth estimator works

`src/hydrosar/flood_map.py`, `estimate_flood_depth`: the `nmad` branch was followed by a plain `if` for `numpy`, so after computing the nmad statistics the code fell through to the final `else` and raised `Unknown flood depth estimator nmad`. `make_FD30 --estimator nmad` always failed. The branches are now chained with `elif`. This is upstream's own fix, commit a030b27250d30c2bd1d56654df5e4bf910f0ac17 on its `numpy-2` branch (2025-10-03), not yet merged to `develop`.

Results of the `iterative` (default), `numpy` and `logstat` estimators do not change. `nmad` now returns depths instead of an error; `tests/test_flood_map.py::test_estimate_flood_depths_nmad` passes with its expected value.

## 2026-10-24: the last water body gets a flood depth

`src/hydrosar/flood_map.py`, `make_flood_map`: the loop over water bodies ran over `range(1, num_labels)`, but `scipy.ndimage.label` numbers them 1 to `num_labels` (0 is the background). The last water body was never processed and always kept a depth of 0. The comment said the "first, largest" label was skipped, but label 0 is never in the range anyway. The loop now runs to `num_labels` inclusive.

This changes results, on purpose, for the pixels of one water body: the one labeled last, which is the last one met in raster order and can be of any size. On the burst checked below (1,251 water bodies) it has 2 pixels, which now get 6.93 and 6.96 m with the iterative estimator (same random seed) or 0.03 and 0.06 m with the numpy estimator instead of 0 m. On ASF's flood map test scene (S1A_IW_20230228T120437, nmad, 134,289 water bodies) the last one has 46 pixels; 44 of them now get WaterDepth values up to 0.57 m. It is known perennial water there, so FloodDepth does not change. Every other pixel of every output is unchanged.

## 2026-10-24: README note on installing from a clone

`src/hydrosar/README.md` names the conda-forge and PyPI releases (1.0.0), which do not carry these fixes. One short note now says how to install this source tree the way the `Dockerfile` does.

## Not changed, for the record

- `make_FD30`'s default `iterative` estimator calls `scipy.optimize.basinhopping` without a seed, so two runs on the same input give different depths (on the burst below, two runs of unchanged upstream code differed in 22,744 of about 40,500 water pixels, by up to 7 m). Flood extent is not affected. Seeding NumPy's global random state before calling `make_flood_map` makes runs repeatable.
- `src/hydrosar/water_map.py:91`, `np.sqrt(dx**2, dy**2)` passes `dy**2` as NumPy's `out` argument, so the slope uses only one gradient direction. Fixing it would change water maps in hilly terrain (no change on the flat burst below), so it is left for the authors.
- Three integration tests still fail against ASF's golden files, which predate later upstream changes: `test_make_water_map` (pixels identical, but GDAL 3.12 adds a `RESAMPLING` key to the overviews), `test_make_flood_map` (the golden still counts perennial water as flooded and used the older 0.95 perennial threshold; 244,912 pixels differ, the same number with or without the last-water-body fix) and `test_make_copernicus_hand` (a few HAND pixels differ, most likely a pysheds version difference).
- The `make_FD30` usage line in `src/hydrosar/README.md` leaves out the VV raster argument (`make_FD30 OUT_RASTER VV_RASTER WATER_EXTENT_MAP HAND_RASTER`).

## Check

On 2026-10-25, on Linux x86_64 (2 cores), from a fresh `git clone` of this repository: a conda environment built from `environment.yml` (Python 3.11.17, NumPy 1.26.4, GDAL 3.12.3, asf_tools 0.8.4, pysheds 0.5, SciPy 1.17.1), then `python -m pip install .`.

- `pytest tests`: 8 passed, 9 skipped (the integration tests), as upstream.
- `pytest --integration tests`: 14 passed, 3 failed (the three stale golden files above). Upstream at 479fa65 gives 13 passed, 4 failed; the difference is `test_estimate_flood_depths_nmad`, which now passes. Peak memory 7.1 GB, from the full-scene water map test.
- Input: one OPERA RTC-S1 burst over the Sutlej River flood in Punjab, Pakistan, `OPERA_L2_RTC-S1_T173-369617-IW1_20250901T131223Z_20250901T170447Z_S1A_30_v1.0`, `_VV.tif` and `_VH.tif` (7.0 MB each, ASF DAAC).
- `make_HYDRO30 <out>_WM.tif <VV> <VH>` (HAND built from the public GLO-30 HAND): 43 s, 0.93 GB peak. All six outputs (`_WM`, `_WM_HAND`, `_WM_VH/VV_initial`, `_WM_VH/VV_fuzzy`) are identical pixel for pixel to upstream 479fa65 in the same environment. Against OPERA DSWx-S1 from the same pass (four tiles warped onto the burst grid): 99.28% agreement, IoU 65.17%, recall 94.79%, kappa 0.786, the same numbers as before the changes.
- `make_FD30 <out>_FM.tif <VV> <out>_WM.tif <out>_WM_HAND.tif` (iterative): 54 s, 0.40 GB peak, 1,251 water bodies. Flood extent (`_FloodMask`, `_PW`) is identical to upstream. With NumPy's random state seeded the same way, `_WaterDepth` and `_FloodDepth` differ from upstream only in the 2 pixels of the last water body.
- `make_FD30 --estimator nmad ...`: 4.5 s; upstream stops with `Unknown flood depth estimator nmad`.
- With NumPy 2.4.6 installed in the same environment, upstream's `make_HYDRO30` fails with `setting an array element with a sequence`; this copy runs both tools. Its water map differs from the NumPy 1.26 run in 1 pixel of 5.07 million, and seeded flood depths differ by less than 0.0001 m.

Versions come from git tags through setuptools_scm. With the tag `v1.0.0.post1` on this commit the package reports `1.0.0.post1`; without any tag it would report `0.1.devN+g<sha>`.
