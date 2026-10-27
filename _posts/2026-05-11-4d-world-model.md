---
layout: distill
title: "UGG: Unified Generative Geometry Model"
description: "One model learns appearance, depth, and camera motion through conditional completion. We explain how joint training helps, and how image-based verification selects among geometric predictions at inference time."
date: 2026-05-11
last_updated: 2026-10-27
featured: true
tags: research
categories:

authors:
  - name: Amani Kiruga
    url: "https://amanikiruga.github.io"
    affiliations:
      name: Harvard University

bibliography: 2025-12-11-4d-world-model.bib

toc:
  - name: 1 · Why unify generation and prediction?
  - name: 2 · Putting appearance and geometry in the same representation
  - name: 3 · Noise as a way to specify the task
  - name: 4 · Does joint training actually help?
  - name: 5 · One model, three tasks
    subsections:
      - name: Camera-controlled generation
      - name: Camera pose estimation
      - name: Depth prediction
  - name: 6 · Sampling a prediction, then checking it against the images
  - name: 7 · What this changes, and what remains difficult

_styles: >
  d-article table { font-size: 0.78rem; border-collapse: collapse; }
  d-article table th, d-article table td { padding: 4px 7px; text-align: center; border-bottom: 1px solid var(--global-divider-color); }
  d-article table th { font-weight: 600; }
  d-article .best { background: rgba(70,130,200,0.18); font-weight: 600; }
  d-article .sec  { background: rgba(70,130,200,0.07); }
  d-article video { max-width: 100%; height: auto; border-radius: 4px; }
  d-article figcaption { font-size: 0.85em; line-height: 1.35; }
  d-article .vid-grid { display:grid; gap: 6px; }
  d-article .scientific-q { border-left: 3px solid var(--global-theme-color); padding: 6px 12px; margin: 12px 0; background: rgba(0,0,0,0.02); }
  d-article .r4d-carousel { margin: 1em 0; }
  d-article .r4d-head { display:flex; justify-content:space-between; align-items:center; gap:10px; flex-wrap:wrap; margin-bottom:8px; padding-bottom:6px; border-bottom:1px solid var(--global-divider-color); }
  d-article .r4d-tabs { display:flex; gap:4px; flex-wrap:wrap; }
  d-article .r4d-tab { padding:3px 9px; font-size:0.78em; cursor:pointer; border-radius:3px; border:1px solid var(--global-divider-color); background:transparent; color:inherit; font-family:inherit; }
  d-article .r4d-tab.active { background:var(--global-theme-color); color:white; border-color:var(--global-theme-color); }
  d-article .r4d-nav { display:flex; align-items:center; gap:6px; font-size:0.8em; }
  d-article .r4d-btn { cursor:pointer; padding:3px 9px; border:1px solid var(--global-divider-color); border-radius:3px; background:transparent; color:inherit; font-family:inherit; }
  d-article .r4d-btn:hover { background:var(--global-theme-color); color:white; }
  d-article .r4d-counter { font-variant-numeric:tabular-nums; min-width:42px; text-align:center; }
  d-article .r4d-slide { display:none; }
  d-article .r4d-slide.active { display:block; }
  d-article .r4d-cell { display:flex; flex-direction:column; }
  d-article .r4d-cell-label { font-size:0.72em; text-align:center; padding:2px 0 4px 0; opacity:0.75; }
  d-article .r4d-cell-label.ours { color:var(--global-theme-color); font-weight:600; opacity:1; }
  d-article .r4d-cell video, d-article .r4d-cell img { width:100%; height:auto; border-radius:3px; display:block; }
  d-article .result-table { max-width:100%; overflow-x:auto; margin:1rem 0; }
  d-article .result-table table { width:max-content; min-width:100%; font-variant-numeric:tabular-nums; }
  d-article .result-table th, d-article .result-table td { white-space:nowrap; }
  d-article .result-table caption { caption-side:top; text-align:left; font-weight:600; padding:0 0 0.5rem; }
  d-article .result-table tbody th { text-align:left; font-weight:400; }
  d-article .result-table .oracle { opacity:0.8; border-top:2px solid var(--global-divider-color); }
  d-article details { margin:1.1rem 0; padding:0.6rem 0; border-top:1px solid var(--global-divider-color); border-bottom:1px solid var(--global-divider-color); }
  d-article summary { cursor:pointer; font-weight:600; }
---

## 1 · Why unify generation and prediction?

A photo or a short video shows only one slice of a scene: visible surfaces from one viewpoint at one moment in time. To understand the scene, we have to infer what the camera did not capture. How far away are the surfaces? Where did the camera move? What would we see from another viewpoint, or a moment later? These questions connect appearance, geometry, and time. We refer to this as *4D scene understanding*.

The field has historically split this job in two. Geometric predictors estimate depth and camera pose from images. Video generators synthesize what a scene might look like as it evolves. Both have made substantial progress, but they are usually trained to answer different questions. A predictor typically returns one geometric estimate; a video generator can produce many plausible videos without explicitly recovering the geometry behind them.

We ask whether these capabilities can be learned together. RGB describes appearance, depth describes visible 3D structure, and camera pose describes viewpoint. Depending on what we observe, any of these can be an input or an output.

Our work, **UGG: Unified Generative Geometry Model**, treats generation and geometric prediction as conditional completion of the same scene. The same model generates video along a camera trajectory, predicts depth, and recovers camera pose. Changing the task changes which parts of the scene we reveal. This also lets us sample several possible geometric explanations and check them against the images at inference time.

<div class="l-body">
<figure>
<img src="{{ '/assets/img/4d-world-model/teaser2.png' | relative_url }}" alt="RGB, depth, and camera geometry as components of one scene.">
<figcaption>The same scene can be queried in different ways: reveal images to infer geometry, or reveal camera geometry and visual context to generate missing images.</figcaption>
</figure>
</div>

## 2 · Putting appearance and geometry in the same representation

For each frame, we represent the scene with an RGB image, a depth map, and a camera-ray map. RGB and depth already live on a pixel grid. A camera is usually represented by a small collection of parameters: position, rotation, and intrinsics, which describe how viewing directions map to pixels. We instead represent the camera by the ray passing through each pixel.

A ray tells us where the camera is and which direction that pixel looks into the world. We use a Plücker representation: a direction $\mathbf d$ and a moment $\mathbf m=\mathbf o\times\mathbf d$, where $\mathbf o$ is the camera center. This turns camera geometry into an image-shaped modality, aligned with RGB and depth. Depth and camera positions use a shared scene scale, so they describe compatible geometry.

The model can now exchange information between modalities at corresponding positions. A depth boundary, an image edge, and a change in viewing direction are represented on aligned grids rather than through unrelated interfaces.

