---
layout: distill
title: "one model, four dimensions: generating and predicting scenes together"
description: "RGB video, depth, and camera pose are three views of the same 4D scene. Treating them as missing-modality completion in a single diffusion model means generation and prediction stop being separate problems."
date: 2025-12-11
featured: true
tags: research
categories:

authors:
  - name: Amani Kiruga
    url: "https://amanikiruga.github.io"
    affiliations:
      name: MIT &middot; Harvard
  - name: collaborators
    url: ""
    affiliations:
      name: "4D World Model Project"

bibliography: 2025-12-11-4d-world-model.bib

toc:
  - name: The setup
  - name: One representation for appearance and geometry
  - name: A single trick called diffusion forcing
  - name: What you get for free
  - name: Does it work?
    subsections:
      - name: Camera-conditioned generation
      - name: Camera pose from video
      - name: Video depth
      - name: Search at inference time
  - name: Where this is going
---

## The setup

A single image, or even a short video clip, only ever shows you part of a scene. If you want a model to *understand* what it is looking at, "understanding" has to include things the camera didn't capture: the unseen sides of objects, where the camera was, how far away each surface is, and how all of that will evolve a few seconds later.

The field has, until now, mostly split that job in two. **Feedforward predictors** estimate depth, pose, and 3D structure directly from pixels --- fast, often accurate, but they give you one deterministic answer that gets brittle when observations are sparse. **Generative video models** hallucinate beautifully but have no idea what a meter is. Bolting them together pipeline-style works only when one stage is willing to accept the other's output as gospel, and that is rarely a good assumption.

The point of this project is small and stubborn: **generation and prediction are the same problem.** Both are conditional completion of a partially observed 4D scene. The only thing that changes between "make a video from a text prompt" and "estimate camera pose from a video" is *which tokens you happen to be holding fixed.*

<div class="l-page">
  <figure>
    <img src="{{ '/assets/img/4d-world-model/teaser2.png' | relative_url }}" alt="One model, three modalities, many tasks."/>
    <figcaption>
      <b>One model, many queries.</b> Given any partial observation --- a text prompt, a single image, an RGB sequence, camera poses, or a depth map --- the same model fills in the rest. Camera-controlled generation, depth estimation, and pose recovery all fall out of the same denoising objective.
    </figcaption>
  </figure>
</div>

## One representation for appearance and geometry

A frame of the world, in this model, has three spatially aligned channels:

- **RGB** $I_i$ --- appearance.
- **Depth** $D_i$ --- visible 3D structure.
- **Camera rays** $R_i$ --- viewpoint, written as a *dense ray map* with a ray direction $\mathbf{d}_i(u,v)$ and a Plücker moment $\mathbf{m}_i(u,v) = \mathbf{o}_i \times \mathbf{d}_i(u,v)$ at every pixel.

Why bother turning camera pose into an image? Because once it is image-shaped, you can token-ize it the same way you token-ize RGB and depth, and a transformer can mix the three through ordinary self-attention. No bespoke camera encoder, no awkward cross-attention scheme --- camera geometry just becomes another channel in the same latent grid.

A shared scale-and-shift normalization on depth and camera pose makes cross-modal constraints (like reprojection error) numerically well-defined, which matters later when we want to *search* over candidate scenes.

## A single trick called diffusion forcing

Once everything is tokens in a shared sequence, training is one line of intuition: **observed tokens get zero noise; missing tokens get full noise.** A standard diffusion transformer denoises the noisy ones, conditioned on the clean ones.

<div class="l-body">
  <figure>
    <img src="{{ '/assets/img/4d-world-model/diffusion_forcing.png' | relative_url }}" alt="Diffusion forcing: per-token noise levels."/>
    <figcaption>
      Per-token noise levels turn one model into many. Clean = "you observed this." Noisy = "fill this in." The pattern decides the task.
    </figcaption>
  </figure>
</div>

So *one architecture* covers all of these by choosing a different mask:

