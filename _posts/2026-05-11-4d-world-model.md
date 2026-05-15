---
layout: distill
title: "Unified Generative-Predictive Modeling for 4D Scene Understanding"
description: "A single diffusion transformer treats RGB video, depth, and camera rays as symmetric modalities, casting visual generation and geometric prediction as the same conditional-completion problem. The result is one model that does camera-controlled video synthesis, depth, and pose, and improves itself by drawing more samples at inference time."
date: 2026-05-11
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
  - name: 1 &middot; Why unify generation and prediction?
  - name: 2 &middot; Background and prior work
  - name: 3 &middot; Method
    subsections:
      - name: 3.1 A shared 4D scene representation
      - name: 3.2 Mixture-of-Transformers backbone
      - name: 3.3 Conditional completion via diffusion forcing
      - name: 3.4 Heterogeneous training
      - name: 3.5 Inference and test-time search
  - name: 4 &middot; Experiments
    subsections:
      - name: 4.1 Camera-controlled generation
      - name: 4.2 Camera pose estimation
      - name: 4.3 Depth prediction
      - name: 4.4 Test-time search
  - name: 5 &middot; Discussion, limitations, and outlook

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
---

## 1 &middot; Why unify generation and prediction?

A photo or a short video shows only one slice of a scene: visible surfaces from one viewpoint at one moment in time. Robust 4D understanding has to recover the things the camera *did not* capture: the unobserved side of an object, the camera's pose, the metric depth at every pixel, and how those quantities evolve through time. The field has historically split this job in two.

**Feedforward predictors** regress geometry directly from pixels, often with strong supervision and large datasets <d-cite key="wang2023dust3r,wang2025vggt,wang2025pi3"></d-cite>. They are fast and accurate inside their training distribution, but they collapse to a single deterministic answer and degrade when the observations are sparse, ambiguous, or out of distribution.

**Generative video models** learn rich appearance priors <d-cite key="wan2025wan"></d-cite> and can hallucinate plausible completions, but they have no native notion of metric depth or 3D structure; their outputs can look right while being geometrically inconsistent <d-cite key="ren2025gen3c,huang2025voyager"></d-cite>.

Chaining the two pipeline-style works only when one stage is willing to treat the other's output as ground truth, which rarely holds.

<div class="scientific-q">
<b>Scientific question.</b> Can a <i>single</i> model do both, by treating generation and prediction as <i>different conditional queries</i> on the same joint distribution over appearance and geometry, and does that unification produce real, measurable benefits (heterogeneous training, posterior search, robustness to sparse observations)?
</div>

RGB, depth, and camera pose are complementary observations of a dynamic scene: RGB describes appearance, depth describes visible 3D structure, and camera pose describes viewpoint. Depending on what is observed, the missing components may be future RGB frames, dense depth, or camera geometry. We answer the question above with a unified generative-predictive model that represents RGB, depth, and camera rays as spatially aligned tokens in a shared multimodal transformer and trains it via *diffusion forcing* <d-cite key="chen2024diffusionforcing"></d-cite>. The clean/noisy pattern across modality-frame groups specifies the task. The same weights then perform camera-controlled video generation, dense video depth estimation, and camera pose recovery, and crucially, the generative formulation lets us *sample* multiple 4D scene hypotheses at inference time and pick the one with the lowest cross-modal reprojection error.

<div class="l-page">
  <figure>
    <img src="{{ '/assets/img/4d-world-model/teaser2.png' | relative_url }}" alt="One model, three modalities, many tasks."/>
    <figcaption>
      <b>Figure 1 &middot; Unified conditional completion.</b> Given any partial observation, such as a text prompt, a single RGB image, an RGB sequence, camera poses (rays), or depth, the same model fills in the rest. Camera-controlled generation, depth estimation, and pose recovery are different masks on the same denoiser.
    </figcaption>
  </figure>
</div>

## 2 &middot; Background and prior work

**Geometric prediction.** Feedforward 3D / 4D regressors (DUSt3R <d-cite key="wang2023dust3r"></d-cite>, VGGT <d-cite key="wang2025vggt"></d-cite>, $\pi^3$ <d-cite key="wang2025pi3"></d-cite>) map pixels to depth, point maps, or pose in one pass. Recent diffusion-based methods adapt generative priors for specific geometric outputs: RayDiffusion <d-cite key="zhang2024raydiffusion"></d-cite> for camera rays, Marigold <d-cite key="ke2023marigold"></d-cite>, DepthCrafter <d-cite key="hu2024depthcrafter"></d-cite>, and ChronoDepth <d-cite key="shao2025chronodepth"></d-cite> for depth, and Geo4D <d-cite key="jiang2025geo4d"></d-cite> for joint 4D reconstruction. Each is still a *unidirectional* RGB &rarr; geometry map trained on strictly paired data.

