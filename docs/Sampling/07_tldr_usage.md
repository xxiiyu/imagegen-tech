# Practical TL;DR of Samplers

## Iteration Speed (How Long 1 Step Takes)

| Relative Speed | Samplers 
| - | - 
| 1x | `euler`, `dpmpp_2m` (and everything with `<number>m` in the name), most others
| 2x | `heun`, `dpm_2`, `dpmpp_sde`, `seeds_2`, `dpmpp_2s_ancestral` (and everything with `2s` in the name)
| 3x | `heunpp`, `seeds_3`, `dpm_adaptive`, anything with `3s` in the name

For example, this means that in the same time that you run `euler` for 20 steps, it takes about the same time to run `dpm_2` for 10 steps.

## Samplers That Change Composition As You Increase Step Count
- `ddpm` 
- `restart` 
- `seeds` 
- `sa_solver(_pece)`
- Any sampler with `ancestral (a)` in the name
- Any sampler with `sde` in the name

Generally, these samplers beat their non-composition-changing counterparts if you use higher step counts.

## SDXL - What Works at Various Step Counts

| Steps  | Works Well | Usually Breaks 
| - | - | -
| 1-5       | Training-based methods like `lcm`, `lightning`, `hyper` | Everything else
| 6-15     |  Many (realistically, 6-10 steps will also leave very visible artifacts, but *may* occasionally be ok.)  | `ipndm(_v)`, `dpmpp_3m_sde`, `seeds_3`
| 16-30    | Mostly everything | `dpmpp_3m_sde` may still leave artifacts
| 30+      | High-order methods like `ipndm(_v)` / Stochastic methods like `dpmpp_3m_sde` | -

!!!note "But `X` Worked/Broke For Me!"  
    These were tested on Amanatsu, a very stable eps-pred SDXL, on a few samples. You should see these only as general rules of thumb on what works in low / high step environments.  
    For example, most SDXL v-pred models require a ton more steps with any sampler, 20+ for only decent results in my experience.
