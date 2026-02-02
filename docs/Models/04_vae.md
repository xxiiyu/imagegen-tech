# VAE (Variational AutoEncoder)


The **Variational AutoEncoder (VAE)** is the most common architecture used to build models that can encode and compress complex data into smaller and simpler representations called *latents.* These latents can then be utilized by diffusion models, making them easier to train and run.

!!!note Focus
    I'll mainly be talking about VAEs through the lens of image generation. However, note that they can be applied to other forms of data as well. 

## Preliminary: The Manifold Hypothesis and Latent Spaces

Most real-world data is "sparse." For example, 512x512 rgb images exist in a 786,432-dimensional space, that is, 786,432 numbers per image. However, assigning each pixel with random rgb values will, 99.99% of the time, result in meaningless static. Only a tiny, tiny subset of all possible pixel arrangements results in coherent "dogs" or "landscapes" or other recognizable objects.

!!!info "The Manifold Hypothesis" 
    The **Manifold Hypothesis** formalizes the above idea. It suggests that real-world high-dimensional data (like natural images) actually lies on a much lower-dimensional well-behaved subspace called a *manifold* inside that high-dimensional space.

This implies that rather than describing natural images in *pixel space* by listing every rgb pixel value, it should be possible to represent them more efficiently with a compact set of underlying features.

A **latent space** $z$ is the coordinate system that uses said compact feature set. 

<!--
An ideal latent space has several crucial advantages over pixel space:

1. Drastically **lower dimensions,** reducing the computation and storage needed to manipulate them.
2. **Distances and directions are meaningful.** Nearby points correspond to semantically similar images, and interpolating between points yields smooth, realistic transitions.  
3. Individual dimensions (or combinations thereof) correspond to **interpretable, independent attributes** of the data, like pose, lighting, expression in a face, etc.
4. **Completeness,** where even randomly sampled points mostly map to valid, coherent images, unlike the sparse pixel space which mostly give back static.
-->

Latent spaces are usually too complex to find manually. Instead, deep neural networks are trained to learn one such space. Though these learned spaces are far from ideal, they have enough advantages to be widely adopted.

## AutoEncoders

An **AutoEncoder (AE)**'s primary purpose is to learn informative latent representations of the data. An AE is simply a pair of 2 functions:

- `Encode(x)`: move an image `x` to latent space
- `Decode(z)`: move a latent image `z` to pixel space

How good an AE is can be measured by how well it preserves the data. Specifically:

1. Take some image `x`
2. Encode it to latent space `z = Encode(x)`
3. Decode it back to pixel space `x' = Decode(z)`
4. `x` and `x'` should be as similar as possible.

This is called the **reconstruction quality.** The loss to train an AE can also be derived from this idea, aptly named the reconstruction loss.

However, as a standard AE is trained to map 1 data point to 1 latent point (and back), the learned latent space has 2 major flaws that make it unsuitable for generative purposes:

1. **Discontinuity:** Points near each other in latent space may represent completely different images. This means:
    - One can't generate more cats by encoding a `cat`, varying the latent a little, then decoding it back.
    - If a diffusion model uses this AE, even a little error in output can result in completely different outputs, making the diffusion model hard to train.
2. **Incompleteness:** The latent space contains many "gaps" which represent latent images that `Decode` has never seen, and thus never learned to reconstruct. Trying to `Decode` this latent will result in nonsensical images. This means:
    - One can't generate more images by randomly sampling a latent and then decoding it, because it's likely that a random latent lands in one of those pesky gaps.
    - Assume an AE trained on `cat` and `dog` images, but not images containing both. If a diffusion model uses this AE, it would be impossible to generate a hybrid image because the necessary latent representation likely falls into a gap.

## Variational AutoEncoders

A **Variational AutoEncoder (VAE)** addresses both issues faced by AEs in generative modeling, by mapping inputs to **probability distributions** rather than single points. Which type of distribution? Why, the humble Gaussian of course! They're simple enough so it's easy to generate, yet flexible enough to express the latents.

Now, instead of producing a specific $z$, `Encode` makes a mean $\bm{\mu_z}$ and a standard deviation $\bm{\sigma_z}$ which together parameterize a Gaussian. Rather than receiving a specific $z$, `Decode` now gets a random sample of this Gaussian $\mathcal N(\bm{\mu_z}, \bm{\sigma_z}).$ This fixes both issues that stop AEs from generative modeling:

1. **Continuity:** As `Decode` now receives a range of inputs around $\bm{\mu_z}$ that should all decode back into the original, it learns to map entire neighborhoods of the latent space to similar images.  
2. **Completeness:** During training, a regularization term (KL Divergence) is introduced to "pull" the latent Gaussians to cluster near the origin. This creates a densely packed region without gaps, where any randomly sampled latent is guaranteed to decode into a valid image.

However, this robustness comes at a cost: blur. The VAE tends to smooth out fine details, resulting in lossy, fuzzy images. This also shows in reconstruction, where a VAE is worse at it than AEs, tending to make the reconstructed images blurrier.

While a VAE isn't ideal for generative modeling by itself, combining it with diffusion models proved to be a great idea: 

- The VAE handles the compression, learning an informative and computationally efficient latent space.
- The diffusion model handles the generation, learning all the complex patterns that finally hone in on high-fidelity images.

### Practical Details

Modern VAEs usually compress 8x8 = 64 normal pixels into 1 latent pixel. This is also why **you can only specify image widths and heights in multiples of 8** - the VAE decode can *only* produce images whose sizes are multiples of 8.

Each normal pixel has 3 channels: red, green, and blue. Each latent pixel has a different number of channels depending on the model. Having more channels per latent pixel means more information could be retained, but is more intense on hardware and sometimes harder to train a diffusion model with.

Originally, most decided to go with a 4-channel VAE, including `SD 1.X` and `SDXL`. In recent times, there has been a move towards higher channel VAEs for higher quality, see page 4 [here](https://arxiv.org/pdf/2309.15807) for an example on better text rendering with higher channels. 

- `SD1.X`, `SDXL`, `Pixart`: 4 channel VAE.
- `Qwen Image`, `Flux 1`, `SD 3.X`, `Lumina 2.0`: 16 channel VAE. 
- `Flux 2`: 32 channel VAE.
- `Hunyuan Image`: 64 channel VAE that compresses a 16x16 patch.
- `PixelFlow`, `Chroma Radiance`: ditched latent space and went back to directly genning in pixels.

## References

- https://www.youtube.com/watch?v=HBYQvKlaE0A
