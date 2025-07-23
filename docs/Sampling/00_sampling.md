# What is Sampling?

**Sampling** means to pick data using a probability distribution that has been learned by a model. For our image generation purposes, this means to generate a new image that is similar to real images that the diffusion model has been trained on.

For diffusion models, sampling is done through the **reverse process** - that is, diffusion models learn the process of removing noise from (**denoising**) an image. Specifically:

1. Start with an "image" of pure noise $x_T$
2. Iteratively apply the reverse process $x_t\to x_{t-1}$, gradually removing noise 
3. After many steps, the result is a clean image $x_0$

As you may guess, **unsampling** (the **forward process**) then refers to the process of adding noise to an image.

## What is a Sampler?

**A sampler** is something that decides the way we sample; a method that denoises an image. This process can be modeled as a [differential equation (DE)](https://en.wikipedia.org/wiki/Differential_equation) with a [given initial condition](https://en.wikipedia.org/wiki/Initial_value_problem) - which is why **"samplers" can be thought of as solvers for DEs.** In fact, a good amount of samplers were conceived by starting with this idea.
