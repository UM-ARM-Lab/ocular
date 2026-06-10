+++
title = "Local Conformal Calibration of Dynamics Uncertainty from Semantic Images"
[extra]
display_title = "Local Conformal Calibration of Dynamics Uncertainty from Semantic Images"
authors = [
    {name = "Luís Marques", url = "https://marquesluis.com/"},
    {name = "Dmitry Berenson", url = "https://berenson.robotics.umich.edu/"}
]
venue = {name = "17th International Workshop on the Algorithmic Foundations of Robotics (WAFR) 2026", url = "https://algorithmic-robotics.org/"}
buttons = [
    {name = "ArXiv", url = "https://arxiv.org/abs/2605.13028"},
    {name = "PDF", url = "https://arxiv.org/pdf/2605.13028"},
    {name = "Poster", url = "poster.pdf"},
    # {name = "Code", url = "https://github.com/UM-ARM-Lab/ocular_code"}
]
katex = true
large_card = true
favicon = true
+++


<section class="ocular-hero" aria-label="SplitCP and OCULAR rollout comparison">
<p class="ocular-hero-caption">(Left) SplitCP calibrates dynamics uncertainty <em>globally</em>, possibly resulting in overconfident and/or overconservative motion. (Right) <strong>OCULAR</strong> calibrates dynamics uncertainty conditioned on velocity, action, and observation, and does not require data from the test-time environment.</p>
<video autoplay muted loop controls playsinline preload="metadata" poster="./hero_videos/ocular_splitcp_ours_hero_maps.jpg?v=20260610-terminal-success">
    <source src="./hero_videos/ocular_splitcp_ours_hero_maps.mp4?v=20260610-terminal-success" type="video/mp4">
</video>
<p class="ocular-hero-legend">The blue marker indicates the start location and the pink markers the subgoals.</p>
</section>

<section class="capture-abstract" aria-label="Abstract">
<p><span class="capture-run-in-heading">Abstract</span> We introduce <strong>O</strong>bservation-aware <strong>C</strong>onformal <strong>U</strong>ncertainty <strong>L</strong>ocal-C<strong>a</strong>lib<strong>r</strong>ation (<strong>OCULAR</strong>), a conformal prediction-based algorithm that uses perception information to provide <em>uncertainty quantification guarantees</em> for <em>unseen</em> test-time environments. While previous conformal approaches lack the ability to discriminate between state-action space regions leading to higher or lower model mismatch, and require environment-specific data, our method uses data collected from visually similar environments to provably calibrate a given linear Gaussian dynamics model of arbitrary fidelity. The prediction regions generated from <strong>OCULAR</strong> are guaranteed to contain future system states with at least a user-set likelihood, despite both aleatoric and epistemic uncertainty. Our guarantees are non-asymptotic and <em>distribution-free</em>, not requiring strong assumptions about the <em>unknown</em> real system dynamics. Our calibration procedure enables distinguishing between observation-velocity-action inputs leading to higher and lower next-state uncertainty, which is helpful for probabilistically safe planning. We validate our algorithm on a double-integrator system subject to random perturbations and significant model mismatch, using both a simplified sensor and a more realistic simulated camera. Our approach appropriately quantifies uncertainty both when in-distribution and out-of-distribution, while being comparatively <em>volume-efficient</em> to baselines requiring environment-specific data.</p>
</section>

# Problem Statement

Let `$s_t:=(p_t,v_t)$`, `$a_t$`, and `$o_t$` denote a robot's state, action, and observation, respectively. We consider stochastic systems evolving according to the *unknown* dynamics `$s_{t+1}\sim f(s_t,a_t)$`. The available approximate dynamics model `$\tilde f$` has arbitrary fidelity, with uncertainty arising from both external disturbances and model mismatch. In our experiments, `$\tilde f$` is a linear-Gaussian model.