We initialize from the Wan2.1-T2V-1.3B video model <d-cite key="wan2025wan"></d-cite> and extend it with depth and camera-ray streams. RGB and depth use a frozen video autoencoder; rays are packed directly into aligned latents. A Mixture-of-Transformers backbone <d-cite key="liang2024mot"></d-cite> gives RGB, depth, and rays their own projections and feed-forward layers, while joint attention lets their tokens communicate. Ray directions and moments occupy separate latent streams but share the ray-branch transformer weights.

<div class="l-body">
<figure>
<img src="{{ '/assets/img/4d-world-model/method2.png' | relative_url }}" alt="Multimodal transformer for RGB, depth, and camera rays.">
<figcaption>RGB, depth, and camera rays enter aligned latent streams. Modality-specific layers preserve their representations; joint self-attention exchanges information across them.</figcaption>
</figure>
</div>

## 3 · Noise as a way to specify the task

A diffusion model learns to recover data from noise. We build on *diffusion forcing*, <d-cite key="chen2024diffusionforcing"></d-cite> which allows different parts of a sequence to have different noise levels. We extend this across both frames and modalities. An observed component is clean; a missing component begins as noise.

To predict geometry from a video, we reveal the RGB frames and denoise depth and camera rays. To generate video along a camera path, we reveal that path and one or two RGB frames, then denoise the missing images and depth. With only the first image revealed, the model completes the rest of the scene sequence.

| Observed context | Missing components to infer |
| --- | --- |
| First RGB frame | Remaining RGB, depth, and camera rays |
| Full RGB sequence | Depth and camera rays |
| First and last RGB frames + camera trajectory | Intermediate RGB and depth |
| First RGB frame + camera trajectory | Remaining RGB and depth |
| First RGB frame + depth sequence | Remaining RGB and camera rays |

<!-- VIDEO-MASKS: A short animation can switch the clean/noisy pattern while keeping the same scene and network. -->

These are different conditioning patterns for the same weights. During inference, the observed tokens stay fixed at every denoising step, while the missing tokens are progressively recovered. We also use history guidance <d-cite key="song2025dfot"></d-cite> to strengthen the influence of the observed context.

The training objective is flow matching. For a clean token $z_0$ and Gaussian noise $\epsilon$, we form

$$
z_t=(1-t)z_0+t\epsilon.
$$

The network predicts the velocity $\epsilon-z_0$. Different tokens can have different values of $t$, allowing clean observations and noisy targets to coexist in one sequence.

This matters for the data as well. Most video datasets do not provide RGB, depth, camera pose, and text together. We train on eight real, synthetic, and robotic datasets with different subsets of annotations, computing a loss only where a target is available. RGB-only clips contribute visual supervision; clips with camera or depth annotations also supervise those modalities. Missing labels do not need to be invented to fit a common training format.

<details markdown="1">
<summary>Training data and available annotations</summary>

The training mixture comprises RealEstate10K, DL3DV-10K, SpatialVID, ScanNet++, AgiBot World, OmniWorld, Virtual KITTI 2, and PointOdyssey. DL3DV-10K and ScanNet++ use depth and camera annotations from the VIPE pipeline; this is training supervision, separate from the test-time verifier. SpatialVID and AgiBot provide captions, and DL3DV-10K and RealEstate10K use annotated scene descriptions. Other datasets train without text conditioning.

| Dataset | RGB | Depth | Camera pose |
| --- | --- | --- | --- |
| RealEstate10K | Yes | — | Yes |
| DL3DV-10K | Yes | VIPE | VIPE |
| SpatialVID | Yes | — | Yes |
| ScanNet++ | Yes | VIPE | VIPE |
| AgiBot World | Yes | — | — |
| OmniWorld | Yes | Yes | Yes |
| Virtual KITTI 2 | Yes | Yes | Yes |
| PointOdyssey | Yes | Yes | Yes |

</details>

## 4 · Does joint training actually help?

Handling several tasks in one network is useful, but it does not by itself show that they help each other. We test this with controlled ablations: the same clips, 7,500 optimizer steps, and a matched training budget, removing one task or design choice at a time.

Removing RGB generation raises KITTI depth error from **0.152 to 0.322**. Removing depth prediction raises ScanNetV2 rotation error from **2.91° to 4.30°**. Removing pose estimation lowers RealEstate10K generation PSNR from **20.18 to 13.26 dB**. Each task contributes to a different task's performance.

We also remove RealEstate10K, which provides camera supervision but no depth labels. Depth error still rises on Bonn, from **0.135 to 0.239**, and on KITTI, from **0.152 to 0.196**. This is evidence that depth prediction can benefit from data that does not directly supervise depth.

The architecture matters too. Replacing image-aligned ray maps with global camera tokens worsens pose estimation and generation in the reported ablations. A single shared transformer also loses performance on several tasks, although it is slightly better for Bonn depth. The benefit comes from allowing the modalities to interact while retaining their own representations.

<details markdown="1">
<summary>Controlled ablation results</summary>

Each comparison uses the same sampled clips and hyperparameters, with an effective batch size of 8. Removing a task sets its loss to zero and replaces its target inputs with noise; retained loss weights are not renormalized. When RGB generation is removed, RGB remains available as clean conditioning. These are controlled smaller-budget runs, separate from the main-model benchmark tables.

| Evaluation | Complete architecture | Removed task or data |
| --- | ---: | ---: |
| KITTI depth Abs Rel ↓; without RGB generation | 0.152 | 0.322 |
| ScanNetV2 pose RPEr ↓ (°); without depth prediction | 2.91 | 4.30 |
| RealEstate10K generation PSNR ↑; without pose estimation | 20.18 | 13.26 |
| Bonn depth Abs Rel ↓; without RealEstate10K | 0.135 | 0.239 |
| KITTI depth Abs Rel ↓; without RealEstate10K | 0.152 | 0.196 |

</details>

## 5 · One model, three tasks

### Camera-controlled generation

Given a camera trajectory and either the first image or the first and last images, UGG generates the remaining video. We compare quantitatively with **Aether**, another unified generation-and-geometry model <d-cite key="zhu2025iccv-aether"></d-cite>, and **SEVA**, a novel-view synthesis model <d-cite key="zhou2025seva"></d-cite>. The qualitative comparisons also include **Wan2.1-FLF-14B** and **DFoT**. Wan's first-last-frame model has no camera-trajectory input, so it tests endpoint-conditioned completion rather than equivalent camera control.

On RealEstate10K, SpatialVID, and DL3DV, UGG leads the compared methods on all three reported metrics in the one-view setting: PSNR and SSIM measure image agreement, while LPIPS measures perceptual similarity. With two views, it leads on RealEstate10K and SpatialVID; SEVA is stronger on DL3DV. For example, two-view RealEstate10K PSNR is **25.73 dB**, compared with **25.25** for SEVA and **20.02** for Aether.

