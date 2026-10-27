# Fluorescence denoising for cell segmentation

**BM3D + Anscombe VST · Python · EECS 556 team project · University of Michigan, 2026**

A classical image restoration baseline for fluorescence microscopy, evaluated through downstream cell segmentation. Our team compared classical and learned denoisers under simulated Poisson noise while holding the Cellpose segmentation model fixed. I implemented the BM3D plus variance stabilizing transform (VST) module.

[Implementation](https://github.com/npkamath/556DenoisingFluorescence/blob/e6b237dd2515c976ecf692fed333d9970c90b9f1/src/denoisers/classical/denoise_bm3d_vst.py) · [Team benchmark figure](https://github.com/npkamath/556DenoisingFluorescence/blob/e6b237dd2515c976ecf692fed333d9970c90b9f1/figures/comparison/ap_comparison.png) · [Repository](https://github.com/npkamath/556DenoisingFluorescence)

## My contribution

The module uses the existing `bm3d` package for Gaussian denoising. I implemented the workflow that adapts it to the project's Poisson inputs:

- **Noise model:** recover simulated Poisson counts from each image's recorded scale, apply the Anscombe VST, and map the denoised output back with an approximate inverse.
- **Channel processing:** denoise the red nucleus and green cytoplasm channels, normalize intensities and noise sigma consistently, and guard against nearly constant inputs.
- **Batch processing:** load the per-image scale table, process `.npy` inputs, and save float32 outputs with clipping and finite value handling.

I also contributed to the team report and README. The [March 25 submission](https://github.com/npkamath/556DenoisingFluorescence/commit/f4e8d8cacf686e8a49baed1f9fa6310e4016eaea) records the original module; a later team reorganization moved it into `src/denoisers/classical/`.

## Reported team results

The team's [archived comparison figure](https://github.com/npkamath/556DenoisingFluorescence/blob/e6b237dd2515c976ecf692fed333d9970c90b9f1/figures/comparison/ap_comparison.png) reports the following mean segmentation scores:

| Input to the frozen segmenter | Mean AP@0.5 |
| --- | ---: |
| Noisy images | 0.3171 |
| **BM3D denoised images** | **0.5559** |
| Clean reference images | 0.7343 |

In this reported comparison, BM3D recovered segmentation performance above the noisy baseline, with a remaining gap to the clean reference. These values are transcribed from the public team figure at revision `e6b237d`. They have not been reproduced for this portfolio.

## Evaluation design

> **Poisson noisy images** → **BM3D + Anscombe VST** → **Frozen Cellpose cyto2** → **Segmentation AP@0.5**

The evaluation uses 68 public Cellpose test image and mask pairs. The same pretrained `cyto2` model processes noisy, denoised, and clean inputs with `channels=[2, 1]`. The metric is the mean of per-image `TP / (TP + FP + FN)` scores at an intersection over union (IoU) threshold of 0.5.

This design measures the usefulness of restoration for segmentation. Its scope is synthetic Poisson corruption and a fixed segmentation model; results depend on that setup and the BM3D script's approximate inverse VST.

## Team and development context

The repository is owned by `npkamath`. Data preparation, segmentation, evaluation, and the other denoisers are shared team work. My contribution is the BM3D/VST module and project reporting, including a [README update](https://github.com/npkamath/556DenoisingFluorescence/commit/a07c80478dcf5f3cd23c0fe5e217efb7d34e1243). Development included both my own coding and AI assistance.

## Reproduction

The run guide follows the [April 22 reference revision](https://github.com/npkamath/556DenoisingFluorescence/tree/e6b237dd2515c976ecf692fed333d9970c90b9f1). Commands have been checked against source, but have not been executed for this portfolio. Most upstream dependencies are unpinned.

<details>
<summary>Environment, data, commands, and output checks</summary>

The guide uses repository revision `e6b237dd2515c976ecf692fed333d9970c90b9f1` from April 22, 2026. Paths below are relative to a fresh checkout of the team repository. Prerequisites are Git, Python 3.10, access to the public test images and masks, and the pretrained `cyto2` weights used by Cellpose. The segmentation commands use the CPU option.

### 1. Get the code and prepare the environment

```sh
git clone https://github.com/npkamath/556DenoisingFluorescence.git
cd 556DenoisingFluorescence
git checkout --detach e6b237dd2515c976ecf692fed333d9970c90b9f1
python3.10 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

The upstream requirements constrain Cellpose to `>=3.0,<4.0`. Most other dependencies are unpinned, so the revision alone does not fix the complete runtime environment. Retain the installed package versions and model weights with any reproduced results.

### 2. Obtain the public test data

Follow the [team dataset instructions](https://github.com/npkamath/556DenoisingFluorescence/blob/e6b237dd2515c976ecf692fed333d9970c90b9f1/README.md#dataset) to obtain `test.zip` from the [Cellpose dataset page](https://www.cellpose.org/dataset). Place the extracted image and mask pairs directly under:

```text
data/raw/test/
  000_img.png
  000_masks.png
  ...
```

This is an example of the expected naming pattern. Check that all 68 image and mask pairs are present. The optional `cyto2` training dataset is not required for this BM3D evaluation route.

The provider's download page currently asks for an institutional email and acceptance of its terms. The dataset is obtained separately from the provider.

### 3. Prepare Poisson noise and run BM3D

The data preparation step uses seed 42 and saves each image's noise scale to `data/noise_params.csv`. Run it once in the fresh checkout, then use those same generated inputs for every compared method.

```sh
python src/data_prep/data_prep.py --noise_mode poisson --seed 42
python src/denoisers/classical/denoise_bm3d_vst.py \
  --noisy_dir data/noisy/poisson \
  --pscale_csv data/noise_params.csv \
  --out_dir results/denoised/bm3d \
  --sigma_vst 1.0
```

### 4. Segment and evaluate

```sh
python src/pipeline/segment.py \
  --input_dir results/denoised/bm3d \
  --output_dir results/pred_masks/bm3d --no_gpu
python src/pipeline/evaluate.py \
  --gt_dir data/masks \
  --pred_dir results/pred_masks/bm3d \
  --out_dir results/ap_scores --method_name bm3d
```

For the noisy and clean references, run the same segmentation and evaluation steps on the prepared inputs:

```sh
python src/pipeline/segment.py \
  --input_dir data/noisy/poisson \
  --output_dir results/pred_masks/noisy --no_gpu
python src/pipeline/evaluate.py \
  --gt_dir data/masks --pred_dir results/pred_masks/noisy \
  --out_dir results/ap_scores --method_name noisy

python src/pipeline/segment.py \
  --input_dir data/clean \
  --output_dir results/pred_masks/clean --no_gpu
python src/pipeline/evaluate.py \
  --gt_dir data/masks --pred_dir results/pred_masks/clean \
  --out_dir results/ap_scores --method_name clean
```

Cellpose uses the project's green cytoplasm and red nucleus mapping, `channels=[2, 1]`. Keep the input set, noise realization, model weights, and segmentation settings the same across methods.

### Expected outputs and completeness checks

These are the files the referenced scripts are designed to write; they are not bundled experiment results.

| Output | Contents |
| --- | --- |
| `data/noise_params.csv` | Image identifiers and Poisson noise scales |
| `results/denoised/bm3d/*.npy` | Denoised `(H, W, 3)` float32 arrays, clipped to `[0, 1]` |
| `results/pred_masks/bm3d/*.npy` | `(H, W)` integer instance masks |
| `results/ap_scores/bm3d.csv` | Per-image scores across IoU thresholds |
| `results/ap_scores/bm3d_summary.csv` | Mean scores and matching counts at each threshold |

Before treating an evaluation as the full 68-image benchmark, verify that the noisy images, denoised images, predicted masks, and ground-truth masks have the same 68 `.npy` filenames. The evaluator scores matching filenames and can return a result for an incomplete subset. Also check output dimensions and finite values, retain the run log, and preserve the package versions, model weights, seed, and noise scale CSV.

</details>

<details>
<summary>Source files</summary>

- [Project README at the referenced revision](https://github.com/npkamath/556DenoisingFluorescence/blob/e6b237dd2515c976ecf692fed333d9970c90b9f1/README.md)
- [Dependency requirements](https://github.com/npkamath/556DenoisingFluorescence/blob/e6b237dd2515c976ecf692fed333d9970c90b9f1/requirements.txt)
- [Data preparation](https://github.com/npkamath/556DenoisingFluorescence/blob/e6b237dd2515c976ecf692fed333d9970c90b9f1/src/data_prep/data_prep.py)
- [Frozen Cellpose segmentation](https://github.com/npkamath/556DenoisingFluorescence/blob/e6b237dd2515c976ecf692fed333d9970c90b9f1/src/pipeline/segment.py)
- [Evaluation and CSV outputs](https://github.com/npkamath/556DenoisingFluorescence/blob/e6b237dd2515c976ecf692fed333d9970c90b9f1/src/pipeline/evaluate.py)

</details>