Given access to an exchangeable dataset of robot transitions `$D_{\mathrm{cal}}:=\{(s_t,a_t,o_t,s_{t+1})_i\}_{i=1}^{n}$`, collected in environments that are different from but visually similar to the deployment environment, our goal is to construct a state-action-observation-dependent and volume-efficient prediction region `$\hat{\mathcal C}\subseteq \mathcal S$` that is guaranteed to contain the *unknown* future system state `$s_{t+1}$` with at least a user-selected likelihood `$1-\alpha \in(0,1)$`, i.e., satisfy
```
$$
\mathbb P\!\left(s_{t+1} \in \hat{\mathcal C}\right) \ge 1-\alpha.
$$
```
**OCULAR** *calibrates* the approximate dynamics model `$\tilde f$` to construct the prediction set `$\hat{\mathcal C}$` *without requiring data from the test-time environment*.

# Method: OCULAR

**OCULAR** performs local conformal calibration using robot-frame perception, body velocity, and action information. Given raw calibration samples `$X_i^{\mathrm{raw}}:=\left(s_t,a_t,o_t\right)_i$`, it maps each observation to a planar semantic footprint `$\mathrm{Project}(o_t)$` and encodes that footprint with a CAE. The processing step constructs `$X_i:=\mathrm{Process}(X_i^{\mathrm{raw}})=\left(v_t,a_t,\mathrm{Encode}(\mathrm{Project}(o_t))\right)_i$`, and a Regression Decision Tree partitions this processed input space using fitted scores. SplitCP is then performed per Decision Tree leaf, resulting in an input-dependent score threshold `$\hat q_k\in \mathbb R$`, which acts as a *local* probabilistic upper bound on motion uncertainty. The calibration data is split into two disjoint subsets `$D_{\mathrm{cal}} = D_{\mathrm{cal}}^{\mathrm{part}} \sqcup D_{\mathrm{cal}}^{\mathrm{CP}}$`.

{% figure(alt=["OCULAR offline calibration pipeline"] src=["./offline_diagram_v2.svg?v=20260610-newnew-blue"] dark_src=["./offline_diagram_v2_dark.svg?v=20260610-newnew-blue"]) %}
**Offline component of OCULAR.** 1: observations `$o_t$` from `$D_{\mathrm{cal}}^{\mathrm{part}}$` are projected into planar footprints `$\mathrm{Project}(o_t)$`. A CAE is trained to reconstruct these footprints, and the decoder is discarded. 2: All data in `$D_{\mathrm{cal}}$` starts as raw tuples `$X_i^{\mathrm{raw}}:=\left(s_t,a_t,o_t\right)_i$`; these are processed into `$X_i:=\mathrm{Process}(X_i^{\mathrm{raw}})=\left(v_t,a_t,\mathrm{Encode}(\mathrm{Project}(o_t))\right)_i$` and scored by `$R_i := r(s_{t+1,i}, \tilde{\mathcal N}_{t+1,i})$`. 3: a Decision Tree is trained on `$D_{\mathrm{cal}}^{\mathrm{part}}$` to partition the learned input space `$\mathcal{X}$` into regions of approximately constant score. 4: The holdout processed `$D_{\mathrm{cal}}^{\mathrm{CP}}$` data is fed through the DTree and scores are grouped per leaf node `$k$`. SplitCP is performed on each input-space partition `$\mathcal{X}_k$` to get an input-dependent probabilistic threshold `$\hat{q}_k$`.
{% end %}

{% figure(alt=["OCULAR online uncertainty calibration pipeline"] src=["./online_diagram_v2.svg?v=20260610-online-v3"] dark_src=["./online_diagram_v2_dark.svg?v=20260610-online-v3"]) %}
**Online component of OCULAR.** Given an estimated Gaussian at time `$t$`, a desired action `$a_t$`, and observation `$o_t$`, we create an approximate next-step Gaussian `$\tilde{\mathcal N}_{t+1}$` via the approximate model `$\tilde f$`. The current raw tuple `$(s_t,a_t,o_t)$` is processed into features `$(v_t,a_t,\mathrm{Encode}(\mathrm{Project}(o_t)))$`, and this learned representation is passed to the Decision Tree. The resulting leaf node `$\mathcal{X}_k$` has an associated `$\hat{q}_k$`, which is multiplied by a fixed constant to get `$\xi_k$`. The approximate uncertainty estimate is then calibrated by scaling its covariance by `$\xi_k$`, and the output is passed to the following planning step.
{% end %}