The videos also let us inspect something a table cannot fully describe: whether dynamic content remains coherent as the viewpoint changes. A requested camera motion and a moving person both have to be accounted for in the same completion.

<!-- VIDEO-NVS: Replace or extend this gallery with the updated capability videos. -->
<div class="l-page"><figure>
<div class="r4d-carousel" data-r4d="nvs"></div>
<figcaption>Camera-controlled generation from first and last images and a supplied trajectory. The existing gallery compares UGG, SEVA, Wan2.1-FLF, and DFoT with the reference video; Aether is included in the quantitative table.</figcaption>
</figure></div>

<details markdown="1">
<summary>Full camera-controlled generation results</summary>

Both protocols also receive the target camera trajectory. Wan2.1-FLF and DFoT are qualitative comparisons, not rows in the current quantitative table. Best and second-best shading is computed among the non-oracle rows within each protocol, with ties determined by the displayed rounded values. Oracle rows are ground-truth diagnostics and excluded from that ranking.

#### Two-view generation

<div class="result-table"><table data-result-table="generation-2-view">
<caption>Two-view generation</caption>
<thead>
<tr><th scope="col" rowspan="2">Method</th><th scope="colgroup" colspan="3">RealEstate10K</th><th scope="colgroup" colspan="3">SpatialVID</th><th scope="colgroup" colspan="3">DL3DV</th></tr>
<tr><th scope="col">PSNR ↑</th><th scope="col">LPIPS ↓</th><th scope="col">SSIM ↑</th><th scope="col">PSNR ↑</th><th scope="col">LPIPS ↓</th><th scope="col">SSIM ↑</th><th scope="col">PSNR ↑</th><th scope="col">LPIPS ↓</th><th scope="col">SSIM ↑</th></tr>
</thead>
<tbody>
<tr><th scope="row">Aether</th><td>20.02</td><td>0.196</td><td>0.624</td><td>13.84</td><td>0.489</td><td>0.334</td><td>13.14</td><td>0.399</td><td>0.243</td></tr>
<tr><th scope="row">SEVA</th><td class="sec">25.25</td><td class="sec">0.081</td><td class="sec">0.838</td><td class="sec">15.44</td><td class="sec">0.344</td><td class="sec">0.466</td><td class="best">17.61</td><td class="best">0.211</td><td class="best">0.552</td></tr>
<tr><th scope="row">UGG</th><td class="best">25.73</td><td class="best">0.061</td><td class="best">0.849</td><td class="best">16.99</td><td class="best">0.288</td><td class="best">0.535</td><td class="sec">16.61</td><td class="sec">0.239</td><td class="sec">0.495</td></tr>
</tbody>
</table></div>


#### One-view generation

<div class="result-table"><table data-result-table="generation-1-view">
<caption>One-view generation</caption>
<thead>
<tr><th scope="col" rowspan="2">Method</th><th scope="colgroup" colspan="3">RealEstate10K</th><th scope="colgroup" colspan="3">SpatialVID</th><th scope="colgroup" colspan="3">DL3DV</th></tr>
<tr><th scope="col">PSNR ↑</th><th scope="col">LPIPS ↓</th><th scope="col">SSIM ↑</th><th scope="col">PSNR ↑</th><th scope="col">LPIPS ↓</th><th scope="col">SSIM ↑</th><th scope="col">PSNR ↑</th><th scope="col">LPIPS ↓</th><th scope="col">SSIM ↑</th></tr>
</thead>
<tbody>
<tr><th scope="row">Aether</th><td>15.79</td><td>0.335</td><td>0.469</td><td>11.81</td><td>0.632</td><td>0.283</td><td>11.96</td><td>0.506</td><td>0.240</td></tr>
<tr><th scope="row">SEVA</th><td class="sec">19.58</td><td class="sec">0.194</td><td class="sec">0.703</td><td class="sec">13.35</td><td class="sec">0.472</td><td class="sec">0.343</td><td class="sec">13.50</td><td class="sec">0.393</td><td class="sec">0.335</td></tr>
<tr><th scope="row">UGG</th><td class="best">21.79</td><td class="best">0.150</td><td class="best">0.779</td><td class="best">14.86</td><td class="best">0.428</td><td class="best">0.405</td><td class="best">13.92</td><td class="best">0.385</td><td class="best">0.346</td></tr>
</tbody>
</table></div>

</details>

### Camera pose estimation

For geometric prediction, we distinguish **dense** inputs, where the full RGB video is observed, from **sparse** inputs, where only the first and last images are given to the model. We compare pose quantitatively with **Geo4D, Gen3R, and Aether** <d-cite key="jiang2025geo4d,Huang_2026_CVPR,zhu2025iccv-aether"></d-cite>; RayDiffusion appears in the qualitative comparisons.

Dense pose is evaluated on RealEstate10K, Sintel, TUM, ScanNetV2, and ScanNet++. After aligning trajectories up to global rotation, translation, and scale, we measure absolute trajectory error (ATE) and relative translation and rotation errors. Without search, UGG has lower ATE and relative translation error than all three external baselines on all five benchmarks. On ScanNet++, ATE is **0.087**, compared with **0.190** for Geo4D. Rotation is also strong, though Gen3R remains better on TUM.

With only two observed images, we evaluate their relative pose. Translation angular error (tAE) measures whether the predicted direction of motion is correct; rotation error measures the change in orientation. UGG has the lowest errors among the external baselines on both metrics across all five datasets. On RealEstate10K, its translation angular error is **8.260°**, compared with **39.834°** for Aether, **44.004°** for Gen3R, and **117.079°** for Geo4D.

<!-- VIDEO-POSE: Replace or extend this gallery with the updated capability videos. -->
<div class="l-page"><figure>
<div class="r4d-carousel" data-r4d="pose"></div>
<figcaption>Representative camera trajectories from the existing qualitative gallery: reference, UGG, Geo4D, and RayDiffusion. Gen3R and Aether are included in the quantitative table. These examples illustrate trajectory structure; they are separate from the updated benchmark tables.</figcaption>
</figure></div>

<details markdown="1">
<summary>Full camera pose results</summary>

Dense trajectories use Sim(3) Umeyama alignment. ATE and RPEt are reported in the paper’s aligned coordinate convention; RPEr is in degrees. Sparse pose scores the relative camera between the two observed endpoints, with both tAE and RPEr in degrees. UGG uses shared intrinsics. Sparse TTS uses additional real RGB for verification, as described in Section 6; the base UGG and external baseline rows use the sparse model-input protocol. Best and second-best shading is computed among the non-oracle rows within each protocol, with ties determined by the displayed rounded values. Oracle rows are ground-truth diagnostics and excluded from that ranking.