**Geometry-aware video generation.** Video world models have moved from pure pixel synthesis <d-cite key="wan2025wan"></d-cite> toward injecting 3D structure for spatial consistency <d-cite key="ren2025gen3c,huang2025voyager"></d-cite> or accepting camera trajectories as control <d-cite key="he2024cameractrl,zhou2025seva"></d-cite>. The closest concurrent work (One4D <d-cite key="mione4d2025"></d-cite> and WorldReel <d-cite key="fang2025worldreel"></d-cite>) jointly emits RGB plus 4D geometric quantities, but along fixed input/output paths.

**The gap we close.** All of the above either (i) treat geometry as a byproduct of generation, or (ii) treat generation as one-way prediction. None of them treats RGB, depth, and camera geometry as *symmetric* modalities in a single any-to-any conditional distribution. Doing so requires (a) modality-symmetric tokenization, (b) a training objective that handles heterogeneous, partially annotated data, and (c) an inference procedure that exploits the joint distribution. The Mixture-of-Transformers <d-cite key="liang2024mot"></d-cite> backbone plus diffusion forcing <d-cite key="chen2024diffusionforcing,song2025dfot"></d-cite> are the right tools, and that is what we use.

Philosophically the formulation echoes *perception as inference* from cognitive science <d-cite key="kersten2004object,yuille2006vision,battaglia2013simulation"></d-cite>: scene understanding is generating-and-checking, not one-pass regression.

## 3 &middot; Method

### 3.1 A shared 4D scene representation

A 4D scene is a sequence of $N=50$ frames; each frame $i$ has an RGB image $I_i$, a depth map $D_i$, and camera parameters $C_i = (K_i, T_i)$.

To put camera geometry in the same spatial container as RGB and depth we convert $C_i$ into a **dense Plücker ray map** <d-cite key="sitzmann2021lfn"></d-cite>:

$$R_i(u,v) = \big(\,\mathbf{d}_i(u,v),\ \mathbf{m}_i(u,v)\,\big), \qquad \mathbf{m}_i(u,v) = \mathbf{o}_i \times \mathbf{d}_i(u,v),$$

where $\mathbf{o}_i$ is the camera center and $\mathbf{d}_i(u,v)$ the unit viewing direction in world coordinates. Ray direction carries orientation + intrinsics; ray moment carries the camera center up to a global scene scale. Unlike raw extrinsics or token-form camera embeddings, the per-pixel ray map places camera geometry on the same $(u,v,i)$ grid as RGB and depth, so cross-modal attention and reprojection are well-defined. Depth is reparameterized to a bounded disparity-like channel

$$\tilde{D} = \frac{2}{1 + D/s} - 1 \in [-1, 1],$$

with a single clip-level scale $s$ that is also applied to the ray moments. This shared normalization is what makes cross-modal constraints like photometric reprojection numerically well defined.

The full scene is then $\{(I_i, D_i, R_i)\}_{i=1}^{N}$: three image-shaped streams over the same $(u,v,i)$ grid.

### 3.2 Mixture-of-Transformers backbone

<div class="l-page">
  <figure>
    <img src="{{ '/assets/img/4d-world-model/method2.png' | relative_url }}" alt="Generative-predictive transformer architecture."/>
    <figcaption>
      <b>Figure 2 &middot; Architecture.</b> RGB and depth are encoded by a frozen Wan2.1 VAE; ray maps are packed directly into latents at the VAE resolution. Modality-specific patch embeddings and feed-forward layers preserve per-modality statistics, while a <i>shared multimodal self-attention</i> mixes information across appearance, depth, and rays. Per-token diffusion timesteps drive adaptive modulation.
    </figcaption>
  </figure>
</div>

We initialize from **Wan2.1-T2V-1.3B** <d-cite key="wan2025wan"></d-cite> (a 1.3B-parameter video DiT) and extend it into a four-stream Mixture-of-Transformers <d-cite key="liang2024mot"></d-cite>:

$$z = [\,z^I;\ z^D;\ z^{R_d};\ z^{R_m}\,],$$

with separate streams for RGB, depth, ray direction, and ray moment. Each clip is 50 frames at $240\times320$; after VAE encoding (stride $4\times8\times8$) each modality becomes a $14 \times 30 \times 40$ latent with 16 channels. A 3D patch size of $[1,2,2]$ yields $4{,}200$ tokens per modality and $16{,}800$ tokens per clip. Ray maps bypass the VAE; four consecutive 3-channel ray frames pack into the first 12 channels of a 16-channel latent so ray tokens stay aligned in space and time with RGB/depth.

