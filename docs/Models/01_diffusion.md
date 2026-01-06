# A Premier on Diffusion

!!!note "Sources"
    This guide mostly references [MIT](https://diffusion.csail.mit.edu/docs/lecture-notes.pdf)'s lecture notes, and thus might differ from how other literature formuate things.

    For example, since flow could be thought of as a special case for diffusion, I'll be referring to both as "diffusion" here.

As mentioned in [#what is a model](./00_models.md#what-does-generating-mean), generative models are models that allow us to sample from the data's underlying probability distribution $p_\text{data}.$ Diffusion is the latest development.

## Motivation for Diffusion

Recall from [#the issue in modeling $p_\text{data}$](./00_models.md#why-modeling--is-a-problem) that directly modeling $p_\text{data}$ proves problematic.

Diffusion models instead tries to learn to *nudge* Gaussian noise $p_\text{init}$ and gradually transforming it into $p_\text{data}.$ 

This seemingly simple change of perspective lifts it of many drawbacks and restrictions other solutions had, leading to them becoming SOTA in fields like image and video generation, and enabling tasks like inpainting to be done.

## How 2 Diffuse

Start by defining paths that link between the noise distribution $p_\text{data}$ and the images distribution $p_\text{data}.$ To be concrete, I'll define the **timestep $t$**:

- $t=0$ is the start of the path, where $x_0$ is pure Gaussian noise from $p_\text{init}$.
- $t=1$ is the destination, where $x_1$ is a realistic image from $p_\text{data}$.
- Any other $t$ is somewhere inbetween, where $x_t$ is some combination of data and noise.

Following such a path is the same as gradually turning noise into clean images.

A diffusion model learns something called a **velocity field** which can navigate this path. Given a noisy image $x_t$ at a specific time $t,$ the velocity field tells you in what way to change the pixels, such that you end up with a realistic image by the time you hit $t=1.$

Mathematically, an equation that relates to an object and how it should change is called a **Differential Equation (DE).**

???note "Illustrative Analogy"
    You just landed at the airport ($x_0,$ noise) and you want to get to a restaruant ($x_1,$ image), but you don't have a map. You can navigate by asking people directions:

    1. Ask for a direction (Query the Model).
    1. Walk in that direction for a small distance (Step).
    1. Ask again to see if the direction has changed (Repeat).

    There is a tradeoff: Asking for directions takes time. If you ask at every single step, you will arrive exactly at the restaurant, but it will take all day. If you ask once and walk a mile, you might overshoot or turn down the wrong alley. In generative modeling, "asking" is a **Neural Function Evaluation** (running the neural network). More steps lead to higher accuracy but slower generation.

Differential equations come in 2 kinds: **stochastic (SDE)** or **ordinary (ODE)**. The basic difference is that SDEs include a noise term while ODEs don't. This also leads to the concept of SDEs being **non-convergent,** where one can observe that a stochastic sampler like `euler_ancestral` never really "settle" on an image and increasing step counts can drastically change the final image composition.

To generate an image then is to solve one of these SDEs or ODEs. This also means that all samplers can be thought of as SDE or ODE solvers, and SDE or ODE solvers developed before diffusion, like the Runge-Kutta methods, can be used as samplers.

!!!info "On ODE / SDE Sampling of Diffusion / Flow"
    While denoising diffusion and flow matching were formulated using SDEs and ODEs respectively, it has been shown that one can construct an equivalent ODE / SDE from the other, like in [Song et. al.](https://arxiv.org/abs/2011.13456) for example.

    This means that when implemented correctly, you can use ODE methods with denoising diffusion and SDE methods with flow. 

???note "Differences in Notation"
    Earlier works derived diffusion from another framework (Markov Chain Monte Carlo, MCMC), and you thus may see conflicting conventions.

    For example, some refer to the transformation $p_\text{init}\to p_\text{data}$ as the **reverse process,** preferring a **time reversal** convention where instead, this process steps *backwards* through **discrete time:** $t=1000,999,998,...,0.$

    Some conventions from then still linger to this day, such as data prediction being written as $x_0$-prediction and not $x_1$-prediction.


## Latents - Why and How

Directly working with images is costly - each image has thousands of pixels, each with their rgb values.

**Latents** are compressed representation of images that still retain most of the information, like colors, composition, etc. Intuitively, think of converting images to their latents as jpg compression - you lose a little detail in exchange for massively reduced file size.

Thus, **latent diffusion** was born - it's basically just diffusion + using latents. This significantly reduces the hardware reqs for both training and inferencing, which is why they dominate the market. `SD`, `Flux`, you name it, it probably uses latents.

How do we compress and decompress an image from / to its latent? Well, we don't know... but we can train a neural network to do it for us! For this, **Variational AutoEncoders (VAEs)** are the dominant players. 

!!!info "VAE En/Decode is Lossy"
    Like jpg compression, turning images into latents and back is "lossy" - that is, you lose a bit of information in the process. 
    
    This often comes up when doing many inpainting iterations: **you do not want to en/decode the same image many times, lest the image quality slowly but surely degrades.**

### Practical Details

Modern VAEs usually compress 8x8 = 64 normal pixels into 1 latent pixel. This is also why **you can only specify image widths and heights in multiples of 8** - the VAE decode can *only* produce images whose sizes are multiples of 8.

Each normal pixel has 3 channels: red, green, and blue. Each latent pixel has differing numbers of channels depending on model. Having more channels per latent pixel means more information could be retained, but is more intense on hardware and sometimes harder for a diffusion model to learn.

Originally, most decided to go with a 4-channel VAE, including `SD 1.X` and `SDXL`. In recent times, there has been a move towards higher channel VAEs for higher quality, see page 4 [here](https://arxiv.org/pdf/2309.15807) for an example on better text rendering with higher channels. 

- `SD1.X`, `SDXL`, `Pixart`: 4 channel VAE.
- `Qwen Image`, `Flux 1`, `SD 3.X`, `Lumina 2.0`: 16 channel VAE. 
- `Flux 2`: 32 channel VAE.
- `Hunyuan Image`: 64 channel VAE that compresses a 32x32 patch.
- `PixelFlow`: ditched latent space and has gone back to directly genning in pixels.
