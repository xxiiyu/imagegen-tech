# [CFG++: Manifold-Constrained Guidance](https://cfgpp-diffusion.github.io/)
**Rescales CFG to work in the range of 0 to 1** making it work better at low guidance scales. It also makes the diffusing process smoother, and may lead to reduced artifacts.
One usually accesses these by choosing predefined samplers, i.e. in ComfyUI `euler_ancestral_cfg_pp` is `euler_ancestral` using CFG++.

!!!note "CFG++ in Practice"
    In my and Bex's personal tests:

    - CFG++ samplers give eps-pred models a very good color range, rivaling that of what v-pred models claim to achieve. 
    - Using CFG++ to inpaint at high steps breaks the image for reasons unknown.

!!!info "CFG++ for Flow Matching"
    Most CFG++ implementations right now assume a VE/VP schedule, and lack a scaling parameter $\alpha_t = 1-\sigma_t$ required for flow matching models, making them break. Exceptions I know of include comfyui's `euler_(ancestral_)cfg_pp`

<!--
Note that in ComfyUI, the `_cfg_pp` samplers in `KSampler` are alt implementations where you instead simply want to divide the CFG you're used to by 2.  
For example, if you usually run `euler_ancestral` at CFG=7, you'd run `euler_ancestral_cfg_pp` at CFG=3.5.  
In practice, the reasonable range I find for these is CFG between `[0.7, 3]`, and `1.6` is a sane default to start with.
-->