The transformer has 30 layers, hidden 1536, FFN 8960, 12 heads. Queries / keys / values / FFNs are **stream-specific**; the attention itself is **shared** across all streams, so RGB tokens attend to depth and ray tokens (and vice versa) inside every block. Text conditioning is injected via cross-attention with Wan's text path (4096-dim embeddings, max 512 tokens). We add 3D RoPE over latent $(u,v,i)$ coordinates so the model reasons jointly over space, time, and modality.

**Initialization from the RGB checkpoint.** Extending an RGB-only video DiT to four modality streams without retraining from scratch hinges on a careful copy: RGB patch-embedding and output-head weights initialize the new depth and ray-map patch-embeddings and output heads, while attention, cross-attention, normalization, feed-forward, time-embedding, and text-conditioning weights are copied into each modality stream when shapes are compatible. This preserves the pretrained video prior and lets the new modality branches specialize during finetuning.

### 3.3 Conditional completion via diffusion forcing

Diffusion forcing <d-cite key="chen2024diffusionforcing"></d-cite> assigns each token an independent diffusion level $t_j \in [0,1]$:

$$\tilde{z}_j = (1 - t_j)\,z_j^0 + t_j\,\epsilon_j, \quad \epsilon_j \sim \mathcal{N}(0, I).$$

$t_j=0$ marks an observed context token, $t_j=1$ a fully noised target, and intermediate values a partially corrupted token. The clean/noisy *pattern* over modality-frame groups specifies the task (generation, prediction, interpolation, or anything in between) without changing the architecture.

<div class="l-body">
  <figure>
    <img src="{{ '/assets/img/4d-world-model/diffusion_forcing.png' | relative_url }}" alt="Noise as mask."/>
    <figcaption>
      <b>Figure 3 &middot; Noise as mask.</b> Different per-token noise patterns over RGB / depth / ray tokens select different tasks from one model: image-to-video, camera-controlled video, depth from RGB, pose from RGB.
    </figcaption>
  </figure>
</div>

We use a shifted flow-matching schedule $t' = \alpha t / (1 + (\alpha-1)t)$ with $\alpha=3$ and predict the velocity $v^\star = \epsilon - z_0$. The loss is

$$\mathcal{L} = \mathbb{E}_{t,\epsilon}\!\left[\sum_j a_j \lambda_{m(j)} \big\lVert \hat{v}_j - v_j^\star \big\rVert_2^2\right],$$

where $a_j \in \{0,1\}$ is the availability mask (1 if token $j$ has ground-truth supervision in this sample) and $\lambda_{m(j)}$ balances the four streams (RGB, depth, ray direction, ray moment). Five **conditioning patterns** are sampled during training, restricted by what each dataset annotates:

1. First RGB frame &rarr; rest of RGB + depth + rays (scene completion)
2. All RGB &rarr; depth + rays (RGB-to-geometry)
3. First+last RGB + all rays &rarr; middle RGB + depth (camera-controlled interpolation)
4. First RGB + all rays &rarr; rest of RGB + depth (camera-controlled extrapolation)
5. First RGB + all depth &rarr; rest of RGB + rays (depth-conditioned generation)

**Imperfect history during training.** Context tokens are not always forced to be perfectly clean: with probability $0.5$ they are clamped at $t_j=0$, and otherwise they receive a random lower noise level. Target tokens receive independently sampled higher noise levels. This teaches the model to use both perfectly clean context and imperfect context, which improves robustness at inference time when conditioning on generated or partially corrupted history.

### 3.4 Heterogeneous training

This is where the formulation pays off. Each training sample is represented in a common (RGB, depth, ray-map) format with a modality-availability mask; missing streams are filled with Gaussian placeholders, forced to maximum noise, and zeroed out of the loss. An RGB-only video clip and a fully annotated synthetic clip train the *same* model side by side; the loss simply lights up different terms. The per-modality losses are equally weighted, $\mathcal{L} = \tfrac{1}{4}(\mathcal{L}\_I + \mathcal{L}\_{R_d} + \mathcal{L}\_{R_m} + \mathcal{L}\_D)$, with availability masking applied within each term.

We train on a heterogeneous mixture of eight real, synthetic, and robotic video datasets that together cover different subsets of RGB, depth, camera pose, and text supervision (Table 1). Depth and pose for DL3DV-10K and ScanNet++ are obtained via the VIPE annotation pipeline <d-cite key="huang2025vipe"></d-cite>. Captions are native for SpatialVID and AgiBot World; we additionally annotate DL3DV-10K and RealEstate10K with scene-level description prompts; the remaining datasets train without text conditioning. 7-Scenes is held out for evaluation only.

