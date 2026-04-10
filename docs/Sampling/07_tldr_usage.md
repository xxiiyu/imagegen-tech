# Practical TL;DR of Samplers

## Iteration Speed (How Long 1 Step Takes)

| How much slower | Sampler                                                                                                                                                                                                                                                                                                                                                                                          |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1x              | `euler, euler_cfg_pp, euler_ancestral, euler_ancestral_cfg_pp, dpm_fast, dpmpp_2m, dpmpp_2m_cfg_pp, dpmpp_2m_sde, dpmpp_2m_sde_gpu, dpmpp_3m_sde, dpmpp_3m_sde_gpu, ddpm, lcm, ipndm, ipndm_v, deis, res_multistep, res_multistep_cfg_pp, res_multistep_ancestral, res_multistep_ancestral_cfg_pp, gradient_estimation, gradient_estimation_cfg_pp, er_sde, sa_solver, ddim, uni_pc, uni_pc_bh2` |
| 2x              | `heun, dpm_2, dpm_2_ancestral, dpmpp_2s_ancestral, dpmpp_2s_ancestral_cfg_pp, dpmpp_sde, dpmpp_sde_gpu, seeds_2, sa_solver_pece, exp_heun_2_x0, exp_heun_2_x0_sde`                                                                                                                                                                                                                               |
| 3x              | `heunpp, seeds_3`                                                                                                                                                                                                                                                                                                                                                                                |

For example, it takes the same amount of time to run `euler` for 20 steps and to run `dpmpp_sde` for 10 steps.

Notes on more special samplers:

- `dpm_fast` switches between the `DPM-3`, `DPM-2`, and `DPM-1` algorithm in a way that the total time spent is the same as `euler` if you set the same amount of steps, while each step may take differing amounts of time.
- `dpm_adaptive` ignores the scheduler and decides how many steps to run on its own.

## Noisy Samplers That Keep Change Composition
- `ddpm` 
- `restart` 
- `seeds` 
- `sa_solver(_pece)`
- Any sampler with `ancestral (A)` in the name
- Any sampler with `sde` in the name

Generally, these samplers beat their non-composition-changing brethren if you use higher step counts.

## Opinionated Sampler Choices

How many steps to use concretely obviously depends on the model. However in general:

- Note the `euler` family will generally be slightly blurrier. Consider using it if that's preferrable, or using a distilled model, or in other nicher situations.
- Consider non-noisy samplers (ODE) when using low-medium step count.
    - `res_multistep` is usually a sane default for most situations.
    - `gradient_estimation` or `uni_pc` can be a bit sharper / more saturated.
- Consider noisy samplers (SDE) when using medium-high step count.
    - `er_sde` is usually a sane default for most situations.
    - `sa_solver(_pece)` can be a bit sharper / more saturated.
    - As of 16/Mar/2026, in ComfyUI, all `ancestral` samplers are broken on flow matching models except `euler_ancestral` and `dpmpp_2s_ancestral`. This is due to missing scaling by a renoising coefficient.
- Try `_cfg_pp` variants when available to see if you prefer them. Start with `cfg=1` and nudge it slightly higher to e.g. `1.5` if coherency issues appear.
  - As of 16/Mar/2026, in ComfyUI, all `cfg_pp` samplers are broken on flow matching models except `euler_cfg_pp`. This is due to missing a scaling factor of $\alpha_t=1-\sigma_t.$
- Consider `lcm` or a noisy sampler when using distilled `sdxl`, such as `dmd2, hyper, turbo`. (These used techniques distinct from newer distills such as `ZIT, Flux.2-[klein]-base, Flux.2-[klein]`, etc. which is why `sdxl` distills wants lots of noise.)

## Common Misconceptions

> Euler sampler is the worst one.

While `euler` is indeed the "most inaccurate" sampler (details [#here](./01_accuracy.md)), accuracy is not directly related to whether the final output is aesthetically pleasing to you.

> DPM++ 2M takes longer than Euler.

Practically false. `dpmpp_2m` does more calculations yes, but at most that's a few extra *milliseconds* - which is nothing when the time spent running the model each step *takes seconds*.

Since bpth `dpmpp_2m` and `euler` run the model exactly once each step, they are effectively the same speed. This is also why [#iteration speed](#iteration-speed-how-long-1-step-takes) nicely separates into 1x, 2x, 3x etc brackets - samplers that run the model once, twice, or thrice per step fall into those brackets respectively.

> DPM++ 2M uses more VRAM than Euler.

Practically false. `dpmpp_2m` needs to additionally store the previous image latent yes, but those are tiny. Assuming you're using Flux2 VAE, the one that produces the largest latents on the market right now, the latent for a 1024x1024 still only takes ~0.5MB of vram.
