# Diffusion Metrics-to-Anatomical Registration

<!-- brainlife-app:begin (generated from the App manifest; edit the manifest, not this block) -->

> **Registers a DTI scalar map (FA, MD, RD or AD) to a T1w or T2w anatomical image with ANTs (translation, rigid, affine or SyN) and reslices it onto the anatomical grid, with QC figures.**

Runs antsRegistration with configurable stages and pyramid depth, then antsApplyTransforms to reslice the moving map onto the anatomical image; outputs the transforms, the resliced map and before/after edge-overlay QC images. Use it to bring diffusion scalar maps onto the anatomical field of view, for example so that tract profiles of an anatomically constrained tractogram do not fall outside the diffusion grid. Only one map is registered: the first available of fa, md, rd, ad; apply the transform to the other maps with app-apply-ants-transform. Not for vector fields such as NODDI dir.

| | |
|---|---|
| 📥 **Inputs** | `neuro/tensor` · `neuro/anat/t1w` (optional) · `neuro/anat/t2w` (optional) |
| 📤 **Outputs** | `neuro/transform/nifti` · `neuro/tensor` · `generic/images` |
| 🐳 **Container** | `gamorosino/tract_align:v1.1` |
| ⚙️ **Resources** | 1 CPUs · 1 h walltime |
| 🔗 **Run it** | [doi.org/10.25663/brainlife.app.978](https://doi.org/10.25663/brainlife.app.978) |

#### 📥 Inputs

| Input | Datatype | Required | Description |
|---|---|---|---|
| `tensor_moving` | `neuro/tensor` | yes | DTI scalar maps; the first available map (fa, md, rd, ad) is used as the moving image. 3D scalar volumes. |
| `t1_fixed` | `neuro/anat/t1w` | no | T1-weighted anatomical reference (fixed image). Used if present. |
| `t2_fixed` | `neuro/anat/t2w` | no | T2-weighted anatomical reference, used only when no T1w is given. At least one of t1_fixed or t2_fixed is required. |

#### 📤 Outputs

| Output | Datatype | Description |
|---|---|---|
| `transformations` | `neuro/transform/nifti` | Affine + optional warp/inverse-warp computed by ANTs registration. |
| `transformed` | `neuro/tensor` | The registered scalar map, resliced onto the anatomical grid. |
| `figures` | `generic/images` | Registration QC: edge-overlay before/after, orthoview + per-plane. |

<details>
<summary><b>⚙️ Parameters</b> (2)</summary>

| Parameter | Type | Default | Description |
|---|---|---|---|
| `transformation` | enum | `rigid` | Stages to run: translation only, +rigid, +affine, or +nonlinear (SyN). Options: `translation`, `rigid`, `affine`, `nonlinear`. |
| `settings` | enum | `1` | Depth of the multiresolution (pyramidal) registration strategy. Higher = more levels, finer correspondence, slower. Options: `1`, `2`, `3`, `4`. |

</details>

<!-- brainlife-app:end -->

A reproducible and automated pipeline to register a diffusion-derived scalar
map — tensor-based (FA, MD, RD, AD) or NODDI-based (NDI, ISOVF, ODI) — to an
anatomical MRI reference (e.g. T1-weighted), using configurable transformation
stages, and to reslice that map onto the anatomical grid.

This is a companion to
[`app-registration-dwi2anat`](https://github.com/gamorosino/app-registration-dwi2anat),
trimmed down for the case where you already have a scalar diffusion metric
map instead of a raw 4D DWI: no b=0 extraction step, and no DWI gradient
reorientation (there is nothing to reorient in a scalar map). The moving
image goes straight into ANTs.

Motivation: diffusion-derived scalar maps and the whole-brain tractogram
they're meant to be sampled against are not always reconstructed on the same
field of view. A tractogram built with anatomically-constrained tracking
(against a T1-derived tissue mask) can extend past the edges of the native
diffusion-reconstruction grid — most visibly at the inferior brainstem/
cerebellum boundary — and any downstream tool that indexes those scalar maps
by voxel coordinate (e.g. tract-profile extraction) can throw an
out-of-bounds error for streamline points that fall just outside that grid.
Registering (and reslicing) the scalar maps onto the anatomical's larger FOV
resolves this.

---

## Author

**Gabriele Amorosino**
(email: [gabriele.amorosino@utexas.edu](mailto:gabriele.amorosino@utexas.edu))

---

## Description

This pipeline registers a diffusion-derived scalar map to a selected
anatomical image using ANTs (`antsRegistration`, staged translation → rigid →
affine → nonlinear, configurable), then reslices that same map onto the
anatomical grid with `antsApplyTransforms`. The workflow is configured via a
`config.json` file and supports execution in containerized environments
(Singularity).

**Supported moving images:** `fa`, `md`, `rd`, `ad` (tensor), `ndi`, `isovf`,
`odi` (NODDI) — all plain 3D scalar volumes.

**Not supported:** NODDI's `dir` (a 3-vector orientation field). Reslicing
each of its components independently under anything beyond a pure
translation would rotate the underlying directions incorrectly. Reorienting
`dir` needs a vector-aware tool (e.g. FSL's `vecreg`) — out of scope here.

**Applying the resulting transform to other maps:** this app only computes
the transform and reslices the one map you register with. To apply that same
transform to additional scalar maps (e.g. register FA, then also reslice
MD/RD/AD/NDI/ISOVF/ODI with the transform already computed), use
[`app-apply-ants-transform`](https://github.com/gamorosino/app-apply-ants-transform)
with this app's `transformations/` output.

---

## Requirements

- [Singularity](https://sylabs.io/guides/latest/user-guide/)

---

## Usage

### Running on Brainlife.io

#### On Brainlife.io via Web UI

1. Navigate to the [Brainlife.io](https://brainlife.io) platform and locate the `app-registration-diffmetrics2anat` app.
2. Click the **Execute** tab.
3. Upload required inputs:
   - A diffusion-derived scalar map (`fa`/`md`/`rd`/`ad`/`ndi`/`isovf`/`odi`, `.nii.gz`)
   - An anatomical image (T1w or T2w)
4. Submit the job. Output will include the registered/resliced map and its transformation matrices/fields.

#### On Brainlife.io via CLI

1. Install the Brainlife CLI: https://brainlife.io/docs/cli/
2. Log in:
   ```bash
   bl login
   ```
3. Run the app:
   ```bash
   bl app run --id <app_id> --project <project_id> \
     --input fa:<fa_dataset_id> \
     --input t1:<t1_dataset_id>
   ```

Replace IDs with the appropriate dataset and project references.

---

### Running Locally

#### Option 1: Using a Configuration File

1. Clone the repository:
   ```bash
   git clone https://github.com/gamorosino/app-registration-diffmetrics2anat.git
   cd app-registration-diffmetrics2anat
   ```

2. Create a `config.json`:
   ```json
   {
       "fa": "sub-01_fa.nii.gz",
       "t1": "sub-01_T1w.nii.gz",
       "transformation": "rigid",
       "settings": "1"
   }
   ```
   (`moving` also works as an explicit generic key instead of `fa`/`md`/`rd`/`ad`/`ndi`/`isovf`/`odi`.)

3. Run the pipeline:
   ```bash
   bash ./main
   ```

#### Option 2: Using the CLI Wrapper

```bash
./main_cli.sh --moving sub-01_fa.nii.gz --t1w sub-01_T1w.nii.gz \
              --transformation rigid --settings 1
```

---

## Outputs

- `ANTs_outputs/` — ANTs registration outputs: affine matrix, warp fields
- `transformations/` — Final transformation files (formatted for Brainlife: `affine.txt`, `warp.nii.gz`, `inverse-warp.nii.gz`)
- `transformed/` — The registered/resliced scalar map, on the anatomical grid
- `QC/` — Edge-overlay registration QC figures (before/after, orthoview + per-plane)
