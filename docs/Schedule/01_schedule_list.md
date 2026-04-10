# (Non-exhaustive) List of Schedulers

- **`normal`:** Time uniform sampling that includes both `σ_max` and `σ_min`
	- In practice, this may lead to having a final tiny step (`σ_min` to 0) that doesn't behave well
- **`karras`:** The schedule introduced in [EDM](https://arxiv.org/abs/2206.00364) by Karras et al.
- **`exponential`:** A schedule where the `σ` follows an exponential curve
- **`sgm_uniform`:** `normal`, except that `σ_min` is not included
	- You can probably replace `normal` with this one in all situations
	- AFAICT, the naming comes from **S**tabilityAI's **g**enerative-**m**odels [repository](https://github.com/Stability-AI/generative-models/tree/main).
- **`simple`:** [comfy wrote the simplest scheduler they could think of for fun](https://github.com/comfyanonymous/ComfyUI/discussions/227). In practice, basically the same as `sgm_uniform`.
- **`ddim_uniform`:** The schedule used in the original DDIM paper
- **[`beta`](https://arxiv.org/abs/2407.12173):** Paper authors found the model settles general composition in the early steps while ironing out small details in the later steps, with "nothing" really happening in the middle steps. They thus argue to use the [beta](https://en.wikipedia.org/wiki/Beta_distribution) distribution for generating a schedule, which would give more steps to both ends and leave a few steps in the middle.
- **`kl_optimal`:** Introduced in [AYS](https://arxiv.org/pdf/2404.14507) paper that theoretically would minimize the kl divergence.
- **[`AYS`](https://arxiv.org/pdf/2404.14507) (Align Your Steps):** Predefined lists of noise levels found through numerical optimization to be good for a specific model architecture.
	- `SD1` / `SDXL` / `SVD`: Match this with the one your model is based on. 
	- `AYS 11` / `AYS 32`: `AYS` tuned for 10 / 31 steps generation. If you select any step count besides 10 / 31, log-linear interpolation is used to create the undefined noise levels.
- **[`GITS`](https://arxiv.org/abs/2405.11326) (Geometry-Inspired Time Scheduling):** Paper authors examine different imagegen neural nets and found surprisingly similar generation trajectories across them. They exploit the structure of this trajectory, which is GITS.