**Table 1 &middot; Training datasets and modality availability.** ✓ = supervised; -- = not used; *(VIPE)* = annotated via the VIPE pipeline; *native* = captions provided by the dataset; *scene prompt* = scene-level descriptions we annotated.

<div style="overflow-x:auto;">
<table>
<thead>
<tr><th>Dataset</th><th>Type</th><th>RGB</th><th>Depth</th><th>Camera pose</th><th>Text</th></tr>
</thead>
<tbody>
<tr><td>RealEstate10K</td><td>real</td><td>✓</td><td>--</td><td>✓</td><td>scene prompt</td></tr>
<tr><td>DL3DV-10K</td><td>real</td><td>✓</td><td>✓ <i>(VIPE)</i></td><td>✓ <i>(VIPE)</i></td><td>scene prompt</td></tr>
<tr><td>SpatialVID</td><td>real</td><td>✓</td><td>--</td><td>✓</td><td>native</td></tr>
<tr><td>ScanNet++</td><td>real</td><td>✓</td><td>✓ <i>(VIPE)</i></td><td>✓ <i>(VIPE)</i></td><td>--</td></tr>
<tr><td>AgiBot World</td><td>robotic</td><td>✓</td><td>--</td><td>--</td><td>native</td></tr>
<tr><td>OmniWorld</td><td>synthetic</td><td>✓</td><td>✓</td><td>✓</td><td>--</td></tr>
<tr><td>Virtual KITTI 2</td><td>synthetic</td><td>✓</td><td>✓</td><td>✓</td><td>--</td></tr>
<tr><td>PointOdyssey</td><td>synthetic</td><td>✓</td><td>✓</td><td>✓</td><td>--</td></tr>
<tr><td><i>7-Scenes</i></td><td><i>eval-only</i></td><td>✓</td><td>✓</td><td>✓</td><td>--</td></tr>
</tbody>
</table>
</div>

### 3.5 Inference and test-time search

At inference, observed tokens are *clamped* to their clean values at every denoising step and assigned $t_j=0$; unknown tokens are denoised from Gaussian noise with **UniPC** (40 steps). A 50-frame clip takes $\sim$1 minute on one H100.

We additionally use **history guidance** <d-cite key="song2025dfot"></d-cite>, which is classifier-free guidance applied to the *observed context* rather than the text prompt:

$$\hat{v}_{\mathrm{HG}} = \hat{v}_{\mathcal{H}} + w_{\mathrm{hist}}\big(\hat{v}_{\mathcal{H}} - \hat{v}_{\varnothing}\big), \quad w_{\mathrm{hist}} = 2.0,$$

where $\hat{v}\_{\mathcal{H}}$ has history clamped and $\hat{v}\_{\varnothing}$ replaces it with noise.

**Test-time search (TTS).** Because the model defines a *distribution* over 4D scene completions, we can draw $L$ samples from different seeds with the same clamped context, giving $L$ joint hypotheses $\hat{S}^{(\ell)} = \{(\hat{I}_i^{(\ell)}, \hat{D}_i^{(\ell)}, \hat{C}_i^{(\ell)})\}$. We then score each candidate by **cross-frame photometric reprojection** using its *own* predicted depth and camera geometry:

$$e^{(\ell)} = \frac{1}{|\mathcal{P}|}\!\!\sum_{(i,j)\in\mathcal{P}}\!\!\frac{\big\lVert M_{ij}^{(\ell)} \odot (\tilde{I}_{i\to j}^{(\ell)} - \hat{I}_j^{(\ell)}) \big\rVert_1}{\lVert M_{ij}^{(\ell)} \rVert_1 + \epsilon},$$

with $\tilde{I}\_{i\to j}^{(\ell)}$ the warp from frame $i$ to frame $j$, $M\_{ij}^{(\ell)}$ the valid-pixel mask, and $\mathcal{P}$ an exhaustive pair set over a small anchor subsample. We pick $\ell^\star = \arg\min\_\ell e^{(\ell)}$. Candidates whose RGB, depth, and rays disagree under their own geometry warp badly and self-eliminate. $L=16$ throughout.

