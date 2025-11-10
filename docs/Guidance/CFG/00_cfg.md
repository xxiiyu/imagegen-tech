# What is CFG?
!!!note "CFG in Flux"
	**Flux was not trained with CFG, thus nothing in this section applies.** The "Guidance" value you can provide to flux is not CFG.

It's simplest to understand how CFG works exactly by digging through [code](https://github.com/comfyanonymous/ComfyUI/blob/master/comfy/samplers.py):
```
cfg_result = uncond_pred + (cond_pred - uncond_pred) * cond_scale
```
Where `uncond_pred` is the model prediction without conditioning, and `cond_pred` is the model prediction with conditioning. `cond_scale` is CFG, and `cfg_result` is the result image we get. For brevity, I'll be referring to `uncond_pred` and `cond_pred` as `uncond` and `cond` respectively. You can imagine that:

- `uncond` is what the model thinks it should do.
- `cond` is what the model thinks it should do, *given* our guidance.

Let's play with some CFG numbers:

| cfg | `cfg_result` | effect |
| - | - | -
| 0.1 | `uncond*0.9 + cond*0.1` | The effect that our guidance has is pretty weak. 90% of the generation process is still decided by `uncond`.
| 0.5 | `uncond*0.5 + cond*0.5` | The strength of our guidance is on even footing with the unguided conditioning.
| 1 | `cond` | **The `uncond` cancels out, leaving us with only `cond`.** The generation process is entirely decided by our guidance.
| 2 | `2*cond - uncond` | **The model actively moves away from `uncond`,** while the effect that our guidance has increases even more.

When implementing CFG in practice, people also noticed what we found here - namely that when `cfg > 1`, the model moves away from `uncond`. Then, couldn't we use `uncond` as some sort of opposite guidance - "Do anything but this?" Yes! This is what became negative prompts.

Now with that in mind, let's rewrite the equation in more familiar terms:
```
denoised_image = negative + (positive - negative) * cfg
```
Where you can imagine `positive` and `negative` as the representation of the positive and negative prompts respectively that the model understands.
Armed with this knowledge, let's reiterate what happens with prompts at various CFG levels:

- `cfg < 1`: Negative prompts would behave like positive prompts.
- `cfg = 1`: Negative prompts have no effect. 
- `cfg > 1`: The model will actively avoid generating anything in the negative prompt.

I also want to note something special about `cfg = 1` - that is, the negative prompt having no effect. Couldn't we skip calculating `negative` entirely then? Yep. ComfyUI does this, which is why you'll see **2x iteration speed if you set `cfg = 1`.**