# Experiments
We validate **OCULAR** on a double-integrator in Isaac Sim using a floating camera providing depth and semantic segmentation images. We evaluate across three snowy T-junction environments. The white lower-friction regions are out-of-distribution (OOD) relative to the approximate linear-Gaussian model `$\tilde{f}$`, which captures the robot dynamics over the asphalt road (ID). In all experiments, **OCULAR** uses no data from the test map (e.g., for icyMain evaluation, we use system transition data collected in icyMiddle and icySide).

<div class="carousel-title">
<p>The three tested Isaac Sim environments, based on Rivermark.</p>
</div>

{{ figure(alt=["Isaac Sim icy main map flyover", "Isaac Sim icy middle map flyover", "Isaac Sim icy side map flyover"] labels=["icyMain", "icyMiddle", "icySide"] src=["./flyover_map_videos/rivermark_Tsection_icyMain_flyover_maponly_2560x1440_24fps_web_crf32.mp4", "./flyover_map_videos/rivermark_Tsection_icyMiddle_flyover_maponly_2560x1440_24fps_web_crf32.mp4", "./flyover_map_videos/rivermark_Tsection_icySide_flyover_maponly_2560x1440_24fps_web_crf32.mp4"]) }}

Using Monte Carlo propagation, we estimate the likelihood of the prediction regions `$\hat{\mathcal C}\subseteq \mathcal S$` containing the future *unknown* state `$s_{t+1}$`. This containment likelihood (i.e., coverage) is reported separately for the nominal road regions (ID relative to `$\tilde f$`) and the icy regions (OOD relative to `$\tilde f$`). Prediction region volume-efficiency is reported as a ratio relative to a linear-Gaussian oracle with access to the unknown ground-truth dynamics.