<details>
<summary><b>Implementation notes</b> (architecture, training, optimization, inference)</summary>
<p>
<b>Base &middot;</b> Wan2.1-T2V-1.3B; 30 layers, hidden 1536, FFN 8960, 12 heads; VAE stride $[4,8,8]$, patch $[1,2,2]$; ray latents bypass the VAE and pack four consecutive 3-channel ray frames into a 16-channel latent.<br/>
<b>Training &middot;</b> Conditioning patterns 1--5 in Sec.&nbsp;3.3 are sampled per batch, restricted to each dataset's available modalities. Missing modalities use Gaussian placeholders forced to the maximum noise level and are masked out of the loss.<br/>
<b>Optimization &middot;</b> AdamW, lr $1\!\times\!10^{-5}$, $(\beta_1,\beta_2)=(0.9,0.95)$, weight decay $0.05$, grad clip $5.0$, $1$k warmup, constant schedule; bf16 mixed precision; $4\times$ H200 GPUs, effective batch $16$ via grad-accum $4$, $\sim$$30$k steps; gradient checkpointing.<br/>
<b>Inference &middot;</b> UniPC sampler with $40$ denoising steps and history-guidance strength $w_{\mathrm{hist}}=2.0$. A 50-frame clip at $240\times320$ takes approximately one minute on a single H100. TTS uses $L=16$ candidates and an anchor-pair photometric reprojection scorer.
</p>
</details>

## 4 &middot; Experiments

We evaluate three tasks (camera-controlled generation, pose, depth) under **dense** (full RGB observed) and **sparse** (first+last RGB only) regimes, plus TTS on top of dense + sparse pose/depth. Baselines were chosen to test the unification claim against task-specialists.

### 4.1 Camera-controlled generation

**Setup.** First + last RGB + target camera trajectory are clamped; the model generates intermediate RGB. We evaluate PSNR/SSIM/LPIPS on RealEstate10K <d-cite key="zhou2018realestate"></d-cite>, DL3DV-10K <d-cite key="ling2024dl3dv"></d-cite>, the 55-scene DL3DV-Evaluation split, and SpatialVID.

**Baselines.** Wan2.1-FLF-14B (no camera input) <d-cite key="wan2025wan"></d-cite>, a 14B-parameter first-last-frame video generator, roughly $10\times$ larger than our 1.3B model; DFoT <d-cite key="song2025dfot"></d-cite>, a diffusion-forcing video completer evaluated with camera conditioning; SEVA <d-cite key="zhou2025seva"></d-cite>, a state-of-the-art NVS-specialist diffusion model.

<div class="l-page">
  <figure>
    <div class="r4d-carousel" data-r4d="nvs"></div>
    <figcaption><b>Figure 4 &middot; Camera-controlled generation.</b> The first/last frames and a target camera trajectory (left) are clamped; each column on the right shows a candidate completion. Ours preserves dynamic content along the requested trajectory; SEVA is biased toward static scenes; Wan2.1-FLF has no camera input and drifts on complex motion; DFoT is camera-conditioned but can distort or remove moving humans. Use the dataset tabs and prev/next buttons to browse samples.</figcaption>
  </figure>
</div>

**Table 2 &middot; Camera-controlled generation.** PSNR&uarr; / LPIPS&darr; / SSIM&uarr;. Best in **bold blue**, second in light blue.

<div style="overflow-x:auto;">
<table>
<thead>
<tr><th rowspan="2">Method</th><th colspan="3">RealEstate10K</th><th colspan="3">DL3DV-10K</th><th colspan="3">DL3DV-Eval</th><th colspan="3">SpatialVID</th></tr>
<tr><th>PSNR</th><th>LPIPS</th><th>SSIM</th><th>PSNR</th><th>LPIPS</th><th>SSIM</th><th>PSNR</th><th>LPIPS</th><th>SSIM</th><th>PSNR</th><th>LPIPS</th><th>SSIM</th></tr>
</thead>
<tbody>
<tr><td>Wan2.1-FLF</td><td>15.87</td><td>0.327</td><td>0.478</td><td>12.41</td><td>0.493</td><td>0.241</td><td>12.17</td><td>0.493</td><td>0.232</td><td>12.43</td><td>0.529</td><td>0.295</td></tr>
<tr><td>DFoT</td><td class="sec">21.23</td><td>0.178</td><td class="sec">0.680</td><td>14.15</td><td>0.438</td><td>0.368</td><td class="sec">14.33</td><td>0.365</td><td class="sec">0.328</td><td class="sec">14.71</td><td>0.408</td><td class="sec">0.389</td></tr>
<tr><td>SEVA</td><td class="best">22.77</td><td class="sec">0.164</td><td class="best">0.777</td><td class="best">18.27</td><td class="best">0.310</td><td class="best">0.615</td><td class="best">17.90</td><td class="best">0.298</td><td class="best">0.560</td><td class="best">16.72</td><td class="sec">0.348</td><td class="best">0.587</td></tr>
<tr><td><b>Ours</b></td><td>19.98</td><td class="best">0.154</td><td>0.658</td><td class="sec">15.09</td><td class="sec">0.316</td><td class="sec">0.408</td><td>13.69</td><td class="sec">0.325</td><td>0.308</td><td>14.51</td><td class="best">0.335</td><td>0.386</td></tr>
</tbody>
</table>
</div>

