# Guidance Modification

This section is on various nodes which modify the guidance in the sampling process (prevent CFG artifacts, additional guidance from sources other than text, etc).

**Overshooting** - usually occuring due to a cfg guiding effect that is too strong - is when an image during sampling is guided outside of what the model has been trained on, so the model panics and starts making poor predictions.

Overshooting leads to low-quality images, with common issues including extreme saturation, color distortion, and a general lack of detail. I'll collective call these artifacts **cfg burn.**

!!!note "Assumptions"
    For this section, I'll be assuming that the normal cfg function looks like the following:
    $$
    x_\text{cfg}=x_n+w(x_p-x_n)
    $$
    Where $x_\text{cfg}$ is the normal cfg denoised result, $w$ is cfg, $x_p$ is positive prediction, and $x_n$ is negative prediction. When these symbols are used later that's what they mean.

## [RescaleCFG](https://arxiv.org/abs/2305.08891)

| Aspect        | Content                                     |
| ------------- | ------------------------------------------- |
| Applicable To | All models (originally intended for v-pred) |
| Purpose       | Mitigate overshooting.                      |
| Notes         | **Does nothing if cfg is 1.**               |

This takes the original cfg denoised result $x_\text{cfg},$ and scale it down into $x_\text{scaled}$ to prevent overshoot. The two are then averaged together and that becomes the final output. 

The authors chose to not directly use $x_\text{scaled}$ as the final output, because "the generated images are overly plain." 

- **`multiplier`:** cfg rescale multiplier $\phi.$ Higher = use more of the rescaled result to make the mix.
    - For example, `0.7` means that the final output is 70% $x_\text{scaled}$ and 30% $x_\text{cfg}.$

Compared to simply lowering cfg, this ideally preserves the strong guiding effect of high cfg, but tones it down dynamically during sampling if it's about to overshoot.

!!!note "Deatils"
    Changes the cfg function to the following:
    $$
    x_\text{cfg-rescaled}=\phi\times\frac{\text{std}(x_p)}{\text{std}(x_\text{cfg})}\times x_\text{cfg}+(1-\phi)\times x_\text{cfg}
    $$
    Where $x_\text{cfg-rescaled}$ is the final output and $\text{std}$ is the standard deviation.

    **Motivation:** $x_\text{cfg}$ may become too big, especially with high cfg and/or opposite pointing $x_p,x_n,$ making the model overshoot. 

    **Fix:** Match the size of $x_\text{cfg}$ to that of $x_p.$ 

## RenormCFG

| Aspect        | Content                                       |
| ------------- | --------------------------------------------- |
| Applicable To | All models (originally from Lumina Image 2.0) |
| Purpose       | Mitigate overshooting.                        |
| Notes         | **Does nothing if cfg is 1.**                 |

- **`cfg_trunc`:** during sampling, set cfg to 1 at high noise levels (when `sigma > cfg_trunc`), otherwise sample normally. 
    - This doesn't trigger comfy's fast sampling optimization (where if you set `cfg = 1`, comfy recognizes it can skip calculating the negative, resulting in 2x speed).
    - `100`: Flow models' $\sigma$ range from 1 to 0, so this is intended as the "disable this function" big value. You can connect a `Float (utils > primitive)` to it and set it bigger if you need it.
- **`renorm_cfg`:** cfg renorm multiplier $\rho.$ 
    - `0`: Disable.
    - `<1`: Image becomes very washed very fast.
    - `>1`: Does renorm less aggressively. Higher = less effect = closer to no renorm.

