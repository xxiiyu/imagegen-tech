# [CFG++: Manifold-Constrained Guidance](https://cfgpp-diffusion.github.io/)

## Overview

**CFG++** revamps the way to update the noisy latent $x_t,$ allowing it to work for much lower guidance scales. As discussed in [#cfg and the data manifold](./00_cfg.md#cfg-and-the-data-manifold), it's safe to assume a scale $0\leq\omega\leq1$ stays on the data manifold, and thus is what the authors recommend.  

- One can usually access these by choosing predefined samplers. For example in ComfyUI `euler_ancestral_cfg_pp` is `euler_ancestral` using CFG++.
- In practice, it's often still beneficial to image quality to use scales higher than that, like $\omega=1.5.$

!!!info "CFG++ for Flow Matching"
    At least in comfy, most CFG++ implementations currently assume a VE/VP schedule, and lack a scaling parameter $\alpha_t = 1-\sigma_t$ required for flow matching models, making them break. Exceptions I know of include comfyui's `euler_(ancestral_)cfg_pp`

## [DD] Inner Workings of CFG++

The way to update the noisy latent $x$ closer to the clean image estimate at each step for CFG and CFG++ is as follows:

!!!info "Algorithm 1: CFG Update Rule (Euler Method)"
    1. Use the model to predict the conditional (positive) and unconditional (negative) clean image estimates, then combine them to find the CFG clean image estimate:  
    ${\color{teal}\hat x_c} \gets D_\theta(x, \sigma_i, c)$  
    ${\color{cornflowerblue}\hat x_\emptyset} \gets D_\theta(x, \sigma_i, \emptyset)$  
    ${\color{orangered}\hat x_{cfg}} \gets {\color{cornflowerblue}\hat x_\emptyset} + {\color{orangered}\omega}({\color{teal}\hat x_c} - {\color{cornflowerblue}\hat x_\emptyset})$  
    2. Find the CFG update direction:  
    ${\color{orangered}u_{cfg}} \gets (x - {\color{orangered}\hat x_{cfg}}) / \sigma_i$  
    3. Find how much to update the noisy latent by:  
    $h_i = \text{find\_step\_size}(\sigma_i, \sigma_{i+1})$  
    4. Update the noisy latent in the update direction by that amount:  
    $x \gets x + \underbrace{{\color{orangered}u_{cfg}} \cdot h_i}_{\text{update}}$  

!!!info "Algorithm 2: CFG++ Update Rule (Euler Method)"
    1. Use the model to predict the conditional (positive) and unconditional (negative) clean image estimates:  
    ${\color{teal}\hat x_c} \gets D_\theta(x, \sigma_i, c)$  
    ${\color{cornflowerblue}\hat x_\emptyset} \gets D_\theta(x, \sigma_i, \emptyset)$  
    2. Find the **unconditional** update direction:  
    ${\color{cornflowerblue}u_\emptyset} \gets (x - {\color{cornflowerblue}\hat x_\emptyset}) / \sigma_i$  
    3. Find how much to update the noisy latent by:  
    $h_i = \text{find\_step\_size}(\sigma_i, \sigma_{i+1})$  
    4. Update the noisy latent in the update direction by that amount, **and apply guided shift:**  
    $x \gets x + \underbrace{{\color{cornflowerblue}u_\emptyset} \cdot h_i}_{\text{update}} + \underbrace{{\color{orangered}\omega}({\color{teal}\hat x_c} - {\color{cornflowerblue}\hat x_\emptyset})}_{\text{guided shift}}$

As you can see, the difference between CFG and CFG++ is surprisingly small: 

1. Use the **unconditional** update direction instead of the cfg one.
2. Apply a **guided shift** after the update.

In general, most conventional samplers can be adapted to use CFG++ with minimal changes - simply have the sampler do their thing on the unconditional estimate $\hat x_\emptyset$ (and the previous unconditional estimates for multistep samplers) instead of the CFG estimate $\hat x_{cfg},$ then just add the guided shift.

And in essense, both should have the effect of nudging the image towards the prompted image. However, CFG++'s approach has some advantages, discussed in sections below.

### Oscillation: Why CFG Breaks at Low Scales

Let's examine the evolution of the guided clean estimate $\hat x_{cfg}$ across multiple time steps for a single image. To begin, for brevity, I'll first denote the "guidance signal" as $s=\hat x_c-\hat x_\emptyset.$ Intuitively, this signal pushes the image towards the positive prompt and away from the negative prompt.

For CFG++, it is very simple - between each step, $\hat x_{cfg}$ changes by $\omega\cdot s.$ Intuitively, this gradually evolves the image at a nice rate to match the prompt.  

For CFG, it turns out that the effective change of $\hat x_{cfg}$ is approximately $\omega(s-s_\text{prev})$ - the difference between the current guidance signal and the previous one, multiplied by the scale.  
Since $s$ doesn't change much between consecutive steps, subtracting them practically cancels the signal out, and thus a much higher $\omega$ is required to effectively push the image towards the prompt. However, high $\omega$ also pushes the image off the data manifold, causing artifacts.