**Analysis.** We achieve the *best LPIPS* on RealEstate10K and SpatialVID and second-best on DL3DV, and we outperform Wan2.1-FLF on every metric on every dataset despite using roughly $10\times$ fewer parameters. SEVA wins on the pixel-aligned PSNR/SSIM; expected, since it is purpose-built for NVS. The interpretation: ray-map conditioning gives effective trajectory control under the same backbone that also predicts depth and pose; a specialist still beats us on pixel metrics, but we are *competitive* without specializing.

### 4.2 Camera pose estimation

**Setup.** RGB clamped; rays denoised; predicted ray maps are converted back to $(K, T)$. Dense: full video. Sparse: first+last frame only. Metrics: ATE, RPE$_t$, RPE$_r$ after Sim(3) Umeyama alignment, following Geo4D. Baselines: Geo4D <d-cite key="jiang2025geo4d"></d-cite>, RayDiffusion <d-cite key="zhang2024raydiffusion"></d-cite>.

<div class="l-page">
  <figure>
    <div class="r4d-carousel" data-r4d="pose"></div>
    <figcaption><b>Figure 5 &middot; Predicted camera trajectories.</b> Frustum wireframes for ground truth, our model, Geo4D, and RayDiffusion. Use the dataset tabs and prev/next buttons to browse samples.</figcaption>
  </figure>
</div>

**Table 3 &middot; Pose estimation.** ATE&darr; / RPE$_t$&darr; / RPE$_r$&darr; ($\times\!10^{-2}$).

<div style="overflow-x:auto;">
<table>
<thead>
<tr><th></th><th>Method</th><th colspan="3">RealEstate10K</th><th colspan="3">DL3DV-10K</th><th colspan="3">ScanNet++</th><th colspan="3">SpatialVID</th></tr>
<tr><th></th><th></th><th>ATE</th><th>RPE$_t$</th><th>RPE$_r$</th><th>ATE</th><th>RPE$_t$</th><th>RPE$_r$</th><th>ATE</th><th>RPE$_t$</th><th>RPE$_r$</th><th>ATE</th><th>RPE$_t$</th><th>RPE$_r$</th></tr>
</thead>
<tbody>
<tr><td rowspan="4"><i>Dense</i></td><td>RayDiffusion</td><td>5.712</td><td>7.216</td><td>11.560</td><td>5.840</td><td>9.569</td><td>35.970</td><td>2.824</td><td>4.082</td><td>53.920</td><td>6.262</td><td>8.610</td><td>31.840</td></tr>
<tr><td>Geo4D</td><td>1.103</td><td>0.559</td><td class="best">0.232</td><td>1.868</td><td>0.625</td><td class="best">0.911</td><td>1.716</td><td>0.472</td><td class="best">0.937</td><td>2.533</td><td>0.767</td><td class="best">0.577</td></tr>
<tr><td><b>Ours</b></td><td class="sec">0.593</td><td class="sec">0.222</td><td class="sec">0.952</td><td class="best">1.160</td><td class="best">0.351</td><td class="sec">2.300</td><td class="sec">1.169</td><td class="sec">0.337</td><td>2.372</td><td class="best">2.117</td><td class="sec">0.702</td><td>1.581</td></tr>
<tr><td><b>Ours (TTS)</b></td><td class="best">0.534</td><td class="best">0.220</td><td>0.966</td><td>1.208</td><td class="sec">0.356</td><td>2.381</td><td class="best">0.927</td><td class="best">0.323</td><td class="sec">2.316</td><td class="sec">2.169</td><td class="best">0.701</td><td class="sec">1.464</td></tr>
<tr><td rowspan="4"><i>Sparse</i></td><td>RayDiffusion</td><td>13.715</td><td>24.220</td><td>48.851</td><td>10.185</td><td>22.503</td><td>77.655</td><td>4.475</td><td>14.263</td><td>109.84</td><td>14.486</td><td>33.801</td><td>86.099</td></tr>
<tr><td>Geo4D</td><td>26.516</td><td>37.499</td><td>14.980</td><td>16.156</td><td>22.848</td><td>55.886</td><td>8.875</td><td class="sec">12.551</td><td>71.614</td><td>30.888</td><td>43.682</td><td>45.924</td></tr>
<tr><td><b>Ours</b></td><td class="sec">1.220</td><td class="sec">0.326</td><td class="sec">0.984</td><td class="best">2.261</td><td class="best">0.589</td><td class="best">2.938</td><td class="sec">2.229</td><td class="best">0.518</td><td class="sec">3.602</td><td class="best">3.683</td><td class="best">0.992</td><td class="best">2.055</td></tr>
<tr><td><b>Ours (TTS)</b></td><td class="best">1.088</td><td class="best">0.319</td><td class="best">0.951</td><td class="sec">2.398</td><td class="sec">0.611</td><td class="sec">3.123</td><td class="best">2.031</td><td class="best">0.518</td><td class="best">3.270</td><td class="sec">3.762</td><td class="sec">1.021</td><td class="sec">2.202</td></tr>
</tbody>
</table>
</div>