<p class="result-table-title">Test-case results across three Isaac Sim roads.</p>
<div class="result-table-wrap">
<table class="result-table" aria-label="Test-case results across three Isaac Sim roads">
    <thead>
        <tr>
            <th class="method-cell">Method</th>
            <th class="narrow-col"><span class="nowrap">Tested map</span><br><span class="nowrap"><span class="header-negation">not</span> in <code>$D_{\mathrm{cal}}$</code>?</span></th>
            <th>icySide ID</th>
            <th>icySide OOD</th>
            <th>icyMain ID</th>
            <th>icyMain OOD</th>
            <th>icyMiddle ID</th>
            <th>icyMiddle OOD</th>
        </tr>
    </thead>
    <tbody>
        <tr class="metric-section">
            <th colspan="8">Marginal coverage (%) <svg class="metric-arrow metric-arrow-up" viewBox="0 0 16 16" aria-hidden="true"><path d="M8 13V3M4.5 6.5 8 3l3.5 3.5"/></svg></th>
        </tr>
        <tr>
            <td class="method-cell">NoCP</td>
            <td class="narrow-col"><span class="status neutral">N/A</span></td>
            <td class="ok">90.0</td>
            <td class="bad">56.7</td>
            <td class="ok">90.0</td>
            <td class="bad">56.7</td>
            <td class="ok">90.0</td>
            <td class="bad">56.7</td>
        </tr>
        <tr>
            <td class="method-cell">SplitCP</td>
            <td class="narrow-col"><span class="status cross" aria-label="No">&times;</span></td>
            <td class="ok">99.5</td>
            <td class="bad">89.6</td>
            <td class="ok">100.0</td>
            <td class="ok">98.4</td>
            <td class="ok">99.1</td>
            <td class="bad">85.9</td>
        </tr>
        <tr>
            <td class="method-cell">LUCCa</td>
            <td class="narrow-col"><span class="status cross" aria-label="No">&times;</span></td>
            <td class="ok">91.1</td>
            <td class="ok">91.5</td>
            <td class="ok">90.1</td>
            <td class="ok">91.4</td>
            <td class="ok">90.1</td>
            <td class="ok">90.9</td>
        </tr>
        <tr>
            <td class="method-cell method-ours"><strong>OCULAR (ours)</strong></td>
            <td class="narrow-col"><span class="status check" aria-label="Yes">&#10003;</span></td>
            <td class="ok">91.5</td>
            <td class="ok">90.1</td>
            <td class="ok">90.4</td>
            <td class="ok">90.1</td>
            <td class="ok">91.1</td>
            <td class="ok">90.6</td>
        </tr>
        <tr class="metric-section">
            <th colspan="8">Median volume (relative to oracle) <svg class="metric-arrow metric-arrow-down" viewBox="0 0 16 16" aria-hidden="true"><path d="M8 3v10m3.5-3.5L8 13 4.5 9.5"/></svg></th>
        </tr>
        <tr>
            <td class="method-cell">NoCP</td>
            <td class="narrow-col"><span class="status neutral">N/A</span></td>
            <td>1.00</td>
            <td>0.28</td>
            <td>1.00</td>
            <td>0.28</td>
            <td>1.00</td>
            <td>0.28</td>
        </tr>
        <tr>
            <td class="method-cell">SplitCP</td>
            <td class="narrow-col"><span class="status cross" aria-label="No">&times;</span></td>
            <td>3.73</td>
            <td>1.03</td>
            <td>8.68</td>
            <td>2.40</td>
            <td>3.07</td>
            <td>0.85</td>
        </tr>
        <tr>
            <td class="method-cell">LUCCa</td>
            <td class="narrow-col"><span class="status cross" aria-label="No">&times;</span></td>
            <td>1.08</td>
            <td>1.13</td>
            <td><strong>1.02</strong></td>
            <td>1.10</td>
            <td><strong>1.02</strong></td>
            <td>1.13</td>
        </tr>
        <tr>
            <td class="method-cell method-ours"><strong>OCULAR (ours)</strong></td>
            <td class="narrow-col"><span class="status check" aria-label="Yes">&#10003;</span></td>
            <td><strong>1.03</strong></td>
            <td><strong>1.02</strong></td>
            <td><strong>1.02</strong></td>
            <td><strong>1.02</strong></td>
            <td>1.06</td>
            <td><strong>1.06</strong></td>
        </tr>
    </tbody>
</table>
</div>

<p class="table-note"><span class="bad">red</span> means coverage below 90%. Prediction region volume ratio is reported relative to an oracle using the minimum scaling factor needed to achieve 90% coverage. Each map has 4464 test transitions.</p>

**OCULAR** calibrates the approximate dynamics on both nominal and icy conditions without sacrificing volume efficiency relative to baselines using environment-specific data.

We also conduct motion planning experiments to demonstrate the utility of our method for probabilistically safe MPC under model mismatch and external perturbations.
For all methods the objective function includes a distance-to-subgoal term, a collision penalty, and a penalty for transitions with high estimated uncertainty.

