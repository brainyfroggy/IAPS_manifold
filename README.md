# IAPS Manifold

## Purpose

This repository holds manifold-learning analysis code (PCA, UMAP, diffusion
maps) applied to fMRI searchlight-RSA and EEG data collected while subjects
viewed images from the International Affective Picture System (IAPS).
Voxelwise searchlight-RSA maps (from the companion `IAPS_Searchlight`
pipeline) are embedded into low-dimensional manifolds to characterize
distinct affective-processing "pathways" across the brain — including dorsal
and ventral visual-stream regions, primary visual cortex (V1), and
supramarginal-gyrus/amygdala-adjacent regions. The same searchlight-RSA maps
are also related directly to EEG representational structure, and results are
rendered onto the cortical surface (as static flatmaps and multi-frame
surface animations) for visualization.

This is a research-code-only upload: no raw fMRI/EEG data or computed
intermediates are included (see Caveats below).

**Note on file cleanup:** a prior pass removed `RSA_fmri_eeg og.ipynb`. It
was an earlier draft of `RSA_fmri_eeg.ipynb` covering the same fMRI-EEG RSA
analysis but diverging in its data-loading approach, containing an
unresolved `NameError` partway through, and ending in a manually interrupted
(`KeyboardInterrupt`) whole-brain computation. `RSA_fmri_eeg.ipynb` is the
clean, error-free, complete version and is the one to use.

## Contents

- **`1_explore.ipynb`** — initial, cell-by-cell exploration of the
  searchlight-RSA output array: loading, NaN handling/preprocessing, and a
  first look at manifold embeddings. Meant as a starting point / sanity
  check before the main analysis, not a finished pipeline in itself.

- **`manifold_three_pathway_analysis.ipynb`** — the main analysis. Tests the
  hypothesis that EEG-fMRI representational coupling is organized into three
  distinct visual-processing pathways (dorsal stream, ventral stream, V1).
  Ten parts: intrinsic dimensionality estimation, pathway-hypothesis testing,
  PCA component brain maps, UMAP manifold embedding (at multiple PCA-reduction
  levels), UMAP hyperparameter robustness checks, mapping manifold dimensions
  back onto anatomy, pathway identification via clustering, subject-level
  replication, temporal dynamics (velocity/curvature of the manifold
  trajectory), a diffusion-map alternative to UMAP, and a final summary/export
  of key statistics.

- **`RSA_fmri_eeg.ipynb`** — representational similarity analysis relating
  fMRI searchlight RDMs to EEG RDMs: loads EEG and fMRI data, builds ROI and
  EEG RDMs, computes RSA between them, and evaluates several ROI definitions
  (a CLIP-ViT-derived ROI with behavioral variance excluded, and an AAL3
  atlas-based ROI), ending with per-ROI RSA timecourses.

- **`misc.ipynb`** — despite the generic name, this is a complete, working
  image-processing utility, not scratch work: it tiles rendered inflated
  brain-surface frames (e.g. from the surface-animation pipeline below) into
  single composite images — stripping the background, laying frames out
  left-to-right with configurable overlap and row count, and optionally
  labeling each frame with its timestamp. It contains four successive,
  independently-runnable versions of this tiling script (1 row, 5 rows,
  6 rows, 6 rows with timestamp labels), each fully documented with a
  docstring and each one's cell output confirms it ran successfully.

- **`ROI/nifti_roi_editor.html`** — a standalone, browser-based tool for
  viewing and hand-editing NIfTI-derived ROI masks (paint/undo, per-ROI
  layers, export back to `.nii.gz`). Open the file directly in a browser;
  no server or build step needed.

## How to Use

This repository does not include the underlying fMRI/EEG data or any
computed intermediate (searchlight-RSA arrays, NIfTI volumes, ROI masks —
see Caveats). To rerun the notebooks you will need your own copies of those,
laid out to match the paths near the top of each notebook (several are
hard-coded to the original lab file server/drive letters and will need
updating).

Suggested order:

1. **`1_explore.ipynb`** to load your searchlight-RSA array and sanity-check
   its shape/NaN structure before running the full analysis.
2. **`manifold_three_pathway_analysis.ipynb`** for the main PCA/UMAP/
   diffusion-map pathway analysis — run top to bottom; later parts (subject-
   level robustness, temporal dynamics) depend on arrays computed in earlier
   parts of the same notebook.
3. **`RSA_fmri_eeg.ipynb`** for the fMRI-EEG RSA and ROI-timecourse analysis;
   independent of the manifold notebook above (only shares the underlying
   searchlight-RSA and EEG data).
