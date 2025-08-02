# What is Sampling?

!!!warning "Under Construction"

As said in [#what is generating](../Models/00_models.md#what-does-generating-mean), **sampling** means to pick data out according to a data distribution; A data distribution is something that assigns probabilities to data, for example a cat image distribution $p_\text{CatImages}$ would assign high probability to cat images, and low probability to everything else like dog images.

As said in [#how 2 diffuse](../Models/01_diffusion.md#how-2-diffuse), for diffusion models, sampling is done through solving the corresponding ODE/SDE that would transform an input from an initial distribution, like noise from a gaussian, into a clean image, like a cat image from a cat distribution.

## What is a Sampler?

While not all of them were conceived this way, all samplers can be thought of as something that can solve the diffusion ODE/SDE.
