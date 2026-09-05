# fcad render reference images

Blessed reference images for the FreeCAD fork's golden render tests. This
is a repository of its own, mounted in the source tree at
`tests/render/refs`, because a golden set is binary and is rewritten
whenever it is reblessed -- churn that does not belong in the history of
the source tree.

Consumed by `tests/render/CMakeLists.txt`. Every golden test is skipped,
not failed, when this checkout is absent.

## Layout

    raster/   the committed scene, raster only: one camera (iso), the
              five pipeline stages (RenderDebug_ViewMode 0-4). Runs in
              the default ctest.
    raster-flat/  the same scene and stages captured with
              FC_RENDER_TEST_BG=0: the environment still lights the model
              but is not drawn, so the frame is the model on a flat
              ground. Runs in the default ctest.
    cycles/   the same scene and camera path traced on the CPU, plus the
              mode0 beauty frame beside it (the sidecar of that frame is
              what the camera is restaged from). Behind the
              "render-heavy" ctest label.
    chess/    the MaterialX chess set (scripts/render-test-chess.py), iso
              camera, raster beauty frame plus the CPU path-traced one,
              with the HDR environment drawn behind the model. Behind
              "render-heavy".
    chess-flat/  the same scene captured with FC_RENDER_TEST_BG=0: the
              environment lights the model but is not drawn, so the
              frame is the model. Behind "render-heavy".

`raster-flat` was blessed last, on 2026-09-05, and its first blessing
was wrong in a way worth recording: `render-test-scene.py` turned the
viewer gradient off but left `Render_PBREnvBackground` on (it defaults to
on), and the drawn environment sits over the gradient -- so the "flat"
frame was the lit frame, differing on 7.9% of pixels with a maximum
channel delta of 1. The scene now sets `PBREnvBackground` from the same
flag the chess scene does, and the blessed legs diverge on 79.5% of the
beauty pixels with depth, normal, AO and shadow byte-identical. A `-flat`
set that compares near-identical to its sibling has this bug.

The chess sets were blessed on 2026-09-05, after the defect that
kept them out was fixed: a piece of the set intermittently rendered
with no material on the raster leg, because a frame dump could be
consumed by a frame drawn while that material's shader was still
compiling. The dump now waits for a complete frame, and both sets are
restaged captures blessed the same way as the others. See
`docs/RenderDebug.md` section 5.2a for the chase and what a lone
divergent piece would mean now.

Each set is PNGs plus a `<name>.png.json` sidecar per raster capture. The
sidecar is the test manifest: `render_verify.py` restages the camera, the
`Render_*` properties and the recorded preferences from it, so a golden
stays valid when a default changes.

## Reblessing

Everything here was produced by `scripts/render-verify.sh` in the fcad
repo. The order matters:

    # 1. capture once, fresh-staged
    scripts/render-verify.sh capture /tmp/g1 \
        --scene scripts/render-test-scene.py --cams iso --modes 0,1,2,3,4

    # 2. capture AGAIN, restaged from the first set
    scripts/render-verify.sh capture /tmp/g2 --golden /tmp/g1 \
        --scene scripts/render-test-scene.py --modes 0,1,2,3,4

    # 3. bless the SECOND one
    cp /tmp/g2/*.png /tmp/g2/*.json raster/

The flat legs are the same three steps with `FC_RENDER_TEST_BG=0` set on
both captures, blessed into `raster-flat/` and `chess-flat/`.

**Bless the second directory, never the first.** The sidecar stores the
camera as Coin ASCII, which keeps about 8 significant digits, so a
restaged camera is never bit-identical to the freshly staged one it came
from. Measured on this scene: fresh-vs-restaged differs on up to 0.0098%
of pixels with a maximum channel delta of 254, while restaged-vs-restaged
is byte-exact at `--tol 0 --frac 0`. Bless a fresh capture and every
later run compares against a camera it can no longer reproduce.

The cycles set is blessed the same way, with the flags its ctest entry
uses:

    --modes 0 --cycles --cycles-device CPU --cycles-samples 64 \
        --cycles-size 240x180

**Pass the same `--modes` list the test uses.** A golden holding more
stages than the run compares reports the surplus as a *note*, not a
failure -- a silently partial pass.

**CPU, and only CPU.** The offline Cycles path sets no seed, so the
integrator default applies, and denoising is enabled only on the viewport
path -- which is what makes a path-traced golden reproducible at all. Two
devices do not produce identical pixels, so a set blessed on CUDA cannot
be compared against a CPU run.

The sidecars' `capture.file` is reduced to a basename before blessing;
as captured it holds the absolute path of whatever scratch directory the
run used, which is noise in a committed reference. Nothing reads it --
restaging uses `camera`, `properties` and `preferences`, and the capture
name comes from the filename.

## Provenance of the current set

Blessed 2026-09-05 on the conda RelWithDebInfo tree (OCCT 8.0.1), headless
under xvfb, which is llvmpipe/lavapipe software rendering rather than a
device GPU. That is deliberate: it is what a test machine can reproduce.
A real-GPU leg exists (`render-verify.sh --gpu`) and is not what these are
blessed from.