<div class="inference-video-panel" data-video-picker data-video-template="./pointcamera_videos/single/{map}_episode_{episode}_{method}.mp4?v=20260609-2d-black-alpha28">
    <div class="inference-picker-controls">
        <div class="inference-picker-group" aria-label="Map">
            <span class="inference-picker-label">Map</span>
            <div class="inference-picker-options" role="group">
                <button type="button" class="inference-option active" data-video-token="map" data-video-value="icyMain" aria-pressed="true">icyMain</button>
                <button type="button" class="inference-option" data-video-token="map" data-video-value="icyMiddle" aria-pressed="false">icyMiddle</button>
                <button type="button" class="inference-option" data-video-token="map" data-video-value="icySide" aria-pressed="false">icySide</button>
            </div>
        </div>
        <div class="inference-picker-group" aria-label="Episode">
            <span class="inference-picker-label">Episode</span>
            <div class="inference-picker-options" role="group">
                <button type="button" class="inference-option active" data-video-token="episode" data-video-value="12" aria-pressed="true">13</button>
                <button type="button" class="inference-option" data-video-token="episode" data-video-value="13" aria-pressed="false">14</button>
                <button type="button" class="inference-option" data-video-token="episode" data-video-value="14" aria-pressed="false">15</button>
                <button type="button" class="inference-option" data-video-token="episode" data-video-value="15" aria-pressed="false">16</button>
                <button type="button" class="inference-option" data-video-token="episode" data-video-value="16" aria-pressed="false">17</button>
                <button type="button" class="inference-option" data-video-token="episode" data-video-value="17" aria-pressed="false">18</button>
                <button type="button" class="inference-option" data-video-token="episode" data-video-value="18" aria-pressed="false">19</button>
                <button type="button" class="inference-option" data-video-token="episode" data-video-value="19" aria-pressed="false">20</button>
                <button type="button" class="inference-option" data-video-token="episode" data-video-value="20" aria-pressed="false">21</button>
                <button type="button" class="inference-option" data-video-token="episode" data-video-value="21" aria-pressed="false">22</button>
            </div>
        </div>
        <div class="inference-picker-group" aria-label="Method">
            <span class="inference-picker-label">Method</span>
            <div class="inference-picker-options" role="group">
                <button type="button" class="inference-option" data-video-token="method" data-video-value="nocp" aria-pressed="false">NoCP</button>
                <button type="button" class="inference-option" data-video-token="method" data-video-value="split_cp" aria-pressed="false">SplitCP</button>
                <button type="button" class="inference-option" data-video-token="method" data-video-value="lucca" aria-pressed="false">LUCCa</button>
                <button type="button" class="inference-option" data-video-token="method" data-video-value="dtree_obs_samemap" aria-pressed="false">Ablation: test-map data</button>
                <button type="button" class="inference-option" data-video-token="method" data-video-value="dtree_obs_va" aria-pressed="false">Ablation: velocity/action only</button>
                <button type="button" class="inference-option active" data-video-token="method" data-video-value="dtree_obs" aria-pressed="true"><strong>OCULAR</strong></button>
            </div>
        </div>
    </div>
    <div class="inference-video-stage" data-video-stage></div>
    <p class="inference-video-note" data-video-note hidden>Video for this map/episode/method is not available yet.</p>
</div>

We observe that using the uncalibrated dynamics directly leads to gaining significant momentum over ice and hence collisions. SplitCP can produce overconfident (i.e., collisions) or overconservative (i.e., timeouts) behavior depending on the calibration data distribution. LUCCa is a baseline requiring robot location data in an inertial frame and hence needs transitions collected in the test environment. **OCULAR** moves more slowly over ice and faster over the nominal road, being both safe and efficient. By using perception information, it does *not* require data from the test environment. Below we visualize rollouts from all compared methods at once.

