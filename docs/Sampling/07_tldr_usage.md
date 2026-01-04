# Practical TL;DR of Samplers

## Iteration Speed (How Long 1 Step Takes)

| Relative Speed | Samplers                                                                                                               |
| -------------- | ---------------------------------------------------------------------------------------------------------------------- |
| 1x             | `euler`, `dpmpp_2m` (and everything with `<number>m` in the name), most others                                         |
| 2x             | `heun`, `dpm_2`, `dpmpp_sde`, `seeds_2`, `dpmpp_2s_ancestral`, `sa_solver_pece` (and everything with `2s` in the name) |
| 3x             | `heunpp`, `seeds_3`, `dpm_adaptive`, anything with `3s` in the name                                                    |

For example, this means that in the same time that you run `euler` for 20 steps, it takes about the same time to run `dpm_2` for 10 steps.

`dpm_fast` is a bit special; This sampler switches between the `DPM-3`, `DPM-2`, and `DPM-1` algorithm in a way that the total amount of time spent is the same as `euler` if you set the same amount of steps. Each step may take differing amounts of time.

## Samplers That Change Composition As You Increase Step Count
- `ddpm` 
- `restart` 
- `seeds` 
- `sa_solver(_pece)`
- Any sampler with `ancestral (a)` in the name
- Any sampler with `sde` in the name

Generally, these samplers beat their non-composition-changing counterparts if you use higher step counts.

## SDXL - What Works at Various Step Counts

| Steps | Works Well                                                                                             | Usually Breaks                           |
| ----- | ------------------------------------------------------------------------------------------------------ | ---------------------------------------- |
| 1-12  | Training-based methods like `lcm`, `lightning`, `hyper`                                                | Everything else                          |
| 12-20 | Many (realistically, 6-10 steps will also leave very visible artifacts, but *may* occasionally be ok.) | `ipndm(_v)`, `dpmpp_3m_sde`, `seeds_3`   |
| 20-30 | Mostly everything                                                                                      | `dpmpp_3m_sde` may still leave artifacts |
| 30+   | High-order methods like `ipndm(_v)` / Stochastic methods like `dpmpp_3m_sde`                           | -                                        |

!!!note "But `X` Worked/Broke For Me!"  
    These are very rough estimates. You should see these only as general rules of thumb on what works in low / high step environments.  
    For example, *many* Illustrious derivative models contain DMD2, but they don't tell you. If done well, you'll hardly notice any artifacts, while being able to gen in the very low step regime no problem.
