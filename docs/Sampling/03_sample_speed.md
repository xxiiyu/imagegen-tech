# Generation Speed and How to Go Faster

**Running the neural network is the most computationally expensive operation** by far, when it comes to generating images with diffusion. The time taken for a generation is mostly determined by how many times the neural network needs to be ran. Running the neural network is also called "calling/evaluating the model," doing a "model call/evaluation," etc.

This is why in a sampler speed comparison chart, one can notice that most of they seem to run in integer multiples of `euler` speed, e.g. `heun` is about 2x slower than `euler`: `euler` runs the model once per step, making it a great unit to compare against; `heun` runs the model twice per step, hence it's 2x slower.

This is also why researchers don't compare samplers in number of steps, but Number of Function Evaluations (NFE), or in other words how many times the model was run. It wouldn't make practical sense to compare say `euler` and `heun` both at 20 steps, because the latter would have twice the NFE and run for twice as long.

!!!note "Steps is Not Time Spent"
    Be on the lookout for potentially misleading statements, such as: "`dpmpp_sde` is great for low steps sampling." While technically true, `dpmpp_sde` runs the model twice per step, which means that in the time `dpmpp_sde` runs for 5 steps, you could've run say `dpmpp_2m` for 10 steps. Make sure to take this into account when making speed-quality comparisons.

## Some Things Affecting Sampling Speed

Certain samplers run the model multiple times per step to achieve higher accuracy. See the [#sampler tl;dr](../Sampling/07_tldr_usage.md#iteration-speed-how-long-1-step-takes) for details.

Increasing the image resolution means the model has to ingest and output more information, making it take longer.

In each step of generating an image, the model has to predict how to change the current intermediate output to move it closer to an image described by the positive prompt.

- Enabling negative prompts by setting [#cfg](../Guidance/01_cfg.md) to anything but 1 doubles the amount of work and thus halves the speed. The model now additionally needs to predict on the negative prompt (to move away from it).
- [Perp-Neg](https://perp-neg.github.io/) makes negative prompts very strong, but the model now also needs to predict on the empty prompt. This means 1.5x slower than cfg-not-1 and negative prompts are in play, or 3x slower than if cfg is 1.

## Some Things NOT Affecting Sampling Speed

Schedulers only determine the noise levels of each step, the number of model runs should be unaffected.

`ancestral` or `sde` variants need to spend a bit of time manipulating the noise each step, but the time used for this is tiny compared to calling the model. They largely run as fast as their original version.

`cfg_pp` variants redo how cfg works, but doesn't change the number of model calls.

## Table of Sampler Speeds

For convenience, below is a table of commonly found samplers and how much longer they take to complete 1 iteration, relative to `euler`. 1x means about the same speed, 2x means twice as slow, etc. 

| How much slower | Sampler                                                                                                                                                                                                                                                                                                                                                                                          |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1x              | `euler, euler_cfg_pp, euler_ancestral, euler_ancestral_cfg_pp, dpm_fast, dpmpp_2m, dpmpp_2m_cfg_pp, dpmpp_2m_sde, dpmpp_2m_sde_gpu, dpmpp_3m_sde, dpmpp_3m_sde_gpu, ddpm, lcm, ipndm, ipndm_v, deis, res_multistep, res_multistep_cfg_pp, res_multistep_ancestral, res_multistep_ancestral_cfg_pp, gradient_estimation, gradient_estimation_cfg_pp, er_sde, sa_solver, ddim, uni_pc, uni_pc_bh2` |
| 2x              | `heun, dpm_2, dpm_2_ancestral, dpmpp_2s_ancestral, dpmpp_2s_ancestral_cfg_pp, dpmpp_sde, dpmpp_sde_gpu, seeds_2, sa_solver_pece`                                                                                                                                                                                                                                                                 |
| 3x              | `heunpp, seeds_3`                                                                                                                                                                                                                                                                                                                                                                                |