4. **`ROI/nifti_roi_editor.html`** any time you need to hand-edit an ROI mask
   used as an input to the above — open it in a browser and load your
   `.nii`/`.nii.gz` file.
5. **`misc.ipynb`** after rendering per-timepoint brain-surface images (see
   the rendering pipeline below), to stitch them into a single composite
   figure — point the `glob` pattern at your own rendered-frame folder and
   pick whichever of the four cells matches the row layout/labeling you want.

### Cortical-surface rendering pipeline

The following pipeline projects volumetric searchlight-RSA results onto the
fsaverage cortical surface and renders them as multi-frame surface
animations, using FreeSurfer (via WSL) and FSLeyes. It was originally kept as
an informal instructions file (`readme.docx`) alongside the analysis; it is
reproduced here as markdown instead of being uploaded as a binary document.
`misc.ipynb` above is the final step in this pipeline (tiling the rendered
frames into a static figure) once you have per-frame images.

#### 1. Remove NaNs from the volumetric result (Python)

```python
import nibabel as nib
import numpy as np
from pathlib import Path

p = Path(r"path/to/IAPS_EEG_RSA")
in_file  = p / "rsa_subject_mean.nii.gz"
out_file = p / "rsa_subject_mean_nonan.nii.gz"

img  = nib.load(str(in_file))
data = img.get_fdata()                 # loads full 4D array into RAM
data = np.nan_to_num(data, nan=0.0)    # NaN -> 0

out = nib.Nifti1Image(data.astype(np.float32), img.affine, img.header)
out.header.set_data_dtype(np.float32)   # keep file size reasonable
nib.save(out, str(out_file))
print("Wrote", out_file)
```

#### 2. Project the volume onto the fsaverage surface (WSL, FreeSurfer)

For a single 3D volume:

```bash
WORK="/mnt/n/path/to/IAPS_EEG_RSA/fs_rsa"
cd "$WORK"

# Left hemisphere
mri_vol2surf \
  --mov rsa_tp255_nonan.nii.gz \
  --mni152reg \
  --hemi lh \
  --trgsubject fsaverage \
  --projfrac 0.5 \
  --interp trilinear \
  --o lh.rsa_tp255.mgz

# Right hemisphere (optional)
mri_vol2surf \
  --mov rsa_tp255_nonan.nii.gz \
  --mni152reg \
  --hemi rh \
  --trgsubject fsaverage \
  --projfrac 0.5 \
  --interp trilinear \
  --o rh.rsa_tp255.mgz
```

Convert the resulting `.mgz` into GIFTI (`.gii`) format, and build the fsaverage pial
surface mesh files alongside it:

```bash
mris_convert -c "$WORK/lh.rsa_tp255.mgz" \
    $FREESURFER_HOME/subjects/fsaverage/surf/lh.pial \
    "$WORK/lh.rsa_tp255.shape.gii"

mris_convert $FREESURFER_HOME/subjects/fsaverage/surf/lh.pial \
             "$WORK/lh.pial.surf.gii"
mris_convert $FREESURFER_HOME/subjects/fsaverage/surf/rh.pial \
             "$WORK/rh.pial.surf.gii"
```

#### 3. Multi-frame (per-timepoint) surface projection

For a 4D volume (one frame per timepoint), project every frame separately, then
concatenate:

```bash
WORK="/mnt/n/path/to/IAPS_EEG_RSA/fs_rsa_all"
MOV4D="/mnt/n/path/to/IAPS_EEG_RSA/rsa_subject_mean_nonan.nii.gz"
mkdir -p "$WORK"
cd "$WORK"

CLEAN4D="$MOV4D"
NFRAMES=$(mri_info --nframes "$CLEAN4D")
echo "Frames: $NFRAMES"

# LEFT HEMI: project every frame
rm -f lh.tp*.mgh
for f in $(seq 0 $((NFRAMES-1))); do
  mri_vol2surf \
    --mov "$CLEAN4D" --frame $f \
    --mni152reg \
    --hemi lh --trgsubject fsaverage \
    --projfrac 0.5 --interp trilinear \
    --o "lh.tp$(printf %03d "$f").mgh"
done
mri_concat --i lh.tp*.mgh --o lh.rsa_all.mgh

# RIGHT HEMI: same, with --hemi rh
rm -f rh.tp*.mgh
for f in $(seq 0 $((NFRAMES-1))); do
  mri_vol2surf \
    --mov "$CLEAN4D" --frame $f \
    --mni152reg \
    --hemi rh --trgsubject fsaverage \
    --projfrac 0.5 --interp trilinear \
    --o "rh.tp$(printf %03d "$f").mgh"
done

# Mesh geometry (fsaverage pial)
mris_convert $FREESURFER_HOME/subjects/fsaverage/surf/lh.pial "$WORK/lh.pial.surf.gii"
mris_convert $FREESURFER_HOME/subjects/fsaverage/surf/rh.pial "$WORK/rh.pial.surf.gii"
```