| Mask pattern | Task |
| --- | --- |
| first RGB frame clean, rest noisy | text/image-to-video |
| first + last RGB + all rays clean | camera-controlled interpolation |
| all RGB clean, depth + rays noisy | depth + pose estimation |
| all depth clean, RGB + rays noisy | depth-conditioned generation |

The model never sees a "task label." It sees noise levels.

<div class="l-page">
  <figure>
    <img src="{{ '/assets/img/4d-world-model/method2.png' | relative_url }}" alt="Generative-predictive transformer architecture."/>
    <figcaption>
      <b>The architecture.</b> RGB and depth go through a VAE encoder; ray maps are projected directly; everything is processed by a shared multimodal transformer with modality-specific projections and spatiotemporal RoPE. Output tokens decode back to RGB, depth, and dense ray maps, which convert to camera parameters.
    </figcaption>
  </figure>
</div>

## What you get for free

Two practical consequences fall out of the formulation, and they are the parts I actually find surprising:

<aside class="l-gutter"><b>Heterogeneous data.</b> The web has billions of videos with no depth or pose, plenty with pose only, and a much smaller pile with both. Most pipelines have to throw away examples that don't have every modality. Diffusion forcing doesn't.</aside>

1. **Arbitrary conditioning at inference.** Hand the model whatever you have at test time --- a prompt, a frame, a frame plus camera intrinsics, a sequence plus depth --- and ask for whatever you want. Same weights.

2. **Heterogeneous supervision at training.** The loss only applies to *available* target modalities. So an RGB-only YouTube clip and an RGB+depth+pose synthetic clip can train the same model side by side. No alignment surgery, no separate heads.

## Does it work?

The model is a 1.3B-parameter diffusion transformer initialized from Wan2.1-T2V<d-cite key="wan2025wan"></d-cite>, fine-tuned on a mixture of DL3DV-10K<d-cite key="ling2024dl3dv"></d-cite>, RealEstate10K<d-cite key="zhou2018realestate"></d-cite>, ScanNet++, SpatialVID, AgiBot World, and a few synthetic sets. 50-frame clips at 240&times;320, 10 fps, four H200s.

### Camera-conditioned generation

Given the first and last frame plus a target camera trajectory, the model fills in the intermediate video. It's competitive with specialized novel-view-synthesis systems on perceptual quality (best or second-best LPIPS across RealEstate10K, SpatialVID, DL3DV, DL3DV-Eval), and noticeably better than first-/last-frame generative baselines like Wan2.1-FLF or DFoT<d-cite key="song2025dfot"></d-cite> at staying on the requested trajectory. SEVA<d-cite key="zhou2025seva"></d-cite>, a task-specific NVS model, still wins on pixel-aligned PSNR/SSIM --- the tradeoff for spending capacity on appearance *and* geometry.

### Camera pose from video

Predict the ray map, convert to camera parameters, compare against ground truth. In the **dense** setting (full video observed), our model improves ATE and translation RPE on all four evaluation datasets over both Geo4D<d-cite key="jiang2025geo4d"></d-cite> and RayDiffusion<d-cite key="chen2024raydiffusion"></d-cite>.

<div class="fake-img l-body">
  <div class="row" style="display:flex; gap: 8px; align-items: stretch;">
    <div style="flex:2;">
      <img src="{{ '/assets/img/4d-world-model/pose_input.png' | relative_url }}" alt="Input RGB sequence." style="width:100%; height:auto;"/>
      <div class="caption" style="font-size: 0.85em; text-align:center;">input video (DL3DV)</div>
    </div>
    <div style="flex:1;">
      <img src="{{ '/assets/img/4d-world-model/pose_gt.png' | relative_url }}" alt="GT trajectory." style="width:100%; height:auto;"/>
      <div class="caption" style="font-size: 0.85em; text-align:center;">ground truth</div>
    </div>
    <div style="flex:1;">
      <img src="{{ '/assets/img/4d-world-model/pose_ours.png' | relative_url }}" alt="Predicted trajectory." style="width:100%; height:auto;"/>
      <div class="caption" style="font-size: 0.85em; text-align:center;">ours</div>
    </div>
  </div>
</div>

