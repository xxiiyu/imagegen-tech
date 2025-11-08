# Model Patches

This page is roughly for nodes that input and output a `model`, modifying said model in some way (like adjusting the cfg function, noise schedule, etc.)

At the beginning of each section you can find a summary table, formatted like below:

| Aspect | Description |
| - | - |
| Applicable To | Which models work with the node in theory. In practice you can try whatever, though results will vary. |
| Purpose | Why / When to use this node, at least in theory. |
| Notes | Some things that I feel are important |

### ModelSamplingDiscrete

| Aspect | Description |
| - | - |
| Applicable To | Diffusion |
| Purpose | Manually change prediction type. |
| Notes | You only need this node **if the model file doesn't contain metadata** about which type to use; If it does then comfy will detect it and this node serves no purpose. |

- **`sampling`:** In practice, just match this with what the model you're using was trained on.
    - **`eps`:** Predict the noise to remove. Also called $\epsilon$ (**eps**ilon) prediction.
    - **`v_prediction`:** Predict the velocity (a mix between noise and data; not to be confused with flow matching's velocity).
    - **`lcm`:** [**L**atent **C**onsistency **M**odels](https://arxiv.org/abs/2310.04378) mode. Intended for [#distill models / loras](../Sampling/06_training_based_list.md) that use ideas from LCM, like SDXL Lightning, Hyper, DMD2, etc.  
    Not required per se, but using this follows the theory and probably improves the quality.
    - **`x_0`:** Predict the data (clean image). Also called $x_0$ prediction.
    - **`img_to_img`:** I'd guess for [Lotus](https://arxiv.org/abs/2409.18124) based on when this was added and the commit message adding it.
- **`zsnr: true`:** Force a **Z**ero-Terminal **S**ignal-to-**N**oise **R**atio schedule. See [#current schedules are bad](../Models/02_training_objectives.md#current-schedules-are-bad) for why this is important. Intended for `v_prediction`.

!!!note "Details"
    **`sampling: eps, x0, v_prediction`** are mathematically connected through these formulae:

    $x_\theta=\frac1{\alpha_t}(x_t-\sigma_t\epsilon_\theta)=\alpha_tx_t-\sigma_tv_\theta$

    Where $\alpha, \sigma$ are numbers relating to the noise schedule, $x_\theta, \epsilon_\theta, v_\theta$ are the various prediction types, and $x_t$ is the noise-image-mix currently being diffused.  

    This means *in theory*, you can run an eps-pred model as a v-pred model. In practice, it's best that you use the type that the model was trained with.

    - The model we train already isn't perfect, and it's best to stick to what it was trained with lest it performs worse (train-inference mismatch).
    - In certain situations, $\alpha_t, \sigma_t$ may become 0 / infinite, breaking down the mathematical connection.

### ModelSamplingSD3

| Aspect | Description |
| - | - |
| Applicable To | Flow Matching |
| Purpose | Quality enhancement (**improve composition**). |
| Notes | Sets $\sigma_\text{max}$ and $\sigma_\text{min}$ to 1 and 0, respectively, as is expected for flow matching. This unfortunately means you can't easily apply `shift` to diffusion models, even if in theory you can. |

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

### ModelSamplingAuraFlow

Same as `ModelSamplingSD3`.

!!!note "Details"
    The two only differ in how they make the model represent `timestep`s, however, I don't think comfy explicitly uses those anywhere, or if it does, it's always as a percentage anyway, so it doesn't matter.  
    Specifically, due to how the theory evolved, there are 2 conventions: represent `timestep` from 0 to 1 or from 0 to 1000. `ModelSamplingAuraFlow` uses the former while `ModelSamplingSD3` uses the latter.

### [DifferentialDiffusion](https://differential-diffusion.github.io/)

| Aspect | Description |
| - | - |
| Applicable To | All models |
| Purpose | Improve inpainting (**Improve blending at mask edges**). |

An inpaint mask, which may contain values ranging from 0 to 1, by default results in a binary decision of "this pixel should(n't) be inpainted." 

`DifferentialDiffusion` enables models to utilize the full range of mask values, using them as "how much change should be applied to this pixel."

Theoretically improves blending at borders / reduces seams.

### [FreeU / FreeU_V2](https://chenyangsi.top/FreeU/)

| Aspect | Description |
| - | - |
| Applicable To | UNets (`sd1.5`, `sdxl`) |
| Purpose | Quality enhancement |

- **`s1, s2`:** Skip-connection attenuation scale. Higher = suppress skip-connection outputs. Supposedly decreases unnatural details.
- **`b1, b2`:** Backbone strengthening scale. Higher = amplify backbone outputs. Supposedly improves denoising ability.

YMMV. Probably have to do an exhaustive search to find the best parameters. 

!!!note "Details"
    "Free lunch for diffusion UNets that improve quality with no cost." -authors

    UNet outputs comprise two parts: backbone and skip-connections. Authors argue that the backbone is more important, and at the same time, skip-connections may introduce unnatural high-frequency features.


### [FreSca](https://wikichao.github.io/FreSca/)

| Aspect | Description |
| - | - |
| Applicable To | All models |
| Purpose | Quality enhancement |

Apply independent scaling to high-frequency (details) and low-frequency (structure) features.

Paraphrasing from the paper:

- **`scale_high: >1, scale_low: 1`:** "fine-detail enhancement"
- **`scale_high: <1, scale_low: >1`:** "smoothing"
- **`scale_high, scale_low: 1`:** disables functionality

**`freq_cutoff`:** Higher = more stuff is considered low frequency.

## CFG Mods

**Overshooting** - usually occuring due to a cfg guiding effect that is too strong - is when an image during sampling is guided outside of what the model has been trained on, so the model panics and starts making poor predictions.

Overshooting leads to low-quality images, with common issues including extreme saturation, color distortion, and a general lack of detail. I'll collective call these artifacts **cfg burn.**

!!!note "Details"
    For this section, I'll be assuming that the normal cfg function looks like the following:
    $$
    x_\text{cfg}=x_n+w(x_p-x_n)
    $$
    Where $x_\text{cfg}$ is the normal cfg denoised result, $w$ is cfg, $x_p$ is positive prediction, and $x_n$ is negative prediction. When these symbols are used later that's what they mean.

### [RescaleCFG](https://arxiv.org/abs/2305.08891)

| Aspect | Content |
| - | - |
| Applicable To | All models (originally intended for v-pred) |
| Purpose | Mitigate overshooting. |
| Notes | **Does nothing if cfg is 1.** |

This takes the original cfg denoised result $x_\text{cfg},$ and scale it down into $x_\text{scaled}$ to prevent overshoot. The two are then averaged together and that becomes the final model output. 

The authors chose to not directly use $x_\text{scaled}$ as the final model output, because "the generated images are overly plain." 

- **`multiplier`:** cfg rescale multiplier $\phi.$ Higher = use more of the rescaled result to make the mix.
    - For example, `0.7` means that the final output is 70% $x_\text{scaled}$ and 30% $x_\text{cfg}.$

Compared to simply lowering cfg, this ideally preserves the strong guiding effect of high cfg, but tones it down dynamically during sampling if it's about to overshoot.

!!!note "Deatils"
    Changes the cfg function to the following:
    $$
    x_\text{final}=\phi\times\frac{\text{std}(x_p)}{\text{std}(x_\text{cfg})}\times x_\text{cfg}+(1-\phi)\times x_\text{cfg}
    $$
    Where $x_\text{final}$ is the final output and $\text{std}$ is the standard deviation.

    **Motivation:** $x_\text{cfg}$ may become too big, especially with high cfg and/or opposite pointing $x_p,x_n,$ making the model overshoot. 

    **Fix:** Match the size of $x_\text{cfg}$ to that of $x_p.$ 

### RenormCFG

| Aspect | Content |
| - | - |
| Applicable To | All models (originally from Lumina Image 2.0) |
| Purpose | Mitigate overshooting. |
| Notes | **Does nothing if cfg is 1.** |

- **`cfg_trunc`:** during sampling, set cfg to 1 at high noise levels (when `sigma > cfg_trunc`), otherwise sample normally. 
    - This doesn't trigger comfy's fast sampling optimization, where negative is skipped if you set cfg=1, 2x-ing speed, so it's quite useless.
    - Default is `100` because flow models' sigmas range from 1 to 0, so this is intended as the "disable this function" big value. You can connect a `Float (utils > primitive)` to it and set it bigger if you need it.
- **`renorm_cfg`:** cfg renorm multiplier $\rho.$ 
    - `0`: Disable.
    - `<1`: Image becomes very washed very fast.
    - `>1`: Does renorm less aggressively. Higher = less effect = closer to no renorm.

!!!note "Deatils"
    Actually combines 2 techniques, [CFG-Renormalization](https://arxiv.org/abs/2412.07730) and [CFG-Truncation](https://arxiv.org/abs/2503.21758).

    CFG-Renorm, like CFG-Rescale, also seeks to mitigate overshoot, and also uses $x_p$'s size as a guide. CFG-Renorm just uses this formula instead:
    $$
    x_\text{final}=\rho\times\frac{||x_p||}{||x_\text{cfg}||}\times x_\text{cfg}
    $$
    Where $||...||$ is the L2 norm, in other words $\sqrt{x_1^2+x_2^2+x_3^2+...}$ where the $x_1,x_2$ etc are the elements of said tensor.

    CFG-Trunc skips using cfg at later steps because some other research found it affects little.  
    ...which would mean that comfy's nodes and the research it's based on do opposite things (comfy skips at early steps)? Probably just me being dumb and overlooking something, but idk.

### CFGNorm

| Aspect | Content |
| - | - |
| Applicable To | All models (originally from Hidream E1) |
| Purpose | Mitigate overshooting. |
| Notes | **Does nothing if cfg is 1.** |


Similar to CFG-Renormalization.

!!!note "Details"
    Some differences:

    || `CFGNorm` | `RenormCFG` |
    |-| - | - |
    | Renorm applied to | image being denoised ($x_t$) | change vector ($\epsilon, v, u,$ etc) |
    | Norm based on | only across the channel dimension | standard L2 (across channel, height, width) |

    How does this affect what comes out of `RenormCFG` vs. `CFGNorm`? IDFK, experiment ig.

### [Adaptive Projected Guidance](https://arxiv.org/abs/2410.02416)

| Aspect | Content |
| - | - |
| Applicable To | All models. |
| Purpose | Mitigate overshooting. |

- **`eta`:** Scaling of parallel component. 
    - `>1`: Higher saturation.
    - `0~1`: Intended usage.
    - `<0`: Probably don't (moves away from positive).
- **`renorm_threshold`:** Higher = less aggresive renorming.
- **`momentum`:** Influence of previous steps on current step
    - `<0`: Intended usage. Push the model away from the prior prediction direction.

Some recommend **`eta: 1, norm_threshold: 20, momentum: 0`** to replace `RescaleCFG` for vpred.

!!!note "Details"
    APG incorporates 3 ideas:

    1. **Supress parallel component:** Specifically, decompose $D=w(x_p-x_n)$ into two components, one parallel and one perpendicular to the positive prediction $x_p:$

        - Parallel: Seems to be responsible for saturation.
        - Perpendicular: Seems to be responsible for image quality improvements.  

        Thus, it makes sense to scale the parallel component by some factor $\eta <1.$

    1. **Renormalize CFG:** It's renorm again but different (tm). APG constrains $D$ to have a size of at most `renorm_threshold`.

    1. **Reverse momentum:** Add a negative momentum term that pushes the model away from previously taken directions and encourages the model to focus more on the current update direction.

### [Tangential Dampening CFG](https://5410tiffany.github.io/tcfg.github.io/)

| Aspect | Content |
| - | - |
| Applicable To | All models. |
| Purpose | Mitigate overshooting. |

**Intuition:** During sampling, the negative prediction still points towards "natural images;" Trying to move away from it thus may have unintended consequences.

- Example: You negative `chibi`. However, `chibi` still lies in the overall space of anime images. So, besides removing `chibi`, you're actually also subtly telling the model to move away from anime images in general. Oops.

**Fix:** Remove the part of the negative prediction that points towards natural images, so when you move away from it, you don't move away from natural images. 

!!!note "Details"
    Oversimplified, the idea is that a model, in learning how to transform noise to noise-image mixes to good images, also implicitly learns how to transform noise-image mixes to unnatural images that don't appear in training data. (proven through complex math).

    The component to remove from $x_n$ is approximated using said information and the [SVD](https://en.wikipedia.org/wiki/Singular_value_decomposition) algorithm.

### [Self-Attention Guidance](https://cvlab-kaist.github.io/Self-Attention-Guidance/)

| Aspect | Content |
| - | - |
| Applicable To | All models |
| Purpose | Quality enhancement (**clarity / remove blur and melting color blobs**)  |
| Notes | **Reduce your cfg accordingly. Increases generation time.** |

- **`scale`:** how much the SAG results influence things. Higher = more effect.
- **`blur_sigma`:** the std of the Gaussian it uses for blur guidance. Higher = more aggressive blurring.

In essence, SAG does the following:

- Identify high information (thus, difficult to diffuse) regions through the model's self-attention map.
- Blur this part with Gaussian noise, then predict with the model, resulting in a prediction based on blurry inputs.
- Do the usual cfg sampling on the original unblurred image. Then, additionally, move away from the blurry input prediction by `scale`.

This should **help most in regions where the model is very unconfident** about what to put there, e.g., melting backgrounds.  
It probably doesn't help in regions where the model is confident, whether confidently right or confidently wrong.

### [PerturbedAttentionGuidance](https://cvlab-kaist.github.io/Perturbed-Attention-Guidance/)

| Aspect | Content |
| - | - |
| Applicable To | All models |
| Purpose | Quality enhancement (**remove degraded predictions**)  |
| Notes | **Reduce your cfg accordingly: `PAG_scale + new_cfg = original_cfg`** should be a good rule of thumb. **Increases generation time.** |

Sequel to `Self-Attention Guidance`. They do different things, so you can use both at once.

In essence, PAG does the following:

- Make a *perturbed* model by removing its self-attention and replacing it with an identity matrix. Have it predict on the image.
- Do the usual cfg sampling on the original image. Then, additionally, move away from the perturbed model prediction by `scale`.

### [Mahiro is so cute that she deserves a better guidance function!! (。・ω・。)](https://huggingface.co/spaces/yoinked/blue-arxiv)

| Aspect | Content |
| - | - |
| Applicable To | All models |
| Purpose | Quality enhancement (**Better alignment to positive**)  |

**Motivation:** normal cfg's negative prompt implicitly has the effect of making the model pay attention to the polar opposite of said negative.

**Fix:** When applicable, pay more attention to the positive and less to the negative.

!!!note "Details"
    Specifically:

    1. Calculate "positive leap," which is $\text{cfg}\times x_p.$ In the same vein, calculate "negative leap."
        - A "leap" semantically means "following positive/negative." cfg means "follow positive *as well as* move away from negative."
    1. Merge the positive leap and cfg, then calculate the similarity between that and the negative leap.
        - High similarity: The leap was bad, trust cfg more.
        - Low similarity: The leap was good, trust the leap more.

    (Find the quick read in the link, id `2024-1208.1`. Find a graph [here](https://www.desmos.com/calculator/wcztf0ktiq) comparing normal cfg and mahiro.)