Convert each per-frame `.mgh`/`.mgz` into GIFTI:

```bash
WORK="/mnt/n/path/to/IAPS_EEG_RSA/fs_rsa_all"
LHSURF="$FREESURFER_HOME/subjects/fsaverage/surf/lh.pial"
RHSURF="$FREESURFER_HOME/subjects/fsaverage/surf/rh.pial"
cd "$WORK"

for f in $(ls -1v "$WORK"/lh.tp*.mg? 2>/dev/null); do
  base="${f%.*}"
  mris_convert -c "$f" "$LHSURF" "${base}.func.gii"
done
for f in $(ls -1v "$WORK"/rh.tp*.mg? 2>/dev/null); do
  base="${f%.*}"
  mris_convert -c "$f" "$RHSURF" "${base}.func.gii"
done
```

Then merge the per-frame GIFTI files into a single multi-frame GIFTI (Python):

```python
from pathlib import Path
import nibabel as nib

work = Path(r"path/to/IAPS_EEG_RSA/fs_rsa_all")

def merge(pattern, outname):
    files = sorted(work.glob(pattern))
    first = nib.load(str(files[0]))
    darrays = [nib.gifti.GiftiDataArray(nib.load(str(f)).darrays[0].data.astype('float32'))
               for f in files]
    out = nib.gifti.GiftiImage(darrays=darrays, meta=first.meta, labeltable=first.labeltable)
    nib.save(out, str(work/outname))

merge("lh.tp*.func.gii", "lh.rsa_all.func.gii")
merge("rh.tp*.func.gii", "rh.rsa_all.func.gii")
```

#### 4. Inflated-surface variant

The same per-frame projection can instead be rendered onto the inflated fsaverage
surface (better for viewing sulcal/gyral patterns):

```bash
# One-off: convert inflated meshes to GIFTI (geometry)
mris_convert $FREESURFER_HOME/subjects/fsaverage/surf/lh.inflated lh.inflated.surf.gii
mris_convert $FREESURFER_HOME/subjects/fsaverage/surf/rh.inflated rh.inflated.surf.gii
```

Per-frame conversion to inflated GIFTI (robust version that skips missing frames):

```bash
#!/usr/bin/env bash
set -uo pipefail

WORK="/mnt/n/path/to/IAPS_EEG_RSA/fs_rsa_all"
LHSURF_INFL="$FREESURFER_HOME/subjects/fsaverage/surf/lh.inflated"
RHSURF_INFL="$FREESURFER_HOME/subjects/fsaverage/surf/rh.inflated"
cd "$WORK"

last_idx() {
  local side="$1"
  ls -1v "${side}.tp"*.mg? 2>/dev/null | sed -n 's/.*\.tp\([0-9][0-9][0-9]\)\.mg[hz]$/\1/p' | tail -n1
}
END_LH=$(last_idx lh); END_RH=$(last_idx rh)
: "${END_LH:=999}"; : "${END_RH:=999}"
START=1

for i in $(seq -w $START $END_LH); do
  in_mgh="$WORK/lh.tp${i}.mgh"; in_mgz="$WORK/lh.tp${i}.mgz"
  if   [[ -f "$in_mgh" ]]; then in_file="$in_mgh"
  elif [[ -f "$in_mgz" ]]; then in_file="$in_mgz"
  else continue
  fi
  out_file="${in_file%.*}.inflated.func.gii"
  [[ -f "$out_file" ]] && continue
  mris_convert -c "$in_file" "$LHSURF_INFL" "$out_file"
done

for i in $(seq -w $START $END_RH); do
  in_mgh="$WORK/rh.tp${i}.mgh"; in_mgz="$WORK/rh.tp${i}.mgz"
  if   [[ -f "$in_mgh" ]]; then in_file="$in_mgh"
  elif [[ -f "$in_mgz" ]]; then in_file="$in_mgz"
  else continue
  fi
  out_file="${in_file%.*}.inflated.func.gii"
  [[ -f "$out_file" ]] && continue
  mris_convert -c "$in_file" "$RHSURF_INFL" "$out_file"
done
```

Combine the per-frame inflated GIFTI files into one multi-frame GIFTI (Python, using
natural/numeric sort on the `tpNNN` frame index):