**Analysis.** *Dense:* we beat both baselines on ATE / RPE$_t$ on every dataset, e.g. DL3DV ATE 1.868 &rarr; 1.160 (38% lower) vs Geo4D. Geo4D wins on rotation RPE$_r$ where its specialized geometry head helps, but we gain on global trajectory. *Sparse:* the order-of-magnitude gap is the most informative number on this page. With only two RGB frames, both RayDiffusion and Geo4D collapse (ATE 10--30); we stay at 1--4 by falling back on the joint RGB+camera prior learned from heterogeneous data. SpatialVID sparse ATE 30.888 &rarr; 3.683 --- the kind of regime where a feedforward predictor has nothing to match locally and a *generative posterior* is genuinely the right tool.

### 4.3 Depth prediction

**Setup.** RGB clamped; depth denoised. Evaluation on Virtual KITTI 2 <d-cite key="cabon2020vkitti"></d-cite>, DL3DV-10K, ScanNet++ <d-cite key="yeshwanth2023scannetpp"></d-cite>, and 7-Scenes <d-cite key="shotton2013scene"></d-cite>; video-level scale-and-shift alignment, then Abs Rel, RMSE, $\delta\!<\!1.25$. Baselines: Geo4D, ChronoDepth <d-cite key="shao2025chronodepth"></d-cite>.

<div class="l-page">
  <figure>
    <div class="r4d-carousel" data-r4d="depth"></div>
    <figcaption><b>Figure 6 &middot; Video depth qualitative.</b> RGB &middot; GT depth &middot; <b>ours</b> &middot; Geo4D &middot; ChronoDepth. Use the dataset tabs and prev/next buttons to browse samples; ours preserves sharper object boundaries and more coherent temporal structure.</figcaption>
  </figure>
</div>

**Table 4 &middot; Depth prediction.** AbsRel&darr; / RMSE&darr; / $\delta\!<\!1.25$&uarr;.

<div style="overflow-x:auto;">
<table>
<thead>
<tr><th></th><th>Method</th><th colspan="3">Virtual KITTI 2</th><th colspan="3">DL3DV-10K</th><th colspan="3">ScanNet++</th><th colspan="3">7-Scenes</th></tr>
<tr><th></th><th></th><th>Rel</th><th>RMSE</th><th>$\delta$</th><th>Rel</th><th>RMSE</th><th>$\delta$</th><th>Rel</th><th>RMSE</th><th>$\delta$</th><th>Rel</th><th>RMSE</th><th>$\delta$</th></tr>
</thead>
<tbody>
<tr><td rowspan="4"><i>Dense</i></td><td>ChronoDepth</td><td>0.382</td><td>0.287</td><td>0.399</td><td>0.340</td><td class="best">0.161</td><td>0.509</td><td class="sec">0.185</td><td>0.208</td><td class="sec">0.796</td><td>1.443</td><td>0.294</td><td>0.381</td></tr>
<tr><td>Geo4D</td><td class="best">0.211</td><td>0.194</td><td>0.675</td><td class="best">0.304</td><td>0.206</td><td class="best">0.631</td><td class="best">0.150</td><td class="best">0.191</td><td class="best">0.863</td><td class="best">1.362</td><td class="best">0.258</td><td>0.531</td></tr>
<tr><td><b>Ours</b></td><td>0.225</td><td class="sec">0.189</td><td class="sec">0.680</td><td>0.308</td><td class="sec">0.171</td><td>0.545</td><td>0.297</td><td class="sec">0.201</td><td>0.749</td><td>1.387</td><td>0.260</td><td class="sec">0.607</td></tr>
<tr><td><b>Ours (TTS)</b></td><td class="sec">0.220</td><td class="best">0.183</td><td class="best">0.695</td><td class="best">0.304</td><td>0.175</td><td class="sec">0.570</td><td>0.251</td><td>0.218</td><td>0.736</td><td class="sec">1.385</td><td class="sec">0.259</td><td class="best">0.619</td></tr>
<tr><td rowspan="4"><i>Sparse</i></td><td>ChronoDepth</td><td>1.309</td><td class="best">0.412</td><td>0.127</td><td>0.835</td><td class="best">0.284</td><td>0.274</td><td>0.689</td><td>0.292</td><td>0.500</td><td class="sec">1.431</td><td class="sec">0.268</td><td>0.483</td></tr>
<tr><td>Geo4D</td><td>1.309</td><td class="best">0.412</td><td>0.127</td><td>0.855</td><td class="sec">0.288</td><td>0.268</td><td>0.699</td><td>0.296</td><td>0.486</td><td class="best">1.371</td><td class="best">0.223</td><td class="best">0.737</td></tr>
<tr><td><b>Ours</b></td><td class="sec">0.747</td><td>0.632</td><td class="sec">0.401</td><td class="sec">0.731</td><td>0.369</td><td class="best">0.422</td><td class="best">0.418</td><td class="best">0.272</td><td class="sec">0.655</td><td>1.451</td><td>0.274</td><td>0.486</td></tr>
<tr><td><b>Ours (TTS)</b></td><td class="best">0.716</td><td class="sec">0.573</td><td class="best">0.449</td><td class="best">0.670</td><td>0.291</td><td class="sec">0.397</td><td class="sec">0.454</td><td class="sec">0.280</td><td class="best">0.695</td><td>1.445</td><td>0.272</td><td class="sec">0.498</td></tr>
</tbody>
</table>
</div>