#### Dense pose

<div class="result-table"><table data-result-table="pose-dense">
<caption>Dense pose</caption>
<thead>
<tr><th scope="col" rowspan="2">Method</th><th scope="colgroup" colspan="3">RealEstate10K</th><th scope="colgroup" colspan="3">Sintel</th><th scope="colgroup" colspan="3">TUM</th><th scope="colgroup" colspan="3">ScanNetV2</th><th scope="colgroup" colspan="3">ScanNet++</th></tr>
<tr><th scope="col">ATE ↓</th><th scope="col">RPEt ↓</th><th scope="col">RPEr ↓ (°)</th><th scope="col">ATE ↓</th><th scope="col">RPEt ↓</th><th scope="col">RPEr ↓ (°)</th><th scope="col">ATE ↓</th><th scope="col">RPEt ↓</th><th scope="col">RPEr ↓ (°)</th><th scope="col">ATE ↓</th><th scope="col">RPEt ↓</th><th scope="col">RPEr ↓ (°)</th><th scope="col">ATE ↓</th><th scope="col">RPEt ↓</th><th scope="col">RPEr ↓ (°)</th></tr>
</thead>
<tbody>
<tr><th scope="row">Geo4D</th><td>0.026</td><td>0.028</td><td>0.147</td><td>0.160</td><td>0.097</td><td>0.911</td><td>0.052</td><td>0.029</td><td>1.414</td><td class="sec">0.074</td><td>0.044</td><td>1.762</td><td>0.190</td><td>0.090</td><td>5.591</td></tr>
<tr><th scope="row">Gen3R</th><td>0.030</td><td class="sec">0.011</td><td class="sec">0.141</td><td>0.233</td><td>0.066</td><td>1.538</td><td>0.046</td><td class="sec">0.016</td><td class="best">0.943</td><td>0.075</td><td class="sec">0.024</td><td>1.018</td><td>0.232</td><td>0.066</td><td>5.760</td></tr>
<tr><th scope="row">Aether</th><td>0.018</td><td>0.014</td><td>0.851</td><td>0.180</td><td>0.867</td><td>1.354</td><td>0.041</td><td class="sec">0.016</td><td>1.462</td><td>0.075</td><td>0.030</td><td>1.617</td><td>0.225</td><td>0.068</td><td>5.944</td></tr>
<tr><th scope="row">UGG</th><td class="sec">0.010</td><td class="best">0.004</td><td class="best">0.076</td><td class="sec">0.098</td><td class="sec">0.034</td><td class="sec">0.708</td><td class="sec">0.018</td><td class="best">0.007</td><td class="sec">1.052</td><td class="best">0.038</td><td class="best">0.016</td><td class="sec">0.938</td><td class="sec">0.087</td><td class="sec">0.036</td><td class="best">3.861</td></tr>
<tr><th scope="row">UGG (TTS)</th><td class="best">0.009</td><td class="best">0.004</td><td class="best">0.076</td><td class="best">0.091</td><td class="best">0.033</td><td class="best">0.684</td><td class="best">0.017</td><td class="best">0.007</td><td>1.054</td><td class="best">0.038</td><td class="best">0.016</td><td class="best">0.937</td><td class="best">0.082</td><td class="best">0.035</td><td class="sec">4.105</td></tr>
<tr class="oracle"><th scope="row">UGG (Oracle; GT diagnostic)</th><td>0.008</td><td>0.004</td><td>0.076</td><td>0.072</td><td>0.030</td><td>0.695</td><td>0.015</td><td>0.006</td><td>1.053</td><td>0.033</td><td>0.015</td><td>0.939</td><td>0.072</td><td>0.035</td><td>4.078</td></tr>
</tbody>
</table></div>


#### Sparse pose

<div class="result-table"><table data-result-table="pose-sparse">
<caption>Sparse pose</caption>
<thead>
<tr><th scope="col" rowspan="2">Method</th><th scope="colgroup" colspan="2">RealEstate10K</th><th scope="colgroup" colspan="2">Sintel</th><th scope="colgroup" colspan="2">TUM</th><th scope="colgroup" colspan="2">ScanNetV2</th><th scope="colgroup" colspan="2">ScanNet++</th></tr>
<tr><th scope="col">tAE ↓ (°)</th><th scope="col">RPEr ↓ (°)</th><th scope="col">tAE ↓ (°)</th><th scope="col">RPEr ↓ (°)</th><th scope="col">tAE ↓ (°)</th><th scope="col">RPEr ↓ (°)</th><th scope="col">tAE ↓ (°)</th><th scope="col">RPEr ↓ (°)</th><th scope="col">tAE ↓ (°)</th><th scope="col">RPEr ↓ (°)</th></tr>
</thead>
<tbody>
<tr><th scope="row">Geo4D</th><td>117.079</td><td>3.407</td><td>108.628</td><td>24.160</td><td>100.224</td><td>11.936</td><td>90.812</td><td>43.458</td><td>82.747</td><td>83.019</td></tr>
<tr><th scope="row">Gen3R</th><td>44.004</td><td>4.197</td><td>75.984</td><td>24.984</td><td>82.875</td><td>12.374</td><td>79.871</td><td>43.481</td><td>92.324</td><td>82.540</td></tr>
<tr><th scope="row">Aether</th><td>39.834</td><td>14.969</td><td>78.012</td><td>33.640</td><td>48.436</td><td>6.234</td><td>70.925</td><td>36.720</td><td>77.234</td><td>81.681</td></tr>
<tr><th scope="row">UGG</th><td class="sec">8.260</td><td class="best">1.287</td><td class="sec">35.479</td><td class="sec">22.065</td><td class="sec">9.938</td><td class="best">1.371</td><td class="sec">28.240</td><td class="sec">21.417</td><td class="sec">40.194</td><td class="sec">43.078</td></tr>
<tr><th scope="row">UGG (TTS)</th><td class="best">3.475</td><td class="sec">1.393</td><td class="best">18.733</td><td class="best">20.646</td><td class="best">8.520</td><td class="sec">1.444</td><td class="best">20.057</td><td class="best">18.911</td><td class="best">31.835</td><td class="best">37.875</td></tr>
<tr class="oracle"><th scope="row">UGG (Oracle; GT diagnostic)</th><td>1.550</td><td>1.399</td><td>8.521</td><td>26.135</td><td>3.401</td><td>1.154</td><td>9.683</td><td>20.688</td><td>7.892</td><td>43.836</td></tr>
</tbody>
</table></div>

</details>

### Depth prediction