```python
from pathlib import Path
import re
import nibabel as nib
from nibabel.gifti import GiftiImage, GiftiDataArray
from nibabel.nifti1 import intent_codes

work = Path(r"path/to/IAPS_EEG_RSA/fs_rsa_all")

def natural_key(p: Path):
    m = re.search(r'\.tp(\d+)\.', p.name)
    return int(m.group(1)) if m else p.name

def combine_dataonly(pattern: str, outname: str):
    files = sorted(work.glob(pattern), key=natural_key)
    scalars = []
    nverts = None
    for f in files:
        img = nib.load(str(f))
        darrays = [da for da in img.darrays
                   if da.intent not in (intent_codes['NIFTI_INTENT_POINTSET'],
                                        intent_codes['NIFTI_INTENT_TRIANGLE'])]
        arr = darrays[0].data
        nverts = nverts or arr.shape[0]
        scalars.append(GiftiDataArray(arr.astype('float32'),
                                      intent=intent_codes['NIFTI_INTENT_SHAPE']))
    out = GiftiImage(darrays=scalars)
    nib.save(out, str(work / outname))

combine_dataonly("lh.tp*.inflated.func.dataonly.func.gii", "lh.dataonly.multiframe.func.gii")
combine_dataonly("rh.tp*.inflated.func.dataonly.func.gii", "rh.dataonly.multiframe.func.gii")
```

#### 5. Render an animated GIF (FSLeyes)

1. Open the FSLeyes GUI and load the multi-frame surface/volume file.
2. Go to **View -> 3D view** to create a 3D panel.
3. Open **View -> Python console**.
4. Paste the following into the FSLeyes Python console:

```python
panel = [p for p in frame.viewPanels if p.__class__.__name__ == 'Scene3DPanel'][0]

from fsleyes.actions.moviegif import MovieGifAction
movie_action = MovieGifAction(overlayList, displayCtx, panel)
movie_action()
```

#### 6. Break the GIF into frames, then tile them (this repo) or recombine into a video

Use `misc.ipynb` (see Contents above) to tile the extracted per-frame images into a
single composite figure. See `generate_video.ipynb` (kept alongside the original
`IAPS_EEG_RSA` project, not part of this repository) for the frame-extraction/
video-assembly step itself.

## Dependencies

- Standard scientific Python stack: `numpy`, `scipy`, `pandas`, `matplotlib`,
  `seaborn`, `scikit-learn`.
- `umap-learn` for UMAP manifold embedding (`manifold_three_pathway_analysis.ipynb`).
- `nibabel`, `nilearn` for NIfTI I/O and volume handling.
- `Pillow` (`PIL`) for the frame-tiling utility (`misc.ipynb`).
- `ROI/nifti_roi_editor.html` is dependency-free — it runs entirely client-side
  in any modern browser (uses `pako.js` from a CDN for gzip decompression of
  `.nii.gz` files).
- The surface-rendering pipeline additionally requires FreeSurfer (`mri_vol2surf`,
  `mris_convert`, `mri_concat`, `mri_info` — run via WSL on Windows) and FSLeyes
  (for the animated-GIF export step).

## Caveats / not included

This repository contains code and documentation only. The following are intentionally
excluded and are **not** in version control here:
- All `.npy` array files (`cluster_brain_maps.npy`, `cluster_labels.npy`,
  `extreme_voxels_*.npy`, `pca_spatial_loadings.npy`, `pca_temporal_scores.npy`,
  `spatial_cluster_labels_*.npy`, `umap_brain_maps.npy`, `voxel_umap_*.npy`, `mask.npy`,
  etc.). In particular, `searchlight_rsa.npy` (~5 GB) was not copied or opened at all.
- All `.nii.gz` volumetric files (`rsa_subject01..20.nii.gz`,
  `searchlight_rsa_max_over_time.nii.gz`, and the `ROI/*.nii.gz` masks) and the
  `rsa_tp255.mgz` surface file.
- `ROI/*.gz` files (`Dorsal_L.gz`, `Dorsal_R.gz`, `V1_L.gz`, `V1_R.gz`, `Ventral_L.gz`,
  `Ventral_R.gz`) — these are NIfTI-format ROI masks despite the plain `.gz` extension,
  so they were excluded along with the other neuroimaging data.
- `readme.docx` itself was not uploaded; its instructional content is reproduced above
  as the "Cortical-surface rendering pipeline" section instead.

Anyone re-running this analysis will need to regenerate the searchlight-RSA arrays and
NIfTI volumes from the original fMRI/EEG data before these notebooks will run.
