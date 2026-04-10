# [CFG++: Manifold-Constrained Guidance](https://cfgpp-diffusion.github.io/)

## Overview

**CFG++** revamps the way to update the noisy latent $x_t,$ allowing it to work for much lower guidance scales. As discussed in [#cfg and the data manifold](./00_cfg.md#cfg-and-the-data-manifold), it's safe to assume a scale $0\leq\omega\leq1$ stays on the data manifold, and thus is what the authors recommend.  

- One can usually access these by choosing predefined samplers. For example in ComfyUI `euler_ancestral_cfg_pp` is `euler_ancestral` using CFG++.
- In practice, it's often still beneficial to image quality to use scales higher than that, like $\omega=1.5.$

!!!info "CFG++ for Flow Matching"
    At least in comfy, most CFG++ implementations currently assume a VE/VP schedule, and lack a scaling parameter $\alpha_t = 1-\sigma_t$ required for flow matching models, making them break. Exceptions I know of include comfyui's `euler_(ancestral_)cfg_pp`

### Cancellation: Why CFG Breaks at Low Scales

Let's examine how the guided clean estimate $\color{orangered}\hat{x}_{cfg}$ evolves across denoising steps - that is, what the model believes the final image should look like, given your positive and negative prompts.

Intuitively, each denoising step removes some noise from $\color{purple}x$, so the model's estimate becomes progressively more accurate. As a result, $\color{orangered}\hat{x}_{cfg}$ should converge toward a clean, high-quality image over time.

To start, I'll define the **guidance signal** as $s = {\color{teal}\hat{x}_c} - {\color{cornflowerblue}\hat{x}_\emptyset},$ the difference between the positive and negative clean estimates. This signal is what steers the image toward your positive prompt and away from the negative one; Other components not shown here handle more generic denoising.

For **CFG++**, at each step, $\color{orangered}\hat{x}_{cfg}$ is effectively nudged by $\omega\cdot s$. This means the prompt guidance accumulates steadily at a rate controlled by the CFG scale $\omega$ - simple, right?

For **CFG**, it turns out the effective update is instead about $\omega(s-s_\text{prev})$ - the *change* in the guidance signal from the previous step, scaled by $\omega$. Because $s$ varies slowly between consecutive steps, $s-s_\text{prev}$ is tiny, and a much larger $\omega$ is needed to compensate lest the prompt has little effect.

## [DD] CFG++ Algorithm

The way to update the noisy latent $x$ closer to the clean image estimate at each step for CFG and CFG++ is as follows:

!!!info "Algorithm 1: CFG Update Rule (Euler Method)"
    1. Use the model to predict the conditional (positive) and unconditional (negative) clean image estimates, then combine them to find the CFG clean image estimate:  
    ${\color{teal}\hat x_c} \gets D_\theta({\color{purple}x}, \sigma_i, c)$  
    ${\color{cornflowerblue}\hat x_\emptyset} \gets D_\theta({\color{purple}x}, \sigma_i, \emptyset)$  
    ${\color{orangered}\hat x_{cfg}} \gets {\color{cornflowerblue}\hat x_\emptyset} + \omega({\color{teal}\hat x_c} - {\color{cornflowerblue}\hat x_\emptyset})$  
    2. Find the CFG update direction:  
    ${\color{orangered}u_{cfg}} \gets ({{\color{orangered}\hat x_{cfg}}} - {\color{purple}x}) / \sigma_i$  
    3. Find how much to update the noisy latent by:  
    $h_i = \text{find\_step\_size}(\sigma_i, \sigma_{i+1})$  
    4. Update the noisy latent in the update direction by that amount:  
    ${\color{purple}x} \gets {\color{purple}x} + \underbrace{{\color{orangered}u_{cfg}} \cdot h_i}_{\text{update}}$  

!!!info "Algorithm 2: CFG++ Update Rule (Euler Method)"
    1. Use the model to predict the conditional (positive) and unconditional (negative) clean image estimates:  
    ${\color{teal}\hat x_c} \gets D_\theta({\color{purple}x}, \sigma_i, c)$  
    ${\color{cornflowerblue}\hat x_\emptyset} \gets D_\theta({\color{purple}x}, \sigma_i, \emptyset)$  
    2. Find the **unconditional** update direction:  
    ${\color{cornflowerblue}u_\emptyset} \gets ({{\color{cornflowerblue}\hat x_\emptyset}} - {\color{purple}x}) / \sigma_i$  
    3. Find how much to update the noisy latent by:  
    $h_i = \text{find\_step\_size}(\sigma_i, \sigma_{i+1})$  
    4. Update the noisy latent in the update direction by that amount, **and apply guided shift:**  
    ${\color{purple}x} \gets {\color{purple}x} + \underbrace{{\color{cornflowerblue}u_\emptyset} \cdot h_i}_{\text{update}} + \underbrace{\omega({\color{teal}\hat x_c} - {\color{cornflowerblue}\hat x_\emptyset})}_{\text{guided shift}}$

As you can see, the difference between CFG and CFG++ is surprisingly small: 

1. Use the **unconditional** update direction instead of the cfg one.
2. Apply a **guided shift** after the update.

In general, most conventional samplers can be adapted to use CFG++ with minimal changes - simply have the sampler do their thing on the unconditional estimate $\color{cornflowerblue}\hat x_\emptyset$ (and the previous unconditional estimates for multistep samplers) instead of the CFG estimate $\color{orangered}\hat x_{cfg},$ then just add the guided shift.

Additionally, by examining Algorithm 1, it becomes clear why cancellation occurs (with the assupmtion that $s={\color{teal}\hat x_c} - {\color{cornflowerblue}\hat x_\emptyset}$ varies slowly, implying $\color{orangered}\hat x_{cfg}$ also varies slowly):  

At some step, $\color{purple}x$ is updated by ${\color{orangered}u_{cfg}}\cdot h_i,$ and $\color{orangered}u_{cfg}$ contains $\color{orangered}\hat x_{cfg}$ (scaled) - thus, the updated noisy image $\color{purple}x'$ contains a bit of $\color{orangered}x_{cfg}$.  
In the next step, to find the new update, it calculates ${\color{orangered}u'_{cfg}} \gets ({{\color{orangered}\hat x'_{cfg}}} - {\color{purple}x'}) / \sigma_{i+1}.$ Since $\color{purple}x'$ contains a bit of $\color{orangered}\hat x_{cfg}$ and $\color{orangered}\hat x'_{cfg}$ should be close to ${\color{orangered}\hat x_{cfg}},$ a good chunk of them cancel, leaving a tiny difference. Then a large $\omega$ is needed to make up for it.