We evaluate depth on ScanNet++, 7-Scenes, KITTI, Sintel, and Bonn, against **ChronoDepth, Geo4D, Gen3R, Aether, and Marigold** <d-cite key="shao2025chronodepth,jiang2025geo4d,Huang_2026_CVPR,zhu2025iccv-aether,ke2023marigold"></d-cite>. We align depth at the video level for scale and shift, then report absolute relative error (Abs Rel; lower is better) and the fraction of pixels within a factor of 1.25 of the reference depth ($\delta_{1.25}$; higher is better). These are aligned-depth results, rather than a test of absolute metric scale.

UGG is competitive with specialized depth systems across these settings. In dense ScanNet++, its Abs Rel is **0.073**, compared with **0.106** for Geo4D, the strongest external baseline there. Its advantage is particularly clear with sparse observations: UGG leads the external baselines on both depth metrics on ScanNet++, 7-Scenes, Sintel, and Bonn. Specialized models retain advantages on some dense benchmarks, including Aether on KITTI and Geo4D on Sintel.

<!-- VIDEO-DEPTH: Replace or extend this gallery with the updated capability videos. -->
<div class="l-page"><figure>
<div class="r4d-carousel" data-r4d="depth"></div>
<figcaption>RGB, reference depth, UGG, Geo4D, and ChronoDepth from the existing qualitative gallery. The updated quantitative comparisons also include Gen3R, Aether, and Marigold. These examples are separate from the updated benchmark tables.</figcaption>
</figure></div>

<details markdown="1">
<summary>Full depth results</summary>

Depth is aligned for scale and shift at the video level. Abs Rel is lower-is-better and δ1.25 is higher-is-better. Sparse depth evaluates the two observed endpoints. Sparse TTS additionally reads neighbouring real RGB during selection, as described in Section 6. Oracle metrics describe a ground-truth-selected candidate, not an independently optimized best value in each column. Best and second-best shading is computed among the non-oracle rows within each protocol, with ties determined by the displayed rounded values. Oracle rows are ground-truth diagnostics and excluded from that ranking.

#### Dense depth

<div class="result-table"><table data-result-table="depth-dense">
<caption>Dense depth</caption>
<thead>
<tr><th scope="col" rowspan="2">Method</th><th scope="colgroup" colspan="2">ScanNet++</th><th scope="colgroup" colspan="2">7-Scenes</th><th scope="colgroup" colspan="2">KITTI</th><th scope="colgroup" colspan="2">Sintel</th><th scope="colgroup" colspan="2">Bonn</th></tr>
<tr><th scope="col">Abs Rel ↓</th><th scope="col">δ1.25 ↑</th><th scope="col">Abs Rel ↓</th><th scope="col">δ1.25 ↑</th><th scope="col">Abs Rel ↓</th><th scope="col">δ1.25 ↑</th><th scope="col">Abs Rel ↓</th><th scope="col">δ1.25 ↑</th><th scope="col">Abs Rel ↓</th><th scope="col">δ1.25 ↑</th></tr>
</thead>
<tbody>
<tr><th scope="row">ChronoDepth</th><td>0.177</td><td>0.724</td><td>0.116</td><td>0.871</td><td>0.120</td><td>0.857</td><td>0.394</td><td>0.530</td><td>0.092</td><td>0.920</td></tr>
<tr><th scope="row">Geo4D</th><td>0.106</td><td>0.894</td><td class="sec">0.080</td><td class="sec">0.923</td><td>0.097</td><td>0.923</td><td class="best">0.247</td><td class="best">0.676</td><td class="best">0.057</td><td class="best">0.972</td></tr>
<tr><th scope="row">Gen3R</th><td>0.157</td><td>0.772</td><td class="best">0.079</td><td class="best">0.937</td><td>0.185</td><td>0.777</td><td>0.414</td><td>0.520</td><td>0.168</td><td>0.805</td></tr>
<tr><th scope="row">Aether</th><td>0.166</td><td>0.769</td><td>0.109</td><td>0.898</td><td class="best">0.056</td><td class="best">0.975</td><td>0.351</td><td>0.607</td><td>0.095</td><td>0.911</td></tr>
<tr><th scope="row">Marigold</th><td>0.175</td><td>0.730</td><td>0.094</td><td>0.910</td><td>0.098</td><td>0.910</td><td>0.388</td><td>0.517</td><td>0.082</td><td>0.933</td></tr>
<tr><th scope="row">UGG</th><td class="sec">0.073</td><td class="sec">0.952</td><td>0.083</td><td>0.921</td><td>0.064</td><td>0.960</td><td>0.303</td><td>0.654</td><td>0.060</td><td>0.963</td></tr>
<tr><th scope="row">UGG (TTS)</th><td class="best">0.072</td><td class="best">0.954</td><td>0.084</td><td>0.920</td><td class="sec">0.063</td><td class="sec">0.962</td><td class="sec">0.293</td><td class="sec">0.661</td><td class="sec">0.059</td><td class="sec">0.969</td></tr>
<tr class="oracle"><th scope="row">UGG (Oracle; GT diagnostic)</th><td>0.063</td><td>0.963</td><td>0.074</td><td>0.923</td><td>0.055</td><td>0.967</td><td>0.263</td><td>0.692</td><td>0.056</td><td>0.973</td></tr>
</tbody>
</table></div>


#### Sparse depth

