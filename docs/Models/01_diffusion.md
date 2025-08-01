# A Premier on Flow and Diffusion

!!!warning "UNDER CONSTRUCTION"

!!!note "On Differentiating Between Diffusion and Flow"
    Since flow could be thought of as a special case for diffusion, I'll be referring to both as "diffusion" here.

As mentioned in [#what is a model](./00_models.md#what-does-generating-mean), generative models can be formalized as models that allow us to sample from the data's underlying probability distribution $p_\text{data}.$ Diffusion is the latest development.

## Motivation for Diffusion

Recall from [#the issue in modeling $p_\text{data}$](./00_models.md#why-modeling--is-a-problem) that we trained a neural network $\text{NN}(\text{Input}),$ and use $\frac{e^{-\text{NN}(\text{Input})}}Z$ as our output to (1) force our outputs be positive since probabilities should be positive, and (2) force that the sum of all possible outputs be 1 since it's guaranteed for there being *an* outcome to any event.

The issue lies in $Z$ being hard to calculate. Remedies have been designed, yet all come with undesirable downsides. Diffusion solves these issues, specifically:

1. Diffusion doesn't have to calculate nor approximate $Z.$
1. Diffusion imposes no restrictions on neural network architecture, nor does it use the unstable adversarial loss.
1. Diffusion still models $p_\text{data},$ ensuring samples are actually similar to the training examples. 

These strong foundations lead to them becoming SOTA in fields like image and video generation, and enabling tasks like inpainting to be done.

## How 2 Diffuse

Diffusion models are trained to learn "trajectories" that take the input $x_1$ sampled from $p_\text{init}$ (for example, noise taken from a gaussian) to a data point $x_0$ that could realistically come from $p_\text{data}$ (for example, a cat image taken from the distribution of cat images).

These trajectories can be naturally described by something called a **differential equation,** which is a fancy name for an equation that relates something with how it should change. For example, let's say you're half way through generating a cat image with diffusion, so you have a half-noise-half-cat image $x_{0.5}.$ The differential equation would tell you how you should change $x_{0.5}$ to get closer to a real cat image: these pixels should be a bit brighter, those should be less saturated, ... etc.

!!!note "Analogy"
    Imagine flying to France, and you just landed at the airport ($x_1$) and you want to get to a restaruant ($x_0$), but you don't have a map. What you 

Differential equations come in 2 kinds: **stochastic (SDE)** or **ordinary (ODE),** which correspond with **diffusion** and **flow matching** respectively. The basic difference is that SDEs include a noise term while ODEs don't. This also leads to the concept of SDEs being **non-convergent,** where one can observe that a stochastic sampler like `euler_ancestral` never really "settle" on an image and increasing step counts can drastically change the final image composition.

To generate an image then is to solve one of these SDEs or ODEs. This also means that all samplers can be thought of as SDE or ODE solvers, and SDE or ODE solvers developed before diffusion, like the Runge-Kutta methods, can be used as samplers.

!!!info "On ODE / SDE Sampling of Diffusion / Flow"
    While diffusion and flow matching were formulated using SDEs and ODEs respectively, it has been shown that one can construct an equivalent ODE / SDE from the other, like in [Song et. al.](https://arxiv.org/abs/2011.13456) for example.

    This means that when implemented correctly, you can use ODE methods with diffusion and SDE methods with flow. 

## Latents - Why and How

Directly working with images is costly - each image has thousands of pixels, each with their rgb values.

**Latents** are compressed representation of images that still retain most of the information, like colors, composition, etc. Intuitively, think of converting images to their latents as jpg compression - you lose a little detail in exchange for massively reduced file size.

Thus, **latent diffusion** was born - it's basically just diffusion + using latents. This significantly reduces the hardware reqs for both training and inferencing, which is why they dominate the market. `SD`, `Flux`, you name it, it probably uses latents.

How do we compress and decompress an image from / to its latent? Well, we don't know... but we can train a neural network to do it for us! For this, **Variational AutoEncoders (VAEs)** are the dominant players. 

!!!info "VAE En/Decode is Lossy"
    Like jpg compression, turning images into latents and back is "lossy" - that is, you lose a bit of information in the process. This is especially important for e.g. doing many inpainting iterations: **you do not want to en/decode the same image many times, lest the image quality slowly but surely degrades.**

### Practical Details

Modern VAEs usually compress 8x8 = 64 normal pixels into 1 latent pixel. This is also why you can only specify image widths and heights in multiples of 8 - the VAE decode can *only* produce images whose sizes are multiples of 8.

Each normal pixel usually has 3 channels: red, green, and blue. Each latent pixel has differing amounts of channels depending on model. Having more channels per latent pixel means more information could be retained, but the hardware reqs are increased.

Originally, most decided to go with a 4-channel VAE, including `SD 1.X` and `SDXL`. In recent times, there has been a move towards higher channel VAEs. `Flux`, `SD 3.X`, `Lumina 2.0`, all use 16 channel VAEs.

## References

- [(MIT) An Introduction to Flow Matching and Diffusion Models](https://diffusion.csail.mit.edu/docs/lecture-notes.pdf)
- [Loss Functions in Diffusion Models: A Comparative Study](https://arxiv.org/abs/2507.01516)
- [(Karras) Elucidating the Design Space of Diffusion-Based Generative Models](https://arxiv.org/abs/2206.00364)
- [(Song) Score-Based Generative Modeling through Stochastic Differential Equations](https://arxiv.org/abs/2011.13456)
