# COLMAP 4.2.0 mapper ordering fixes

This fork maintains execution-order corrections to the incremental SfM mapper
and default Ceres bundle adjuster. It is the corrected **Pipeline B** source
used in the Synthetic Deformable Tubular Benchmark study, separate from the
thesis-era COLMAP-for-colonoscopy derivative used as Pipeline A.

## Source identity and attribution

- Original project: [COLMAP](https://github.com/colmap/colmap), authored by
  Johannes Schönberger and its contributors; original acknowledgments and
  scientific citations are retained in [README.md](README.md).
- Upstream base: [release 4.2.0](https://github.com/colmap/colmap/tree/4.2.0),
  commit `be5e29168d4aff238409d60424812df66aac919f`.
- Corrected source and regression tests: commit
  [`f77beb84e5278dc8435b404fff989073476f115d`](https://github.com/icirauqui/colmap/commit/f77beb84e5278dc8435b404fff989073476f115d).
- Maintained branch: [`mapper-ordering-fixes`](https://github.com/icirauqui/colmap/tree/mapper-ordering-fixes).
- Fixed source release: [`4.2.0-ordering-fixes.1`](https://github.com/icirauqui/colmap/tree/4.2.0-ordering-fixes.1).
  This tag includes the same corrected code and tests, with fork documentation.
  The executable retains the upstream version string `4.2.0`; record the fork
  tag/commit as well to distinguish it from an unmodified upstream build.

The existing `main` branch retains upstream history. The default branch is
`mapper-ordering-fixes`, so the repository opens on the documented corrections.
The fork retains COLMAP's new BSD license and dependency notices in
[COPYING.txt](COPYING.txt). It is maintained by Íñigo Cirauqui-Viloria; it is not
an official upstream COLMAP release. Research using this derivative should cite
the relevant original COLMAP papers and additionally identify this fork's tag
or commit. For incremental SfM, the original reference is Schönberger and Frahm,
*Structure-from-Motion Revisited*, CVPR 2016,
[doi:10.1109/CVPR.2016.445](https://doi.org/10.1109/CVPR.2016.445).

## Changes relative to upstream 4.2.0

Only two runtime source files and their two existing test files are patched.

| Change | Implementation | Reason |
| --- | --- | --- |
| Initial-pair search in candidate order | `src/colmap/sfm/incremental_mapper_impl.cc` | The upstream worker stop flag and shared pair history could let a later, faster candidate prevent an earlier candidate from being tried. Search now visits the declared priority order serially and records attempted pairs in that order. |
| Stable image-ranking ties | Same mapper file | Equal initial-image ranks, next-image ranks and local-bundle overlap ranks are resolved by ascending image ID. |
| Sorted Ceres problem assembly | `src/colmap/estimators/bundle_adjustment_ceres.cc` | Image IDs, explicit variable-point IDs and explicit constant-point IDs are traversed in sorted order before adding residual/parameter blocks, avoiding process-dependent hash-set traversal. Images remain inserted before points. |

The study calls the mapper changes **B-INIT1** and the combined mapper plus
bundle-adjustment changes **B-ORDER1**. The geometry thresholds, optimization
objective and model-size policy are unchanged. Initial-pair search trades its
former candidate-level parallelism for stable priority; this is not a general
speed or reconstruction-accuracy improvement claim.

To inspect the complete source/test patch independently of this documentation:

```bash
git diff 4.2.0..4.2.0-ordering-fixes.1 -- src/colmap
```

## Native regression tests

The patched tests cover the following finite cases:

- `IncrementalMapperTest.InitialPairIndependentOfWorkerCount`: a fixed synthetic
  scene checks the exact selected pair and initial pose at requested worker
  counts 1, 2, 4 and 8, with ten repetitions of each.
- `NumericOrderRegression.CameraResidualAssemblyUsesIdentityOrder`: sixteen
  configuration insertion orders check camera residual assembly by image ID.
- `NumericOrderRegression.RepeatedSolvePreservesExactRawValues`: sixteen solves
  of an unchanged synthetic problem check exact stored pose and point values
  under a fixed single-thread solver configuration.

These controls accompanied the qualified research runs. Exact repetition in
the study was checked under its frozen software, inputs and homogeneous CPU
affinity. The patches and tests do not establish bitwise equality across
different platforms, compiler/dependency versions, GPU implementations or
arbitrary threading configurations.

## Build and test this source release

Install the prerequisites described in the pinned [installation guide](doc/install.rst),
including Ceres and the test dependencies. An example CPU-only source build is:

```bash
git clone --branch 4.2.0-ordering-fixes.1 https://github.com/icirauqui/colmap.git colmap-ordering-fixes
cd colmap-ordering-fixes
cmake -S . -B build -GNinja -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX="$PWD/install" \
  -DGUI_ENABLED=OFF -DCUDA_ENABLED=OFF -DTESTS_ENABLED=ON
cmake --build build --parallel 4
ctest --test-dir build --output-on-failure \
  -R '^(sfm/incremental_mapper_test|estimators/bundle_adjustment_ceres_test)$'
```

The CLI is then `build/src/colmap/exe/colmap`; optional installation with
`cmake --install build` uses the local `install/` prefix. Prebuilt official COLMAP
packages are upstream software and do not supply this fork's changes.

This generic build example is distinct from reproducing the paper's exact
executables. The study's reproduction records bind source patches, compiler
and dependencies, build flags, executable hashes, seeds, thread counts, CPU
affinity and stage commands. In the appearance comparison, B-INIT1 extraction
and matching were retained, followed by B-ORDER1 mapping; physical inputs were
matched with B-ORDER1. This combined release does not relabel those historical
producer identities or imply fresh end-to-end runs for the retained matches.
The separate [reproducibility material](https://huggingface.co/datasets/icirauqui/synthetic-deformable-tubular-benchmark-reproducibility/tree/v1.1.0)
is publicly available with frozen protocols, source/control records, final
results and qualified runtime archives. This repository provides the corrected
source and native tests.