<div class="result-table"><table data-result-table="depth-sparse">
<caption>Sparse depth</caption>
<thead>
<tr><th scope="col" rowspan="2">Method</th><th scope="colgroup" colspan="2">ScanNet++</th><th scope="colgroup" colspan="2">7-Scenes</th><th scope="colgroup" colspan="2">KITTI</th><th scope="colgroup" colspan="2">Sintel</th><th scope="colgroup" colspan="2">Bonn</th></tr>
<tr><th scope="col">Abs Rel ↓</th><th scope="col">δ1.25 ↑</th><th scope="col">Abs Rel ↓</th><th scope="col">δ1.25 ↑</th><th scope="col">Abs Rel ↓</th><th scope="col">δ1.25 ↑</th><th scope="col">Abs Rel ↓</th><th scope="col">δ1.25 ↑</th><th scope="col">Abs Rel ↓</th><th scope="col">δ1.25 ↑</th></tr>
</thead>
<tbody>
<tr><th scope="row">ChronoDepth</th><td>0.199</td><td>0.700</td><td>0.130</td><td>0.844</td><td class="sec">0.089</td><td>0.905</td><td>0.424</td><td>0.487</td><td>0.072</td><td>0.960</td></tr>
<tr><th scope="row">Geo4D</th><td>0.177</td><td>0.737</td><td>0.098</td><td class="sec">0.912</td><td>0.100</td><td>0.897</td><td>0.310</td><td>0.554</td><td>0.067</td><td>0.966</td></tr>
<tr><th scope="row">Gen3R</th><td>0.238</td><td>0.620</td><td>0.124</td><td>0.867</td><td>0.104</td><td>0.888</td><td>0.347</td><td>0.510</td><td>0.127</td><td>0.847</td></tr>
<tr><th scope="row">Aether</th><td>0.269</td><td>0.567</td><td>0.167</td><td>0.772</td><td>0.149</td><td>0.786</td><td>0.487</td><td>0.424</td><td>0.138</td><td>0.849</td></tr>
<tr><th scope="row">Marigold</th><td>0.169</td><td>0.733</td><td>0.106</td><td>0.890</td><td class="best">0.087</td><td class="sec">0.921</td><td>0.345</td><td>0.527</td><td>0.071</td><td>0.970</td></tr>
<tr><th scope="row">UGG</th><td class="sec">0.101</td><td class="sec">0.884</td><td class="sec">0.087</td><td class="best">0.916</td><td>0.099</td><td>0.917</td><td class="sec">0.258</td><td class="sec">0.660</td><td class="sec">0.049</td><td class="sec">0.977</td></tr>
<tr><th scope="row">UGG (TTS)</th><td class="best">0.082</td><td class="best">0.917</td><td class="best">0.086</td><td class="best">0.916</td><td class="best">0.087</td><td class="best">0.934</td><td class="best">0.219</td><td class="best">0.678</td><td class="best">0.047</td><td class="best">0.979</td></tr>
<tr class="oracle"><th scope="row">UGG (Oracle; GT diagnostic)</th><td>0.055</td><td>0.966</td><td>0.075</td><td>0.923</td><td>0.068</td><td>0.946</td><td>0.177</td><td>0.781</td><td>0.043</td><td>0.979</td></tr>
</tbody>
</table></div>

</details>

## 6 · Sampling a prediction, then checking it against the images

Sparse observations can admit several plausible scene explanations. A generative model can represent alternatives, so we can spend additional inference compute drawing candidates from different noise initializations. In our experiments, we sample **16 candidates** and select one for each scene.

The difficulty is deciding which candidate to trust without the correct depth or camera pose. We use a verifier built around **RAFT optical flow** <d-cite key="teed2020raft"></d-cite>: correspondences describing where pixels move between real frames. It checks whether a candidate's geometry explains that observed motion. Depth selection also uses RGB agreement and, in the dense setting, DINOv2 image features <d-cite key="oquab2023dinov2"></d-cite>. No ground-truth depth or pose, or depth or pose foundation model, supplies the verification target.

First, the verifier checks flow in both directions and removes correspondences whose round trip is inconsistent. It also adapts the spacing between frames to the measured motion. Adjacent frames can have too little parallax to constrain depth; widely separated frames can have too much occlusion for reliable matching. Choosing pairs by pixel displacement gives the geometric check more useful evidence.

**Checking depth.** Camera rotation moves pixels even for very distant surfaces. Translation adds parallax, whose magnitude depends on depth. We estimate a relative camera from the image correspondences and use that same camera for every depth candidate. After accounting for rotation and fitting the candidate's inverse-depth scale and shift, we ask how well its depth explains the remaining motion. Using a shared camera helps separate errors in depth from errors in predicted camera pose.

**Checking pose.** For each candidate, we compare its predicted relative translation direction with the direction estimated from the images, using that candidate's intrinsics. The selected pose rule averages these angular disagreements over several frame pairs. It evaluates each candidate against image evidence, rather than rewarding agreement with the other predictions.

In the sparse setting, **the model receives the two endpoint images, while the verifier also uses real neighbouring RGB frames from the clip.** Depth is checked at the scored endpoints against nearby images; pose is checked along the candidate's predicted trajectory. This lets us generate hypotheses from sparse inputs and use additional visual evidence to choose among them, without requiring depth or pose labels.

| Task | Evidence and selected readout | Frame pairs |
| --- | --- | --- |
| Sparse depth | Parallax fit residual + perpendicular epipolar residual (`corr + epi`) | 8 endpoint-to-neighbour pairs; 16 px motion target |
| Dense depth | Parallax fit residual + DINOv2 feature warp residual + RGB warp residual (`corr + feat + photo`) | 12 adaptive pairs; 16 px target |
| Sparse pose | Translation-direction angle (`t`) | 12 adaptive pairs along the predicted trajectory; 16 px target |
| Dense pose | Translation-direction angle (`t`) | 32 directed pairs from 16 bidirectional anchors; 32 px target |


The verifier selects one complete candidate for each scene. Depth scores are combined through weighted ranks across frame pairs; pose uses the ordering of its translation-direction score. Each candidate is checked against the images rather than a consensus prediction from the pool. We assess the choices of readouts and pair recipes by leaving one benchmark out, choosing on the others, and evaluating on the held-out benchmark. The sparse rules are the same in every fold.

The largest gains appear in sparse prediction. RealEstate10K tAE falls from **8.260° to 3.475°**, and Sintel tAE from **35.479° to 18.733°**. Sparse ScanNet++ depth Abs Rel falls from **0.101 to 0.082**; sparse Sintel falls from **0.258 to 0.219**. Dense gains are generally smaller. The verifier is a proxy for geometric accuracy, so improvements are not uniform across every metric, but the sparse results show that the model generates useful alternatives and that image evidence can help identify better ones.

The tables also include an **oracle** diagnostic: ground-truth-based selection from the same candidate pool. The gap between the verifier and oracle shows room to make better use of the model's candidate diversity. This gives us two directions for improving prediction: learn a stronger scene distribution, and develop better ways to check its samples.

<details markdown="1">
<summary>How the verifier computes and combines its evidence</summary>

**Correspondences and pair spacing.** RAFT-Large is run in both directions. A correspondence from $x$ to $x'=x+F_{ij}(x)$ is retained only if $x'$ is in bounds and

$$
\|F_{ij}(x)+F_{ji}(x')\|^2\leq 0.01\big(\|F_{ij}(x)\|^2+\|F_{ji}(x')\|^2\big)+0.5.
$$

The consistency check filters unreliable correspondences before geometric fitting. For an anchor $i$, the gap is $\operatorname{clip}(\operatorname{round}(\tau/\operatorname{median}_x\|F_{i,i+1}(x)\|),1,12)$. The target $\tau$ is 16 px except for dense pose, which uses 32 px. It is shared across benchmarks within each protocol. Sparse depth pairs each endpoint with real interior neighbours at 1, 1.5, 2, and 2.5 times its adaptive gap. The verifier processes images at 288 × 384 and evaluates residuals on a stride-2 pixel grid.

