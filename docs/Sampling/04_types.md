# Types of Sampling Methods

## `adaptive`
Samplers that choose their own steps, **ignoring your setting for step count and scheduler.** In some implementations, `steps` may instead be used as the "max steps" before it's forcefully stopped lest it takes too long.

## Stochastic (`SDE`, `ancestral (a)`, `sa_solver`, `Restart`)
Samplers that inject noise back into the image. They never *converge* - with higher and higher step counts, they don't land on 1 final image and keep refining, instead, the composition may drastically change even if it's very late down the line.

The theoretical quality of images generated based on non-stochastic vs. stochastic sampling depends on step count:

- **low steps:** samplers make a few big errors (low steps, high step size). Non-stochastic samplers usually make errors smaller than stochastic samplers if you compare 1 step of each. Thus, **non-stochastic methods do better than stochastic methods in low steps.**
- **high steps:** samplers make many small errors (high steps, small step size), which build up over time. It's now the *accumulated* error affecting image quality the most, and the random noise introduced by stochastic methods can gradually correct them. Thus, **stochastic methods do better than non-stochastic methods in high step counts.**

!!!note "Stochastic Methods In Low Steps"  
    In practice, it's almost always better to use stochastic samplers if you don't care about non-convergence.

    Many new stochastic methods also try to incorporate the best of both worlds, working nicely even in low steps. This includes `Restart`, `er_sde`, `sa_solver`, and `seeds`.

### Why Stochasticity Break Flux and More

SD 3.X, Flux, AuraFlow, Lumina 2, and potentially more to come, were all trained on a schedule which is very sensitive to the variance (a statistical measure) of the data. 

Without careful calibration, chances are that your stochastic sampler makes the variance increase without bound, thus breaking these models. This is why people weren't having luck using anything `ancestral` or `sde` etc. on them.

## `singlestep (s)` / `multistep (m)`

| Feature              | Singlestep (s)                                                         | Multistep (m)                                         |
| -------------------- | ---------------------------------------------------------------------- | ----------------------------------------------------- |
| How it works         | Runs the model multiple times each step for better accuracy            | Considers multiple previous steps for better accuracy |
| Model Calls per Step | `k`                                                                    | 1                                                     |
| Speed (per step)     | `k` times slower than 1 `euler`                                        | Basically as fast as `euler`                          |
| Accuracy             | High                                                                   | Lower than `singlestep`                               |
| Example              | `dpmpp_2s_ancestral` (2 model calls per step = 2x slower than `euler`) | `dpmpp_2m` (same speed as `euler`)                    |

## Implicit / Explicit
2 approaches used to solve DEs. 

Implicit methods solve a harder form of the DE, making them slower but more resistant to stiffness. This means in theory, you can use higher order implicit methods without them breaking, leading to *moar* accuracy. (This is *super* slow, though.)

ALL common samplers are explicit. This includes `euler`, `deis`, `ipndm(_v)`, `dpm(pp)` family, `uni_pc`, `res_multistep`, and more.

Implicit methods are usually not used in favor of those based on [#Exponential Integrators](#exponential-integrators). The quality-speed tradeoff of implicit methods seem to limit their popularity. They're also not found as defaults in popular UIs.

!!!info "Where Can I Find Implicit Samplers?"  
    ComfyUI: [RES4LYF](https://github.com/ClownsharkBatwing/RES4LYF)

### Diagonally Implicit
These are a subset of implicit methods whose difficulty to solve sits between implicit and explicit methods. They're easier to solve, but can't get as accurate.

[Here](https://ntrs.nasa.gov/api/citations/20160005923/downloads/20160005923.pdf)'s a comprehensive review of Diagonally Implicit Runge Kutta (DIRK) methods for the interested.

## Training-Free / Training-Based

Training-free methods are those that you can use without further changing the model. You can simply load the model in and use a training-free sampling method on it and it'll (probably) work. These include familiar faces like `euler`, `dpmpp_2m`, etc.

Training-based methods require further modification of the model. This would include LCM, Lightning, Hyper, etc. where in order to use them you need to download a separate version of a model or use a LoRA. The upside is generating images in vastly lower steps like 8 or 4.

Though sometimes they come along with a dedicated sampler like `lcm`, they may work in tandem with training-free samplers in general. For example, you can probably use any of `euler`, `dpmpp_2m`, and more on a model with a Hyper LoRA applied.

## Exponential Integrators

**Exponential Integrators (EI),** sometimes called **Exponential Time Differencing (ETD)** among its many other names, became active research at least 10-20 years before diffusion models were developed. It breaks DEs down into a stiff "linear part," and the less-stiff "non-linear part":

$$
\frac{du}{dt} = \underbrace{Au}_{\text{Linear(very stiff)}} + \underbrace{f(u, t)}_{\text{Non-linear}}
$$

The stiff linear part can then be solved exactly, leaving only the non-linear part to the sampler/solver. Compared to traditional methods, exponential integrators eliminate all errors associated with approximating the stiff linear part, making them great for stiff diffusion DEs.

`deis`, `dpm(pp)`, `unipc`, `res` (and probably more) all use this technique.

You can find a technical evaluation of some exponential integrators on physics problems [here](https://ora.ox.ac.uk/objects/uuid:cc001282-4285-4ca2-ad06-31787b540c61/files/m611df1a355ca243beb09824b70e5e774). (as well as something called "Lawson methods," or "Integrating Factor (IF) methods.")
