# Miscellaneous

## ModelSamplingDiscrete

| Aspect        | Description                                                                                                                                                           |
| ------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Applicable To | Diffusion                                                                                                                                                             |
| Purpose       | Manually change prediction type.                                                                                                                                      |
| Notes         | You only need this node **if the model file doesn't contain metadata** about which type to use; If it does then comfy will detect it and this node serves no purpose. |

- **`sampling`:** Match this with what the model you're using was trained on.
    - **`eps`:** Predict the noise to remove. Also called $\epsilon$ (**eps**ilon) prediction.
    - **`v_prediction`:** Predict the velocity (a mix between noise and data; not to be confused with flow matching's velocity).
    - **`lcm`:** [**L**atent **C**onsistency **M**odels](https://arxiv.org/abs/2310.04378) mode. Intended for [#distill models / loras](../../Sampling/06_training_based_list.md) that use ideas from LCM, like SDXL Lightning, Hyper, DMD2, etc.  
    Not required per se, but using this follows the theory and probably improves the quality.
    - **`x_0`:** Predict the data (clean image). Also called $x_0$ prediction.
    - **`img_to_img`:** I'd guess for [Lotus](https://arxiv.org/abs/2409.18124) based on when this was added and the commit message adding it.
- **`zsnr: true`:** Force a **Z**ero-Terminal **S**ignal-to-**N**oise **R**atio schedule. See [#current schedules are bad](../../Models/12_training_objectives.md#current-schedules-are-flawed) for why this is important. Intended for `v_prediction`.

!!!note "Details"
    **`sampling: eps, x0, v_prediction`** are mathematically connected through these formulae:

    $\hat x=\frac1{\alpha_t}(x_t-\sigma_t\hat\epsilon)=\alpha_tx_t-\sigma_t\hat v$

    Where $\alpha, \sigma$ are numbers relating to the noise schedule, $\hat x, \hat\epsilon, \hat v$ are the model estimates in various prediction types, and $x_t$ is the noise-image-mix currently being diffused. Note that flow matching/rectified flow is simply `v_prediction` with $\alpha_t=1,\sigma_t=t.$  

    This means *in theory*, you can run an eps-pred model as a v-pred model. In practice, it's best that you use the type that the model was trained with.

    - The model already isn't perfect, and it's best to stick to what it was trained with lest it performs worse (train-inference mismatch).
    - In certain situations, $\alpha_t, \sigma_t$ may become 0 / infinite, breaking down the mathematical connection.

## ModelSamplingSD3

| Aspect        | Description                                                                                                                                                                                                          |
| ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Applicable To | Flow Matching                                                                                                                                                                                                        |
| Purpose       | Quality enhancement (**improve composition**).                                                                                                                                                                       |
| Notes         | Sets $\sigma_\text{max}$ and $\sigma_\text{min}$ to 1 and 0, respectively, as is expected for flow matching. This unfortunately means you can't easily apply `shift` to diffusion models, even if in theory you can. |

- **`shift`:** Higher `shift` biases the model to spend more time on higher noise levels, theoretically improving composition while removing detail.
    - `1` disables shift.
    - `3` is the default determined by the SD3 team through human preference testing on 1024x1024. `6` was a close second.

**Motivation:** SD3 realized that using the same scheduler for different resolution images is a bad idea.

**Intuition:** You start with random noise, the models pick up on its patterns to make stuff like shapes, details, etc. Higher resolution = more pixels = too many patterns = too many local details + bad composition.

!!!note "Details"
    To "bias models to certain noise levels," `shift` re-maps timesteps. Specifically:
    $$
    t_\text{new}=\frac{\text{shift}\times t_\text{old}}{1+(\text{shift}-1)\times t_\text{old}}
    $$

## ModelSamplingAuraFlow

Functionally the same as `ModelSamplingSD3`, but implementation details mean that some model need to use this one wile others the `SD3` one.

!!!note "Details"
    The two only differ in how they make the model represent `timestep`s, however, I don't think comfy explicitly uses those anywhere, or if it does, it's always as a percentage anyway, so it doesn't matter.  
    Specifically, due to how the theory evolved, there are 2 conventions: represent `timestep` from 0 to 1 or from 0 to 1000. `ModelSamplingAuraFlow` uses the former while `ModelSamplingSD3` uses the latter.

## [DifferentialDiffusion](https://differential-diffusion.github.io/)

| Aspect        | Description                                              |
| ------------- | -------------------------------------------------------- |
| Applicable To | All models                                               |
| Purpose       | Improve inpainting (**Improve blending at mask edges**). |

An inpaint mask, which may contain values ranging from 0 to 1, by default results in a binary decision of "this pixel should(n't) be inpainted." 

`DifferentialDiffusion` enables models to utilize the full range of mask values, using them as "how much change should be applied to this pixel."

Improves blending at borders / reduces seams.

## [FreeU / FreeU_V2](https://chenyangsi.top/FreeU/)

| Aspect        | Description             |
| ------------- | ----------------------- |
| Applicable To | UNets (`sd1.5`, `sdxl`) |
| Purpose       | Quality enhancement     |

- **`s1, s2`:** Skip-connection attenuation scale. Higher = suppress skip-connection outputs. Supposedly decreases unnatural details.
- **`b1, b2`:** Backbone strengthening scale. Higher = amplify backbone outputs. Supposedly improves denoising ability.

YMMV. Probably have to do an exhaustive search to find the best parameters. 

!!!note "Details"
    "Free lunch for diffusion UNets that improve quality with no cost." -authors

    UNet outputs comprise two parts: backbone and skip-connections. Authors argue that the backbone is more important, and at the same time, skip-connections may introduce unnatural high-frequency features.


## [FreSca](https://wikichao.github.io/FreSca/)

| Aspect        | Description         |
| ------------- | ------------------- |
| Applicable To | All models          |
| Purpose       | Quality enhancement |

Apply independent scaling to high-frequency (details) and low-frequency (structure) features.

Paraphrasing:

- **`scale_high: >1, scale_low: 1`:** "fine-detail enhancement"
- **`scale_high: <1, scale_low: >1`:** "smoothing"
- **`scale_high, scale_low: 1`:** disables functionality

**`freq_cutoff`:** Higher = more stuff is considered low frequency.