**Depth evidence.** A five-point essential-matrix estimate uses up to 6,000 consistent correspondences, RANSAC probability 0.999, and a 1 px threshold, followed by pose recovery. The resulting unit-translation camera is shared across depth candidates. In the current implementation, this estimate is cached using the intrinsics of the first candidate processed. Candidate depths are normalized in inverse depth, and a two-parameter affine gauge is fitted by texture-weighted least squares. Pixels require more than 0.5 px of predicted parallax; the fit retains pixels within the 80th percentile of the perpendicular parallax residual. These filters reduce contamination from motion not explained by the camera model. The `corr` readout measures the texture-weighted along-parallax discrepancy; `epi` measures the texture-weighted perpendicular residual over valid pixels. `photo` is the mean absolute RGB difference under the gauged warp, and `feat` is mean cosine distance between DINOv2 ViT-S/14 features under that warp.

**Depth aggregation.** Within each scene and pair, each named readout is z-normalized across candidates. The normalized readouts are combined with equal weight, then candidates are ranked within the pair. Pair weights use the interquartile spread across candidates (`disc`); pairs that cannot distinguish candidates carry little weight. Weighted mean rank gives the final candidate score. This uses pool statistics for normalization, discrimination weighting, and ranking, but does not use a consensus depth or camera as geometric evidence.

**Pose evidence and aggregation.** For candidate camera-to-world transforms, the relative camera from $i$ to $j$ is $T_p=\mathrm{c2w}_j^{-1}\mathrm{c2w}_i$. The image-derived relative pose is estimated with that candidate's own intrinsics. Translation angle `t`, rotation angle `r`, and the median Sampson residual `samp` were considered; the selected rules use only `t`. Pair angles are averaged per candidate, then candidates are ranked. With a single readout, ranking preserves its ordering. Sparse trajectory pairs without a saved candidate camera are skipped; the method does not invent missing poses.

**Rule selection.** Pair recipes, readout combinations, and motion targets were assessed using leave-one-benchmark-out selection within each task and protocol. The held-out benchmark is excluded when choosing its rule. Sparse depth and sparse pose select the same rule in every fold. The table above gives the final task-level rules. Ground-truth metrics are used for this offline rule-selection study and for evaluation, not as inputs when scoring a new scene.

**Abstention.** If no usable pair remains for a scene, the verifier abstains. Its reported performance for that scene is the expected metric of a random draw from the candidate pool, rather than a ground-truth-based fallback choice.

</details>

<!-- VIDEO-SEARCH: Place a candidate-selection demonstration here. Show actual verifier scores and the selected candidate; reserve GT errors for an explicitly labelled evaluation reveal. -->

## 7 · What this changes, and what remains difficult

The results suggest that generation and geometric prediction can reinforce each other when they are learned as parts of the same scene model. Geometry helps constrain visual generation. Learning appearance and camera motion can improve depth, including through training examples without depth labels. A generative formulation also provides alternative predictions that can be checked at inference time.

The current model operates on short clips at modest resolution, and repeated sampling adds inference cost. Specialized systems remain stronger in some settings, and the gains from search depend on the quality of the visual evidence. These are useful directions for extending the approach to longer videos, larger models, and more effective verification.

A useful next step is to improve how candidates are checked, and eventually use that evidence during denoising rather than only after a full completion. The central idea is already useful: appearance, depth, and camera motion can be learned together, with the observed parts of a scene determining which question the model answers.

*This post accompanies UGG: Unified Generative Geometry Model, by Amani Kiruga, Zhiyi Li, Ruojin Cai, Hansen Lillemark, Xiaoshan Wu, Ayush Tewari, and Yilun Du. The project used computing resources at the Kempner Institute at Harvard University.*

<!-- RELEASE-LINKS: Add the public preprint and final project-page URLs when the coordinated release is ready. -->