<aside class="l-gutter"><b>Headline number.</b><br/>ATE on DL3DV-10K:<br/>Geo4D &rarr; <b>1.868</b><br/>Ours &rarr; <b>1.160</b><br/><br/>~38% lower trajectory error, same model that also does generation.</aside>

The **sparse** setting is the more interesting one. With only the first and last frames, local feature matching can't help you --- the model has to fall back on a learned prior over plausible trajectories between two views. It still produces reasonable cameras, which I read as evidence that the joint RGB+pose training has actually learned a *world prior* and not just a fancy SfM front-end.

### Video depth

Same model, different mask: RGB clean, depth noisy. Across Virtual KITTI2, DL3DV-10K, ScanNet++, and 7-Scenes the model is competitive with specialized video depth methods like ChronoDepth<d-cite key="shao2025chronodepth"></d-cite> and Geo4D, with the best $\delta < 1.25$ on 7-Scenes. In the sparse two-image regime it outperforms both on several metrics.

<div class="l-page">
  <figure>
    <div style="display: grid; grid-template-columns: repeat(5, 1fr); gap: 4px;">
      <div><img src="{{ '/assets/img/4d-world-model/depth_rgb.png' | relative_url }}" alt="RGB"/></div>
      <div><img src="{{ '/assets/img/4d-world-model/depth_gt.png' | relative_url }}" alt="GT depth"/></div>
      <div><img src="{{ '/assets/img/4d-world-model/depth_ours.png' | relative_url }}" alt="Ours"/></div>
      <div><img src="{{ '/assets/img/4d-world-model/depth_geo4d.png' | relative_url }}" alt="Geo4D"/></div>
      <div><img src="{{ '/assets/img/4d-world-model/depth_chrono.png' | relative_url }}" alt="ChronoDepth"/></div>
    </div>
    <figcaption style="text-align:center;">RGB &middot; GT &middot; ours &middot; Geo4D &middot; ChronoDepth (DL3DV sample).</figcaption>
  </figure>
</div>

### Search at inference time

Here's a thing you can do with a generative model that you can't do with a feedforward predictor: **draw several samples, score them, keep the best.** Because RGB, depth, and rays are sampled jointly, you can score a sample by re-projecting it through its own predicted geometry and measuring photometric consistency against the observations. That's a cheap, parameter-free posterior approximation.

<div class="l-body">
  <figure>
    <img src="{{ '/assets/img/4d-world-model/test-time-k-dl3dv.png' | relative_url }}" alt="Test-time search vs. number of samples."/>
    <figcaption>
      Pose and depth metrics improve monotonically with the number of candidate samples on DL3DV-10K. No retraining, no extra parameters --- just more inference compute spent in the right place.
    </figcaption>
  </figure>
</div>

This is the part that I think is philosophically the most fun. Perception as inference<d-cite key="kersten2004object"></d-cite><d-cite key="battaglia2013simulation"></d-cite> --- the idea that vision is generating-and-checking against an internal world model --- has been in cognitive science forever. Diffusion plus a joint scene representation gives you the simplest possible computational instance of it: you literally generate scenes and pick the one whose geometry best explains the pixels.

<details>
<summary>Full appendix tables (PSNR/SSIM/LPIPS, AbsRel/RMSE/$\delta$, ATE/RPE per dataset and method)</summary>
<p>Coming once the paper is on arXiv --- this post is the short version.</p>
</details>

## Where this is going

A few honest limitations: specialized novel-view models still beat us on pixel-aligned metrics; the test-time scorer is RGB reprojection, which fails on textureless or specular regions; and 240&times;320 is not yet the resolution at which video models impress people. None of these feel fundamental --- they feel like dials to turn.

What does feel structural is the formulation: once you stop thinking of "generation" and "prediction" as different *capabilities* and start thinking of them as different *conditional queries on the same joint distribution*, a lot of design decisions get easier. There's nothing left to choose between a video model and a perception model. There's just one model and a noise mask.

---

<small><i>If you want the technical version: the paper is on arXiv with full tables, the appendix, and training details. This post compresses ~9 pages of dense methods prose into the part I'd want to read first.</i></small>