<div class="inference-video-panel inference-comparison-panel" data-video-picker data-video-template="./six_method_comparison/rivermark_Tsection_{map}_episode_{episode}_flyover_methods_grid_3840x1440_o3dyn_trimmed_fixedtrail_interp5_web_crf32.mp4?v=20260608-blackseams-rounded">
    <div class="inference-picker-controls">
        <div class="inference-picker-group" aria-label="Map">
            <span class="inference-picker-label">Map</span>
            <div class="inference-picker-options" role="group">
                <button type="button" class="inference-option active" data-video-token="map" data-video-value="icyMain" aria-pressed="true">icyMain</button>
                <button type="button" class="inference-option" data-video-token="map" data-video-value="icyMiddle" aria-pressed="false">icyMiddle</button>
                <button type="button" class="inference-option" data-video-token="map" data-video-value="icySide" aria-pressed="false">icySide</button>
            </div>
        </div>
        <div class="inference-picker-group" aria-label="Episode">
            <span class="inference-picker-label">Episode</span>
            <div class="inference-picker-options" role="group">
                <button type="button" class="inference-option active" data-video-token="episode" data-video-value="12" aria-pressed="true">13</button>
                <button type="button" class="inference-option" data-video-token="episode" data-video-value="13" aria-pressed="false">14</button>
                <button type="button" class="inference-option" data-video-token="episode" data-video-value="14" aria-pressed="false">15</button>
                <button type="button" class="inference-option" data-video-token="episode" data-video-value="15" aria-pressed="false">16</button>
                <button type="button" class="inference-option" data-video-token="episode" data-video-value="16" aria-pressed="false">17</button>
                <button type="button" class="inference-option" data-video-token="episode" data-video-value="17" aria-pressed="false">18</button>
                <button type="button" class="inference-option" data-video-token="episode" data-video-value="18" aria-pressed="false">19</button>
                <button type="button" class="inference-option" data-video-token="episode" data-video-value="19" aria-pressed="false">20</button>
                <button type="button" class="inference-option" data-video-token="episode" data-video-value="20" aria-pressed="false">21</button>
                <button type="button" class="inference-option" data-video-token="episode" data-video-value="21" aria-pressed="false">22</button>
            </div>
        </div>
    </div>
    <div class="inference-video-stage" data-video-stage></div>
    <p class="inference-video-note" data-video-note hidden>Video for this map/episode is not available yet.</p>
</div>

<script>
(() => {
    const setupPicker = (picker) => {
        const buttons = [...picker.querySelectorAll("[data-video-src], [data-video-token]")];
        const stage = picker.querySelector("[data-video-stage]");
        const note = picker.querySelector("[data-video-note]");
        if (!buttons.length || !stage || !note) return;

        let activeVideo = null;
        let activeSrc = "";
        const videoCache = new Map();

        const setActiveButton = (selectedButton) => {
            const token = selectedButton.dataset.videoToken;
            const groupButtons = token ? buttons.filter((button) => button.dataset.videoToken === token) : buttons;

            groupButtons.forEach((button) => {
                const isActive = button === selectedButton;
                button.classList.toggle("active", isActive);
                button.setAttribute("aria-pressed", String(isActive));
            });
        };

        const getSelectedTokens = () => {
            const tokens = {};
            picker.querySelectorAll("[data-video-token]").forEach((button) => {
                const token = button.dataset.videoToken;
                if (!token) return;

                if (button.classList.contains("active") || tokens[token] === undefined) {
                    tokens[token] = button.dataset.videoValue ?? "";
                }
            });
            return tokens;
        };

        const getSelectedSrc = () => {
            const template = picker.dataset.videoTemplate;
            if (!template) {
                const activeButton = buttons.find((button) => button.classList.contains("active")) || buttons[0];
                return activeButton?.dataset.videoSrc ?? "";
            }

            const tokens = getSelectedTokens();
            return template.replace(/\{([a-zA-Z0-9_]+)\}/g, (_, key) => tokens[key] ?? "");
        };

        const hideActiveVideo = () => {
            if (!activeVideo) return;
            activeVideo.pause();
            activeVideo.classList.remove("active");
            activeVideo.setAttribute("aria-hidden", "true");
            activeVideo = null;
            activeSrc = "";
        };

        const showVideo = (video, src) => {
            if (activeVideo && activeVideo !== video) {
                activeVideo.pause();
                activeVideo.classList.remove("active");
                activeVideo.setAttribute("aria-hidden", "true");
            }
            video.classList.add("active");
            video.removeAttribute("aria-hidden");
            activeVideo = video;
            activeSrc = src;
            note.hidden = true;
            const playPromise = video.play();
            if (playPromise !== undefined) {
                playPromise.catch(() => {});
            }
        };

        const createVideo = (src) => {
            if (videoCache.has(src)) return videoCache.get(src);

            const video = document.createElement("video");
            video.className = "inference-comparison-video";
            video.autoplay = true;
            video.controls = true;
            video.loop = true;
            video.muted = true;
            video.playsInline = true;
            video.preload = "metadata";
            video.src = src;
            video.setAttribute("aria-hidden", "true");
            video.addEventListener("canplay", () => {
                if (activeSrc === src || !activeSrc) {
                    showVideo(video, src);
                }
            });
            video.addEventListener("error", () => {
                if (activeSrc === src || !activeSrc) {
                    hideActiveVideo();
                    note.hidden = false;
                }
            });
            stage.appendChild(video);
            videoCache.set(src, video);
            return video;
        };

        const selectVideo = (button) => {
            setActiveButton(button);
            const src = getSelectedSrc();
            if (!src) return;

            note.hidden = true;
            activeSrc = src;
            const video = createVideo(src);
            if (video.readyState >= HTMLMediaElement.HAVE_CURRENT_DATA) {
                showVideo(video, src);
            } else {
                video.load();
            }
        };

        buttons.forEach((button) => {
            button.addEventListener("click", () => selectVideo(button));
        });

        const initialButton = buttons.find((button) => button.classList.contains("active")) || buttons[0];
        if (initialButton) {
            selectVideo(initialButton);
        }
    };

    document.querySelectorAll("[data-video-picker]").forEach(setupPicker);
})();
</script>

