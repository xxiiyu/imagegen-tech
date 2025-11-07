# Model Patches

This page is roughly for nodes that input and output a `model`, modifying said model in some way (like adjusting the cfg function, noise schedule, etc.)

### ModelSamplingDiscrete

For diffusion models. **Does nothing unless** you need to manually set the prediction type, like when the model doesn't include metadata about which to use.

**`sampling: lcm`** is used for a good chunk of [#distill models / loras](../Sampling/06_training_based_list.md), specifically those using ideas from [**L**atent **C**onsistency **M**odels](https://arxiv.org/abs/2310.04378), like SDXL Lightning, Hyper, DMD2, etc.  
Not required per se, but using this follows the theory and probably improves the quality.

**`sampling: img_to_img`** is used for... I'm not sure. I'd guess [Lotus](https://arxiv.org/abs/2409.18124) based on when this was added and the commit message adding it.

**`zsnr: true`** forces a **z**ero-terminal **s**ignal-**t**o-**n**oise ratio schedule. In other words, it ensures that at the last timestep, it is indeed pure noise, and not just "very high noise but still with a tiny bit of information." Intended for `v_prediction`.

**`sampling: eps, x0, v_prediction`** are mathematically connected through these formulae:

$x_\theta=\frac1{\alpha_t}(x_t-\sigma_t\epsilon_\theta)=\alpha_tx_t-\sigma_tv_\theta$

Where $\alpha, \sigma$ are numbers relating to the noise schedule, $x_\theta, \epsilon_\theta, v_\theta$ are the various prediction types, and $x_t$ is the noise-image-mix currently being diffused.  

This means *in theory*, you can run an eps-pred model as a v-pred model. In practice, it's best that you use the type that the model was trained with.

- The model we train already isn't perfect, and it's best to stick to what it was trained with lest it performs worse (train-inference mismatch).
- In certain situations, $\alpha_t, \sigma_t$ may become 0 / infinite, breaking down the mathematical connection.

### ModelSamplingSD3

For flow matching models. Sets $\sigma_\text{max}$ and $\sigma_\text{min}$ to 1 and 0, respectively, as is expected for those.

- **`shift: 1`** disables shift.
- **`shift: 3`** is the default determined by the SD3 team through human preference testing on 1024x1024. 6 was a close second.

Higher `shift` biases the model to spend more time on higher noise levels, theoretically improving composition while removing detail.

It does this by re-mapping timesteps. Specifically:

$$
t_\text{new}=\frac{\text{shift}\times t_\text{old}}{1+(\text{shift}-1)\times t_\text{old}}
$$

**Motivation:** SD3 realized that using the same scheduler for different resolution images is a bad idea.

**Intuition:** You start with random noise, the models pick up on its patterns to make stuff like shapes, details, etc. Higher resolution = more pixels = too many patterns = too many local details + bad composition.

### ModelSamplingAuraFlow

Same as `ModelSamplingSD3`.

!!!note "Technical Difference"
    The two only differ in how they make the model represent `timestep`s, however, I don't think comfy explicitly uses those anywhere, or if it does, it's always as a percentage anyway, so it doesn't matter.  
    Specifically, due to how the theory evolved, there are 2 conventions: represent `timestep` from 0 to 1 or from 0 to 1000. `ModelSamplingAuraFlow` uses the former while `ModelSamplingSD3` uses the latter.

### [DifferentialDiffusion](https://differential-diffusion.github.io/)

For inpainting.

An inpaint mask, which has values ranging from 0 to 1, by default results in a binary decision of "this pixel should(n't) be inpainted." 

`DifferentialDiffusion` enables models to utilize the full range of mask values, using them as "how much change should be applied to this pixel."

Theoretically improves blending at borders / reduces seams.

### [FreeU / FreeU_V2](https://chenyangsi.top/FreeU/)

For UNet models. (sd1.5, sdxl)

- **`s1, s2`:** Skip-connection attenuation scale. Higher = suppress skip-connection outputs.
- **`b1, b2`:** Backbone strengthening scale. Higher = amplify backbone outputs. 

"Free lunch for diffusion UNets that improve quality with no cost." -authors

UNet outputs comprise two parts: backbone and skip-connections. Authors argue that the backbone is more important, and at the same time, skip-connections may introduce unnatural high-frequency features.

YMMV. Probably have to do an exhaustive search to find the best parameters. 

### [FreSca](https://wikichao.github.io/FreSca/)

Apply independent scaling to high-frequency (details) and low-frequency (structure) features.

Paraphrasing from the paper:

- **`scale_high: >1, scale_low: 1`:** "fine-detail enhancement"
- **`scale_high: <1, scale_low: >1`:** "smoothing"
- **`scale_high, scale_low: 1`:** disables functionality

**`freq_cutoff`:** Higher = more stuff is considered low frequency.

## CFG Mods

For this section, I'll be assuming that the normal cfg function looks like the following:
$$
x_\text{cfg}=x_n+w(x_p-x_n)
$$
where $x_\text{cfg}$ is the normal cfg denoised result, $w$ is cfg, $x_p$ is positive prediction, and $x_n$ is negative prediction.

**Overshooting** is where, because the cfg guiding effect is too strong, an image during sampling is guided too far away - outside of what the model has been trained on, and it panics.  
This leads to low-quality images, with common issues including extreme saturation, color distortion, artifacts, and a general lack of detail. I'll call this **cfg burn.**

### [RescaleCFG](https://arxiv.org/abs/2305.08891)

Mitigates overshooting. The final output becomes a mix of $x_\text{cfg}$ and a "scaled" version of $x_\text{cfg}$.

- **`multiplier`:** cfg rescale multiplier $\phi.$ Higher = use more of the rescaled result to make the mix.

**Does nothing if cfg is 1.**

Changes the cfg function to the following:
$$
x_\text{final}=\phi\times\frac{\text{std}(x_p)}{\text{std}(x_\text{cfg})}\times x_\text{cfg}+(1-\phi)\times x_\text{cfg}
$$
Where $x_\text{final}$ is the final output and $\text{std}$ is the standard deviation.

**Motivation:** $x_\text{cfg}$ may become too big, especially with high cfg and/or opposite pointing $x_p,x_n,$ making the model overshoot. 

**Fix:** Match the size of $x_\text{cfg}$ to that of $x_\text{positive}.$ Compared to simply lowering cfg, this preserves the strong guiding effect of high cfg, but tones it down dynamically during sampling if it's about to overshoot.

### RenormCFG

Mitigates overshooting.

- **`cfg_trunc`:** during sampling, set cfg to 1 if `sigma > cfg_trunc`, otherwise sample normally. 
    - Note this doesn't trigger comfy's fast implementation (where negative is skipped if you set cfg=1, 2x-ing speed), so it's quite literally useless.
    - Default is 100 because flow models' sigmas range from 1 to 0, so this is intended as the "disable this function" big value.
- **`renorm_cfg`:** cfg renorm multiplier $\rho.$ 
    - `renorm_cfg: 0`: Disable.
    - `renorm_cfg: <1`: Image becomes very washed very fast.
    - `renorm_cfg: >1`: Does renorm less aggressively. Higher = less effect = closer to no renorm.

**Does nothing if cfg is 1.**

Actually combines 2 techniques, [CFG-Renormalization](https://arxiv.org/abs/2412.07730) and [CFG-Truncation](https://arxiv.org/abs/2503.21758).

CFG-Renorm, like CFG-Rescale, also seeks to mitigate overshoot, and also uses $x_p$'s size as a guide. CFG-Renorm just uses this formula instead:
$$
x_\text{final}=\rho\times\frac{||x_p||}{||x_\text{cfg}||}\times x_\text{cfg}
$$
Where $||...||$ is the L2 norm, in other words $\sqrt{x_1^2+x_2^2+x_3^2+...}$ where the $x_1,x_2$ etc are the elements of said tensor.

CFG-Trunc skips using cfg at later steps because some other research found it affects little.  
...which would mean that comfy's nodes and the research it's based on do opposite things (comfy skips at early steps)? Probably just me being dumb and overlooking something, but idk.

### CFGNorm

Mitigates overshooting.

Similar to CFG-Renormalization.

|| `CFGNorm` | `RenormCFG` |
|-| - | - |
| Renorm applied to | image being denoised ($x_t$) | change vector ($\epsilon, v, u,$ etc) |
| Norm based on | only across the channel dimension | standard L2 (across channel, height, width) |

How does this affect what comes out of `RenormCFG` vs. `CFGNorm`? IDFK, experiment ig.

Originally specific to Hidream E1, repurposed as a general post-cfg patch.

### [Adaptive Projected Guidance](https://arxiv.org/abs/2410.02416)

Mitigates overshooting.

- **`eta`:** Scaling of parallel component. 
    - `eta: >1`: Higher saturation.
    - `eta: 0~1`: Intended useage.
    - `eta: <0`: Probably don't (moves away from positive).
- **`renorm_threshold`:** Higher = less aggresive renorming.
- **`momentum`:** Influence of previous steps on current step
    - `momentum: <0`: Intended usage. Push the model away from the prior prediction direction.

APG incorporates 3 ideas:

1. **Supress parallel component:** Specifically, decompose $D=w(x_p-x_n)$ into two components, one parallel and one perpendicular to the positive prediction $x_p:$

    - Parallel: Seems to be responsible for saturation.
    - Perpendicular: Seems to be responsible for image quality improvements.  

    Thus, it makes sense to scale the parallel component by some factor $\eta <1.$

1. **Renormalize CFG:** Yeah... renorm again. It's different, though, APG constrains $D$ to have length of at most `renorm_threshold`.

1. **Reverse momentum:** Add a negative momentum term that pushes the model away from previously taken directions and encourages the model to focus more on the current update direction.

Some recommend **`eta: 1, norm_threshold: 20, momentum: 0`** to replace `RescaleCFG` for vpred.

### [Tangential Dampening CFG](https://5410tiffany.github.io/tcfg.github.io/)

Mitigates overshooting.

Oversimplified, the idea is that a model, in learning how to transform noise to noise-image mixes to good images, also implicitly learns how to transform noise-image mixes to unnatural images that don't appear in training data. (proven through complex math).

**Intuition:** During sampling, both $x_p,$ $x_n$ still point towards "natural images;" Trying to move away from $x_n$ thus may have unintended consequences.

- Example: You negative `chibi`. However, `chibi` still lies in the overall space of anime images. So, besides removing `chibi`, you're actually also subtly telling the model to move away from anime images in general. Oops.

**Fix:** Remove the component of $x_n$ that points towards natural images (so when you move away from it, you don't move away from natural images). The component is approximated using [SVD](https://en.wikipedia.org/wiki/Singular_value_decomposition).

### [Self-Attention Guidance](https://cvlab-kaist.github.io/Self-Attention-Guidance/)

Quality enhancer.  
Increases generation time.  
Reduce your cfg accordingly.

- **`scale`:** how much the SAG results influence things. Higher = more effect.
- **`blur_sigma`:** the std of the Gaussian it uses for blur guidance. Higher = more aggressive blurring.

In essence, SAG does the following:

- Identify high information (thus, difficult to diffuse) regions through the model's self-attention map.
- Blur this part with Gaussian noise, then predict with the model, resulting in a prediction based on blurry inputs.
- Do the usual cfg sampling on the original unblurred image. Then, additionally, move away from the blurry input prediction by `scale`.

This should help most in regions where the model is very unconfident about what to put there, e.g., melting backgrounds.  
It probably doesn't help in regions where the model is confident, whether confidently right or confidently wrong.

### [PerturbedAttentionGuidance](https://cvlab-kaist.github.io/Perturbed-Attention-Guidance/)

Quality enhancer.  
Increases generation time.  
Reduce your cfg accordingly: `PAG_scale + new_cfg = original_cfg` should be a good rule of thumb.

- **`scale`:** how much the PAG results influence things. Higher = more effect.

Sequel to `Self-Attention Guidance`. They do different things, so you can use both at once.

In essence, PAG does the following:

- Make a *perturbed* model by removing its self-attention and replacing it with an identity matrix. Have it predict on the image.
- Do the usual cfg sampling on the original image. Then, additionally, move away from the perturbed model prediction by `scale`.

### [Mahiro is so cute that she deserves a better guidance function!! (。・ω・。)](https://huggingface.co/spaces/yoinked/blue-arxiv)

Quality enhancer.

**Motivation:** normal cfg's negative prompt implicitly has the effect of making the model pay attention to the polar opposite of said negative.

**Fix:**

1. Calculate "positive leap," which is $\text{cfg}\times x_p.$ In the same vein, calculate "negative leap."
    - A "leap" semantically means "following positive/negative." cfg means "follow positive *as well as* move away from negative."
1. Merge the positive leap and cfg, then calculate the similarity between that and the negative leap.
    - High similarity: The leap was bad, trust cfg more.
    - Low similarity: The leap was good, trust the leap more.

(Find the quick read in the link, id `2024-1208.1`. Find a graph [here](https://www.desmos.com/calculator/wcztf0ktiq) comparing normal cfg and mahiro.)
