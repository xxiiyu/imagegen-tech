# Diffusion Models

!!!note "Sources"
    This guide mostly references [MIT](https://diffusion.csail.mit.edu/docs/lecture-notes.pdf)'s lecture notes, and thus might differ from how other literature formuate things.

    For example, since flow could be thought of as a special case for diffusion, I'll be referring to both as "diffusion" here.

## Motivation for Diffusion

Recall from [#the issue in modeling $p_\text{data}$](./00_models.md#why-modeling--is-a-problem) that directly modeling $p_\text{data}$ proves problematic.

Diffusion models instead tries to learn how to *nudge* Gaussian noise $p_\text{init}$ and gradually transform it into $p_\text{data}.$ 

This seemingly simple change of perspective lifts it of many drawbacks and restrictions other solutions had, leading to them becoming SOTA in fields like image and video generation, and enabling tasks like inpainting to be done.

## How 2 Diffuse

Start by defining paths that link between the noise distribution $p_\text{data}$ and the images distribution $p_\text{data}.$ To be concrete, define the **timestep $t$**:

- $t=0$ is the start of the path, where $x_0$ is pure Gaussian noise from $p_\text{init}$.
- $t=1$ is the destination, where $x_1$ is a realistic image from $p_\text{data}$.
- Any other $t$ is somewhere inbetween, where $x_t$ is some combination of data and noise.

Following such a path is the same as gradually turning noise into clean images.

A diffusion model learns something called a **velocity field** which can navigate these paths. Given a noisy image $x_t$ at a specific time $t,$ the velocity field shows in what way to change the pixels, such that by the time it hits $t=1,$ $x_t=x_1$ becomes a realistic image.

???note "Illustrative Analogy"
    You just landed at the airport ($x_0,$ noise) and you want to get to a restaruant ($x_1,$ image), but you don't have a map. You can navigate by asking people directions:

    1. Ask for a direction (Query the Model).
    1. Walk in that direction for a small distance (Step).
    1. Ask again to see if the direction has changed (Repeat).

    There is a tradeoff: Asking for directions takes time. Ask every single step, and you will arrive exactly at the restaurant, but it will take all day. If you ask once and walk a mile, you might overshoot or turn down the wrong alley. In generative modeling, "asking" is a **Neural Function Evaluation (NFE),** or running the model once. More NFE leads to higher accuracy but slower generation.

Mathematically, an equation that relates to an object and how it should change is called a **Differential Equation (DE).**

Differential equations come in 2 kinds: **ordinary (ODE)** or **stochastic (SDE)**. The difference is that SDEs additionally consider random noise into the equation, while ODEs don't. Ever noticed that samplers such as `euler_ancestral` never settling on a final image, instead generating wildly different compositions as one increases sampling steps? That's because it's a stochastic sampler, injecting noise at every step.

Depending on which type of DE is chosen to model the noise-image transformation, it leads to 2 different losses:

- ODE: **flow matching** 
- SDE: **score matching**

Practitioners of today mostly use flow matching - in particular, a specific version of it called **Rectified Flow (RF)** - due to its simplicity and efficiency. In fact, RF is so popular that some use RF and Flow Matching interexchangeably.

To generate an image then is to solve one of these learning ODEs or SDEs. This also means that all samplers can be thought of as ODE or SDE solvers, and ODE and SDE solvers developed before diffusion, like the Runge-Kutta methods, can be used as samplers.

!!!info "On ODE / SDE Sampling of Diffusion / Flow"
    While denoising diffusion and flow matching were formulated using SDEs and ODEs respectively, it has been shown that one can construct an equivalent ODE / SDE from the other, like in [Song et. al.](https://arxiv.org/abs/2011.13456) for example.

    This means that when implemented correctly, you can use ODE methods with denoising diffusion and SDE methods with flow. 

???note "Differences in Notation"
    Earlier works derived diffusion from another framework (Markov Chain Monte Carlo, MCMC), and you thus may see conflicting conventions.

    For example, some refer to the transformation $p_\text{init}\to p_\text{data}$ as the **reverse process,** preferring a **time reversal** convention where instead, this process steps *backwards* through **discrete time:** $t=1000,999,998,...,0.$

    Some conventions from then still linger to this day, such as data prediction being written as $x_0$-prediction and not $x_1$-prediction.


## Latents - Why and How

Directly working with images is costly - each image has hundreds of thousands of pixels, each with their rgb values.

**Latents** are compressed representation of images that still retain most of the information, like colors, composition, etc. Intuitively, think of converting images to their latents as jpg compression - a little detail is lost in exchange for massively reduced file size.

Using latents significantly reduces the compute costs for both training and image generating. This is why models working with latents - *Latent Diffusion Models (LDMs)* - dominate the market. `SD`, `Flux`, `Z-Image`, you name it, it probably uses latents.

How does one compress and decompress an image from / to its latent? Well, we don't know... but we can train a neural network to do it for us! For this, **[#Variational AutoEncoders (VAEs)](04_vae.md)** are the dominant players. 

!!!info "En/Decode is Lossy"
    Like jpg compression, turning images into latents and back is "lossy" - that is, you lose a bit of information in the process. 
    
    This often comes up when doing many inpainting iterations: **you do not want to en/decode the same image many times, lest the image quality slowly but surely degrades.**

    This is also why the default ComfyUI workflow for inpainting is complete trash. Instead, use the `Impact` nodepack's detailer nodes or the `CropAndStitch` nodepack, both of which paste the inpainted region back onto the original, non-degraded image.
