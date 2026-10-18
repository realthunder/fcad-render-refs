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
    <set>-metal/  the same set captured on the Metal backend
              (raster-metal, raster-flat-metal, chess-metal,
              chess-flat-metal). A golden is a picture of ONE backend:
              tests/render/CMakeLists.txt looks for <set>-metal on
              Apple and registers nothing at all until it exists, so
              these are what a macOS ctest actually runs against.

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

The Metal sets were blessed on macOS 12 / Intel iGPU (vendor 0x8086
device 0x1622): the raster pair on 2026-09-07, the chess pair the same
day, once the harness could stage that scene there at all (the scene
script hard-coded "bgfx - OpenGL", which Apple's GL 2.1 profile cannot
run, and it reached for pivy, which the macOS conda env does not have).
Both chess sets are restaged from their OpenGL siblings and hold the
RASTER frame only -- that box has BUILD_CYCLES off -- and a golden
holding no traced frame is registered without the traced leg, so the
pair stays honest until someone blesses one with Cycles built.

Against their OpenGL siblings the chess sets diverge on 0.33% of the
beauty pixels (max channel delta 140, mean 0.16), which is the same
cross-backend residual the raster pair shows at 0.35%. The lit and flat
Metal pair cross-checks against the OpenGL pair to four decimal places:
77.7928% of pixels against 77.7924%.

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

A sidecar also records the ABSOLUTE path of an asset the scene set --
`Render_PBREnvImage`, the chess set's HDR environment -- and no other
box has that path. `render_verify.py` relocates such a path onto the
current checkout when it restages (the longest tail of it that exists
under the repository), so a set blessed on one box restages on another.
Without that the setting took the dead string SILENTLY, the environment
was simply missing from the frame, and the chess set restaged from a
Linux blessing diverged on 99.98% of its pixels.

The sidecars' `capture.file` is reduced to a basename before blessing;
as captured it holds the absolute path of whatever scratch directory the
run used, which is noise in a committed reference. Nothing reads it --
restaging uses `camera`, `properties` and `preferences`, and the capture
name comes from the filename.

`raster` and `raster-flat` were re-blessed on 2026-10-19, after the
cavity shader was changed on purpose on 2026-10-09 (fcad `66ccd277b9`,
`dfdfc04c5c`): restaged from the sets of 2026-09-08, so the camera is
theirs, and a second capture restaged from the new sets compares
byte-exact at `--tol 0 --frac 0`. Only the beauty frame changed -- 0.13%
of its pixels, max channel delta 26, all on creases; depth, normal, AO
and shadow are byte-identical to the old sets. The sidecars gained the
preferences recorded since (`AntiAliasing`, `Render_MatcapStripes`). The
Metal sets were not re-blessed and still hold the old cavity shader.

## Provenance of the current set

Blessed 2026-09-05 on the conda RelWithDebInfo tree (OCCT 8.0.1), headless
under xvfb, which is llvmpipe/lavapipe software rendering rather than a
device GPU. That is deliberate: it is what a test machine can reproduce.
A real-GPU leg exists (`render-verify.sh --gpu`) and is not what these are
blessed from.