These results indicate that **OCULAR** can generalize to new unseen environments and achieve both adequate planning performance and safety by leveraging perception information obtained in other environments. Below are numerical results.

<p class="result-table-title">Planning results across three Isaac Sim roads (30 trials each).</p>
<div class="result-table-wrap">
<table class="result-table" aria-label="Planning results across three Isaac Sim roads, 30 trials each">
    <thead>
        <tr>
            <th class="method-cell" rowspan="2">Method</th>
            <th class="narrow-col" rowspan="2"><span class="nowrap">Tested map</span><br><span class="nowrap"><span class="header-negation">not</span> in <code>$D_{\mathrm{cal}}$</code>?</span></th>
            <th colspan="3">Success (%) <svg class="metric-arrow metric-arrow-up" viewBox="0 0 16 16" aria-hidden="true"><path d="M8 13V3M4.5 6.5 8 3l3.5 3.5"/></svg></th>
            <th colspan="3">Steps to completion (mean &plusmn; std) <svg class="metric-arrow metric-arrow-down" viewBox="0 0 16 16" aria-hidden="true"><path d="M8 3v10m3.5-3.5L8 13 4.5 9.5"/></svg></th>
        </tr>
        <tr>
            <th>icySide</th>
            <th>icyMain</th>
            <th>icyMiddle</th>
            <th>icySide</th>
            <th>icyMain</th>
            <th>icyMiddle</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td class="method-cell">NoCP</td>
            <td class="narrow-col"><span class="status neutral">N/A</span></td>
            <td>0</td>
            <td>0</td>
            <td>0</td>
            <td>--</td>
            <td>--</td>
            <td>--</td>
        </tr>
        <tr>
            <td class="method-cell">SplitCP</td>
            <td class="narrow-col"><span class="status cross" aria-label="No">&times;</span></td>
            <td>0</td>
            <td>0</td>
            <td>0</td>
            <td>--</td>
            <td>--</td>
            <td>--</td>
        </tr>
        <tr>
            <td class="method-cell">LUCCa</td>
            <td class="narrow-col"><span class="status cross" aria-label="No">&times;</span></td>
            <td><strong>100</strong></td>
            <td><strong>100</strong></td>
            <td><strong>100</strong></td>
            <td>401.4 &plusmn; 21.2</td>
            <td>517.7 &plusmn; 9.5</td>
            <td>302.3 &plusmn; 9.1</td>
        </tr>
        <tr>
            <td class="method-cell method-ours"><strong>OCULAR (ours)</strong></td>
            <td class="narrow-col"><span class="status check" aria-label="Yes">&#10003;</span></td>
            <td><strong>100</strong></td>
            <td><strong>100</strong></td>
            <td><strong>100</strong></td>
            <td><strong>238.6 &plusmn; 6.5</strong></td>
            <td><strong>302.7 &plusmn; 5.6</strong></td>
            <td><strong>294.7 &plusmn; 11.7</strong></td>
        </tr>
    </tbody>
