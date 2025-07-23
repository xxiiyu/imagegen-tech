# A Premier on Diffusion

!!!warning "UNDER CONSTRUCTION"

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

- [An Introduction to Flow Matching and Diffusion Models](https://diffusion.csail.mit.edu/docs/lecture-notes.pdf)
- [Loss Functions in Diffusion Models: A Comparative Study](https://arxiv.org/abs/2507.01516)
- [Elucidating the Design Space of Diffusion-Based Generative Models](https://arxiv.org/abs/2206.00364)
- [Stochastic Interpolants: A Unifying Framework for Flows and Diffusions](https://arxiv.org/abs/2303.08797)