**Analysis.** *Dense:* we are competitive everywhere and lead on Virtual KITTI 2 RMSE and 7-Scenes $\delta$. Geo4D remains stronger on the indoor ScanNet++ split; specialist geometry models still benefit from indoor-heavy training. *Sparse:* the joint formulation wins decisively on Virtual KITTI 2, DL3DV, and ScanNet++ (AbsRel 0.689 &rarr; 0.418 on ScanNet++). Two frames are not enough for video-depth-specialists to exploit temporal context; the generative prior plus cross-modal training is.

### 4.4 Test-time search

<div class="l-body">
  <figure>
    <img src="{{ '/assets/img/4d-world-model/test-time-k-dl3dv.png' | relative_url }}" alt="TTS curves."/>
    <figcaption><b>Figure 7 &middot; Test-time search</b> on DL3DV-10K. Pose ATE and depth AbsRel improve as we sample more candidates and pick the one with the lowest RGB reprojection error. No retraining, no extra parameters: just more inference compute spent in the right place.</figcaption>
  </figure>
</div>

The TTS rows in Tables 3 and 4 are the same model, with $L=16$ samples and reprojection-based selection. TTS improves on the no-search baseline in 9/12 pose cells (dense) and 11/12 depth cells (sparse). Gains are not uniform: reprojection error is a *proxy* for geometric accuracy, so on textureless or specular regions the selector is misled. Still --- improvements at zero training cost, monotone in $L$ on DL3DV (Fig. 7), and consistent with the cognitive view of perception as generate-and-check <d-cite key="kersten2004object,yuille2006vision,battaglia2013simulation"></d-cite>.

## 5 &middot; Discussion, limitations, and outlook

The headline finding is that **one model + one mask is enough**. A 1.3B-parameter diffusion transformer with three modality streams and shared attention, trained with diffusion forcing on heterogeneous data, handles camera-controlled generation, video depth, and camera pose, and improves itself at inference time by sampling more 4D scene hypotheses. The conditional-completion formulation cleanly answers our scientific question: generation and prediction *are* the same problem, and the benefits (heterogeneous supervision, joint priors, posterior search) are concretely measurable, especially in sparse regimes.

**Implications.** (i) Web-scale RGB-only video can train geometry models without needing matched depth or pose. (ii) Sparse-view geometry (a regime where feedforward predictors collapse) becomes the *strength* of generative scene models, not a weakness. (iii) Inference compute is a usable lever for accuracy.

**Limitations.** (1) Specialist NVS models like SEVA still win on pixel-aligned PSNR/SSIM; spending capacity on geometry costs some appearance fidelity. (2) The TTS scorer is photometric reprojection, which fails on low-texture or specular surfaces; a learned multimodal consistency score would likely improve search. (3) We train at $240\times320$, 50 frames, on 4 H200s --- modest resolution and scale by current video-model standards. (4) Indoor depth still favors specialist geometry models; a better dataset balance would close that gap.

**Looking ahead.** Stronger learned consistency scorers, search *during* the denoising process (not just over seeds), larger-scale training, and embodied/robotic evaluation are the natural next moves. The structural point survives: stop thinking of "generation" and "prediction" as separate capabilities. They are different conditional queries on the same joint distribution, and a diffusion transformer can learn that distribution from messy, partially annotated video at scale.

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