<script>
(function() {
  var R4D_BASE = "{{ '/assets/4d-world-model/results' | relative_url }}";
  var pad = function(n) { return String(n).padStart(5, '0'); };

  var CFG = {
    nvs: {
      datasets: {
        're10k': { label: 'RealEstate10K', samples: [15, 16, 20, 39, 41] },
        'dl3dv': { label: 'DL3DV-10K', samples: [1, 5, 9] },
        'spatialvid_nvs': { label: 'SpatialVID', samples: [1, 5, 9] }
      },
      render: renderNvs
    },
    pose: {
      datasets: {
        'dl3dv': { label: 'DL3DV-10K', samples: [6, 12, 16, 17] },
        'scannetpp': { label: 'ScanNet++', samples: [4, 6] }
      },
      render: renderPose
    },
    depth: {
      datasets: {
        'dl3dv': { label: 'DL3DV-10K', samples: [6, 12, 16, 17] },
        'scannetpp': { label: 'ScanNet++', samples: [4, 6] }
      },
      render: renderDepth
    }
  };

  function renderNvs(ds, sampleNum) {
    var sid = 'sample_' + pad(sampleNum);
    var dir = R4D_BASE + '/' + ds;
    return ''
      + '<div style="display:grid; grid-template-columns: minmax(140px,1fr) 3fr; gap:10px; align-items:start;">'
      +   '<div>'
      +     '<div style="display:grid; grid-template-columns:1fr 1fr; gap:3px;">'
      +       '<div class="r4d-cell"><img src="' + dir + '/img_start_end/' + sid + '/gt_start.png"><div class="r4d-cell-label">First</div></div>'
      +       '<div class="r4d-cell"><img src="' + dir + '/img_start_end/' + sid + '/gt_end.png"><div class="r4d-cell-label">Last</div></div>'
      +     '</div>'
      +     '<div class="r4d-cell" style="margin-top:6px;"><img src="' + dir + '/viz_cam/' + sid + '/gt_frustum_wireframe_cropped.png"><div class="r4d-cell-label">Trajectory</div></div>'
      +   '</div>'
      +   '<div style="display:grid; grid-template-columns:repeat(5,1fr); gap:3px;">'
      +     '<div class="r4d-cell"><video src="' + dir + '/ours_final_nvs/' + sid + '/gt_rgb.mp4" loop muted playsinline preload="none"></video><div class="r4d-cell-label">GT</div></div>'
      +     '<div class="r4d-cell"><video src="' + dir + '/ours_final_nvs/' + sid + '/pred_rgb.mp4" loop muted playsinline preload="none"></video><div class="r4d-cell-label ours">Ours</div></div>'
      +     '<div class="r4d-cell"><video src="' + dir + '/seva_nvs/' + sid + '/samples-rgb.mp4" loop muted playsinline preload="none"></video><div class="r4d-cell-label">SEVA</div></div>'
      +     '<div class="r4d-cell"><video src="' + dir + '/wan_flf_nvs/' + sid + '/pred_rgb.mp4" loop muted playsinline preload="none"></video><div class="r4d-cell-label">Wan2.1-FLF</div></div>'
      +     '<div class="r4d-cell"><video src="' + dir + '/dfot_nvs/' + sid + '/pred_rgb.mp4" loop muted playsinline preload="none"></video><div class="r4d-cell-label">DFoT</div></div>'
      +   '</div>'
      + '</div>';
  }

  function renderPose(ds, sampleNum) {
    var sid = 'sample_' + pad(sampleNum);
    var dir = R4D_BASE + '/' + ds + '/viz_cam/' + sid;
    return ''
      + '<div style="display:grid; grid-template-columns:repeat(4,1fr); gap:8px;">'
      +   '<div class="r4d-cell"><img src="' + dir + '/gt_frustum_wireframe_cropped.png"><div class="r4d-cell-label">GT</div></div>'
      +   '<div class="r4d-cell"><img src="' + dir + '/ours_frustum_wireframe_cropped.png"><div class="r4d-cell-label ours">Ours</div></div>'
      +   '<div class="r4d-cell"><img src="' + dir + '/geo4d_raw_wxyz_frustum_wireframe_cropped.png"><div class="r4d-cell-label">Geo4D</div></div>'
      +   '<div class="r4d-cell"><img src="' + dir + '/raydiffusion_frustum_wireframe_cropped.png"><div class="r4d-cell-label">RayDiffusion</div></div>'
      + '</div>';
  }

  function renderDepth(ds, sampleNum) {
    var sid = 'sample_' + pad(sampleNum);
    var dir = R4D_BASE + '/' + ds + '/viz_depth/' + sid;
    return ''
      + '<div style="display:grid; grid-template-columns:repeat(5,1fr); gap:4px;">'
      +   '<div class="r4d-cell"><video src="' + dir + '/gt_rgb_8fps.mp4" loop muted playsinline preload="none"></video><div class="r4d-cell-label">RGB</div></div>'
      +   '<div class="r4d-cell"><video src="' + dir + '/gt_depth.mp4" loop muted playsinline preload="none"></video><div class="r4d-cell-label">GT depth</div></div>'
      +   '<div class="r4d-cell"><video src="' + dir + '/ours_depth.mp4" loop muted playsinline preload="none"></video><div class="r4d-cell-label ours">Ours</div></div>'
      +   '<div class="r4d-cell"><video src="' + dir + '/geo4d_depth.mp4" loop muted playsinline preload="none"></video><div class="r4d-cell-label">Geo4D</div></div>'
      +   '<div class="r4d-cell"><video src="' + dir + '/chronodepth_depth.mp4" loop muted playsinline preload="none"></video><div class="r4d-cell-label">ChronoDepth</div></div>'
      + '</div>';
  }

  function initCarousel(carouselEl) {
    var kind = carouselEl.getAttribute('data-r4d');
    var cfg = CFG[kind];
    if (!cfg) return;
    var dsNames = Object.keys(cfg.datasets);
    var state = { ds: dsNames[0], idx: 0 };

    var tabsHtml = dsNames.map(function(ds, i) {
      return '<button class="r4d-tab' + (i === 0 ? ' active' : '') + '" data-ds="' + ds + '">' + cfg.datasets[ds].label + '</button>';
    }).join('');

    carouselEl.innerHTML = ''
      + '<div class="r4d-head">'
      +   '<div class="r4d-tabs">' + tabsHtml + '</div>'
      +   '<div class="r4d-nav">'
      +     '<button class="r4d-btn r4d-prev">&laquo; Prev</button>'
      +     '<span class="r4d-counter">1 / 1</span>'
      +     '<button class="r4d-btn r4d-next">Next &raquo;</button>'
      +   '</div>'
      + '</div>'
      + '<div class="r4d-slides"></div>';

    var slidesEl = carouselEl.querySelector('.r4d-slides');
    var counterEl = carouselEl.querySelector('.r4d-counter');

    function buildSlides(ds) {
      var samples = cfg.datasets[ds].samples;
      slidesEl.innerHTML = samples.map(function(s, i) {
        return '<div class="r4d-slide' + (i === 0 ? ' active' : '') + '" data-i="' + i + '">' + cfg.render(ds, s) + '</div>';
      }).join('');
      activateVideos();
    }

    function activateVideos() {
      var allVids = slidesEl.querySelectorAll('video');
      allVids.forEach(function(v) {
        try { v.pause(); } catch(e){}
        v.setAttribute('preload', 'none');
      });
      var active = slidesEl.querySelector('.r4d-slide.active');
      if (!active) return;
      var vids = Array.prototype.slice.call(active.querySelectorAll('video'));
      vids.forEach(function(v) {
        v.setAttribute('preload', 'auto');
        try { v.load(); } catch(e){}
        try { v.currentTime = 0; } catch(e){}
        var p = v.play();
        if (p && typeof p.catch === 'function') p.catch(function(){});
      });
      if (vids.length >= 2) {
        var master = vids[0];
        var followers = vids.slice(1);
        master.onplay = function() { followers.forEach(function(v){ v.play().catch(function(){}); }); };
        master.onpause = function() { followers.forEach(function(v){ v.pause(); }); };
      }
    }

    function updateCounter() {
      var total = cfg.datasets[state.ds].samples.length;
      counterEl.textContent = (state.idx + 1) + ' / ' + total;
    }

    function show(idx) {
      var slides = slidesEl.querySelectorAll('.r4d-slide');
      var total = slides.length;
      idx = ((idx % total) + total) % total;
      slides.forEach(function(s, i) { s.classList.toggle('active', i === idx); });
      state.idx = idx;
      updateCounter();
      activateVideos();
    }

    function setDs(ds) {
      state.ds = ds;
      state.idx = 0;
      carouselEl.querySelectorAll('.r4d-tab').forEach(function(t) {
        t.classList.toggle('active', t.getAttribute('data-ds') === ds);
      });
      buildSlides(ds);
      updateCounter();
    }

    carouselEl.querySelectorAll('.r4d-tab').forEach(function(t) {
      t.addEventListener('click', function() { setDs(t.getAttribute('data-ds')); });
    });
    carouselEl.querySelector('.r4d-prev').addEventListener('click', function() { show(state.idx - 1); });
    carouselEl.querySelector('.r4d-next').addEventListener('click', function() { show(state.idx + 1); });

    buildSlides(state.ds);
    updateCounter();
  }

  function init() {
    document.querySelectorAll('.r4d-carousel').forEach(initCarousel);
  }

  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', init);
  } else {
    init();
  }
})();
</script>