</table>
</div>

<p class="table-note">Success = reaching all subgoals without collisions.</p>

# BibTeX <small><small>(cite this!)</small></small>

<div class="bibtex-copy-container">
    <button id="bibtex-copy-button" type="button" aria-label="Copy BibTeX to clipboard">
        <svg viewBox="0 0 24 24" aria-hidden="true" focusable="false">
            <path d="M16 1H4a2 2 0 0 0-2 2v12h2V3h12V1zm3 4H8a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h11a2 2 0 0 0 2-2V7a2 2 0 0 0-2-2zm0 16H8V7h11v14z"/>
        </svg>
        <span>Copy to clipboard</span>
    </button>
    <pre id="bibtex-content"><code>@misc{marques2026localconformalcalibrationdynamics,
      title={Local Conformal Calibration of Dynamics Uncertainty from Semantic Images},
      author={Luís Marques and Dmitry Berenson},
      year={2026},
      eprint={2605.13028},
      archivePrefix={arXiv},
      primaryClass={cs.RO},
      url={https://arxiv.org/abs/2605.13028},
}</code></pre>
</div>

<script>
(() => {
    const copyButton = document.getElementById("bibtex-copy-button");
    const bibtexContent = document.getElementById("bibtex-content");
    if (!copyButton || !bibtexContent) return;

    const defaultText = "Copy to clipboard";
    const setButtonLabel = (label) => {
        const textSpan = copyButton.querySelector("span");
        if (textSpan) textSpan.textContent = label;
    };

    copyButton.addEventListener("click", () => {
        const bibtexText = bibtexContent.textContent ?? "";
        if (!navigator.clipboard || !navigator.clipboard.writeText) {
            setButtonLabel("Clipboard not available");
            setTimeout(() => setButtonLabel(defaultText), 3000);
            return;
        }

        navigator.clipboard.writeText(bibtexText).then(() => {
            setButtonLabel("Copied!");
            setTimeout(() => setButtonLabel(defaultText), 3000);
        }).catch(() => {
            setButtonLabel("Copy failed");
            setTimeout(() => setButtonLabel(defaultText), 3000);
        });
    });
})();
</script>

<style>
.bibtex-copy-container {
    position: relative;
}

.bibtex-copy-container pre {
    margin-top: 0;
    padding-top: 0.9rem;
    padding-right: 11.2rem;
    overflow-x: auto;
    white-space: pre;
}

.bibtex-copy-container code {
    font-size: 0.86rem;
    line-height: 1.35;
}

#bibtex-copy-button {
    position: absolute;
    top: 0.9rem;
    right: 0.6rem;
    display: inline-flex;
    align-items: center;
    gap: 0.4rem;
    border: 1px solid #d0d7de;
    border-radius: 6px;
    background: #f6f8fa;
    color: #24292f;
    padding: 0.35rem 0.6rem;
    font-size: 0.85rem;
    cursor: pointer;
    z-index: 1;
}

#bibtex-copy-button:hover {
    background: #eef2f6;
}

#bibtex-copy-button svg {
    width: 1rem;
    height: 1rem;
    fill: currentColor;
}

@media screen and (max-width: 768px) {
    #bibtex-copy-button {
        top: 0.45rem;
        right: 0.45rem;
        font-size: 0.78rem;
        padding: 0.3rem 0.5rem;
    }

    .bibtex-copy-container pre {
        padding-top: 0.75rem;
        padding-right: 9.7rem;
    }

    .bibtex-copy-container code {
        font-size: 0.73rem;
        line-height: 1.28;
    }
}
</style>