!!!note "Deatils"
    Actually combines 2 techniques, [CFG-Renormalization](https://arxiv.org/abs/2412.07730) and [CFG-Truncation](https://arxiv.org/abs/2503.21758).

    CFG-Renorm, like CFG-Rescale, also seeks to mitigate overshoot, and also uses $x_p$ as a guide. CFG-Renorm just uses this formula instead:
    $$
    x_\text{final}=\rho\times\frac{||x_p||}{||x_\text{cfg}||}\times x_\text{cfg}
    $$
    Where $||...||$ is the L2 norm, in other words $\sqrt{x_1^2+x_2^2+x_3^2+...}$ where the $x_1,x_2$ etc are the elements of said tensor.

    CFG-Trunc skips using cfg at later steps because some other research found it affects little.  
    ...which would mean that comfy's nodes and the research it's based on do opposite things (comfy skips at early steps)? Probably just me being dumb and overlooking something, but idk.

## CFGNorm

| Aspect        | Content                                 |
| ------------- | --------------------------------------- |
| Applicable To | All models (originally from Hidream E1) |
| Purpose       | Mitigate overshooting.                  |
| Notes         | **Does nothing if cfg is 1.**           |


Similar to (but no the same as) CFG-Renormalization.

## [Adaptive Projected Guidance](https://arxiv.org/abs/2410.02416)

| Aspect        | Content                |
| ------------- | ---------------------- |
| Applicable To | All models.            |
| Purpose       | Mitigate overshooting. |

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

    2. **Renormalize CFG:** It's renorm again but different (tm). APG constrains $D$ to have a size of at most `renorm_threshold`.

    3. **Reverse momentum:** Add a negative momentum term that pushes the model away from previously taken directions and encourages the model to focus more on the current update direction.

## [Tangential Damping CFG](https://5410tiffany.github.io/tcfg.github.io/)

| Aspect        | Content                |
| ------------- | ---------------------- |
| Applicable To | All models.            |
| Purpose       | Mitigate overshooting. |

**Intuition:** During sampling, the negative prediction still points towards "natural images;" Trying to move away from it thus may have unintended consequences. See the [#cfg page](../../Guidance/CFG/00_cfg.md#cfg-and-the-data-manifold) for more details.

- Example: `chibi` is put in negative. However, `chibi` still lies in the overall space of anime images. So besides removing `chibi`, the model's also steering away from anime images in general. Oops.

**Fix:** Remove the part of the negative prediction that points towards natural images, so when moving away from it with cfg, it's not moving away from all natural images.

!!!note "Details"
    Oversimplified, the idea is that a model, in learning how to transform noise to noise-image mixes to good images, also implicitly learns how to transform noise-image mixes to unnatural images that don't appear in training data. (proven through complex math).

    The component to remove from $x_n$ is approximated using said information and the [SVD](https://en.wikipedia.org/wiki/Singular_value_decomposition) algorithm.

## [Self-Attention Guidance](https://cvlab-kaist.github.io/Self-Attention-Guidance/)

| Aspect        | Content                                                                 |
| ------------- | ----------------------------------------------------------------------- |
| Applicable To | All models                                                              |
| Purpose       | Quality enhancement (**clarity / remove blur and melting color blobs**) |
| Notes         | **Reduce your cfg accordingly. Increases generation time.**             |

- **`scale`:** how much the SAG results influence things. Higher = more effect.
- **`blur_sigma`:** the std of the Gaussian it uses for blur guidance. Higher = more aggressive blurring.

In essence, SAG does the following:

- Identify high information (thus, difficult to diffuse) regions through the model's self-attention map.
- Blur this part with Gaussian noise, then predict with the model, resulting in a prediction based on blurry inputs.
- Do the usual cfg sampling on the original unblurred image. Then, additionally, move away from the blurry input prediction by `scale`.

This should **help most in regions where the model is very unconfident** about what to put there, e.g., melting backgrounds.  
It probably doesn't help in regions where the model is confident, whether confidently right or confidently wrong.

## [PerturbedAttentionGuidance](https://cvlab-kaist.github.io/Perturbed-Attention-Guidance/)

| Aspect        | Content                                                                                                                              |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| Applicable To | All models                                                                                                                           |
| Purpose       | Quality enhancement (**remove degraded predictions**)                                                                                |
| Notes         | **Reduce your cfg accordingly: `PAG_scale + new_cfg = original_cfg`** should be a good rule of thumb. **Increases generation time.** |

Sequel to `Self-Attention Guidance`. They do different things, so both can be used simutaneously.

In essence, PAG does the following:

- Make a *perturbed* model by removing its self-attention and replacing it with an identity matrix. Have it predict on the image.
- Do the usual cfg sampling on the original image. Then, additionally, move away from the perturbed model prediction by `scale`.

## [Positive-Biased Guidance](https://huggingface.co/spaces/yoinked/blue-arxiv)

| Aspect        | Content                                                |
| ------------- | ------------------------------------------------------ |
| Applicable To | All models                                             |
| Purpose       | Quality enhancement (**Better alignment to positive**) |

**Motivation:** Normal cfg's negative prompt implicitly has the effect of making the model pay attention to the polar opposite of said negative.

**Fix:** When applicable, pay more attention to the positive and less to the negative.

!!!note "Details"
    1. Calculate "positive leap," which is $\text{cfg}\times x_p.$ In the same vein, calculate "negative leap."
        - A "leap" semantically means "following positive/negative." cfg means "follow positive *as well as* move away from negative."
    1. Merge the positive leap and cfg, then calculate the similarity between that and the negative leap.
        - High similarity: The leap was bad, trust cfg more.
        - Low similarity: The leap was good, trust the leap more.

    (Find the quick read in the link, id `2024-1208.1`. Find a graph [here](https://www.desmos.com/calculator/wcztf0ktiq) comparing normal cfg and mahiro.)

**Trivia:** Previously named "Mahiro is so cute that she deserves a better guidance function!! (。・ω・。)"

# Appendix
## CFG Rescales and Renorms

Many aforementioned methods aim to prevent cfg overshooting by reducing the "size" of intermediate outputs during the sampling process.

The difference lies in how they measure the size, and which intermediate output they apply resizing to:

|                     | `RescaleCFG`                                            | `RenormCFG`    | `CFGNorm`                                         | `Adaptive Projected Guidance`        |
| ------------------- | ------------------------------------------------------- | -------------- | ------------------------------------------------- | ------------------------------------ |
| Size measure        | standard deviation                                      | l2 norm        | l2 norm                                           | l2 norm                              |
| Resizing applied to | clean estimate                                          | clean estimate | intermediate noisy latent                         | specific component of clean estimate |
| Notes               | Final output is a mix of the unscaled and scaled output |                | Only uses the channel dimension to calculate size |

- `Adaptive Projected Guidance` decomposes $\text{cfg}\times(x_\text{positive}-x_\text{negative})$ further and resizes a part of that.

What implications does this have in practice? Honesty, I can't say for sure, so experiment ig.
