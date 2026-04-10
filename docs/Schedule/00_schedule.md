# What is a Noise Schedule?
!!!warning "UNDER CONSTRUCTION"
	This section is incomplete and subject to massive changes. 
!!!Info "What's The Difference Between Samplers and Schedulers?"
	The **sampler** controls **the way to remove** that amount of noise.  
	The **scheduler** controls **how much noise to remove** at each step.


A noise schedule is a list of numbers that quantify the amount of noise that should be left in an image at a specific timestep. These numbers are usually denoted using sigma $\sigma_t.$  
A scheduler generates a noise schedule according to the model, step count, among other parameters; Maybe through complex calculations, or maybe it's just a predefined list.   
For generating images, $\sigma_t$ starts at a high value and ends at a low value, signifying that as you go, the image should become less and less noisy until you get a clean image.

The highest and lowest values of $\sigma_t$ depends on the model's training schedule. This usually isn't something you need to worry about, as your frontend UI should handle that for you.  
Do note that some schedules were designed for specific models, and thus will produce suboptimal outputs when used on a different one.

For practical training reasons, $\sigma_\text{min}$ actually may not be 0 (which would represent a clean, no-noise image). A 0 is often added to the end of the list to represent that there should be no noise in the final result.

<!-- (On the creation of sigmas: https://forums.fast.ai/t/init-noise-sigma/101423/4) -->
