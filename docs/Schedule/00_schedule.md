# What is a Noise Schedule?
!!!warning "UNDER CONSTRUCTION"
	This section is incomplete and subject to massive changes. 
!!!Info "What's The Difference Between Samplers and Schedulers?"
	The **sampler** controls **the way to remove** that amount of noise.
	The **scheduler** controls **how much noise to remove** at each step.

A scheduler is (a way to generate) a list of numbers that quantify the amount of noise in an image (usually denoted using sigma $\sigma$).
For generating images, $\sigma$ starts at a high value (very noisy image) and ends at a very low value (clear images). 

The actual values of the highest and lowest $\sigma$ ($\sigma_\text{max}, \sigma_\text{min}$) are different across models, which in turn affect the values that schedulers generate, but this isn't something you have to worry about as it's handled by your frontend (A1111, ComfyUI, forge, etc).

For practical training reasons, $\sigma_\text{min}$ actually may not be 0 (which would represent a clean, no-noise image). A 0 is often added to the end of the list to represent that there should be no noise in the final result.

<!-- (On the creation of sigmas: https://forums.fast.ai/t/init-noise-sigma/101423/4) -->
