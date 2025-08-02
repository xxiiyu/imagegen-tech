# "Accuracy / Control"

You've probably seen something like this in other diffusion guides:

> `DPM++` is better \[than `Euler`\] for "accuracy" / "precision" "control".

Viewing sampling through the lens of solving the diffusion **differential equation (DE)**, it becomes clearer what this could mean - to solve the DE more accurately. In math terms, we'd say that the more accurate solver is higher **order.**

However, note we're simply looking to generate nice images. Numerical accuracy does not directly translate to good-looking images.

For example, the ever-popular `euler(_ancestral)` is actually the most inaccurate sampler there is. The errors it make manifest as blurring the output and create a soft/dreamy visual style that many find pleasing. 

Inaccuracies don't always play out nicely though. For example, some potential drawbacks may include:

- Small detail is lost: backgrounds merging into meaningless blobs, hair strands losing definition, etc.
- Worse prompt adherence: In severe cases, the errors could become so big that it actively hurts how much the image follows your prompt.

On the other hand, diffusion DEs are very [stiff](https://en.wikipedia.org/wiki/Stiff_equation), especially at high CFG - this increases numerical instability and in practice:

1. Makes `adaptive` solvers take tiny steps = very long to generate an image
2. Makes higher order solvers unstable and do worse, completely breaking in severe cases, especially at low steps

This may be why the community favorites - `euler_ancestral` and `dpmpp_2m` - are "only" first-order and second-order respectively.  
(And `dpm(pp)` uses a clever math trick called the [exponential integrator](https://en.wikipedia.org/wiki/Exponential_integrator) to make it less stiff)

!!!info
    Did you know that as originally formulated in [DDPM](https://arxiv.org/pdf/2006.11239), a diffusion model has to take *1000 steps* to generate an image? What we're doing now all seems like "low step count" compared to that!

## How Does Order Affect Error?

!!!note "Oversimplification"
    I'll be brushing over many details and oversimplifying things for ease of understanding here. For more accurate information on this topic, see [truncation error](https://en.wikipedia.org/wiki/Truncation_error_(numerical_integration)).

**Order** measures how much the errors a sampler makes scale down when you decrease the step size - or equivalently, increase the number of steps.

Let's assume for simplicity that the error of any sampler taking 1 step is 10. As in, by some measure, the difference between the truth and the answer produced by a sampler is 10.

Now, let's take `euler`, `heun`, and `bosh3`, which have an order of 1, 2, 3 respectively, and look at the error at various steps:

| steps | `euler` | `heun` | `bosh3`
| - | - | - | -
| 2 | `10 / 2` = 5 | `10 / (2*2)` = 2.5 | `10 / (2*2*2)` = 1.25
| 3 | `10 / 3` ≈ 3 | `10 / (3*3)` ≈ 1.1 | `10 / (3*3*3)` ≈ 0.37
| 4 | `10 / 4` = 2.5 | `10 / (4*4)` = 0.625 | `10 / (4*4*4)` ≈ 0.156
| `n` | `10 / n` | `10 / (n*n)` | `10 / (n*n*n)` 

In general, if the order of a sampler is `O`, and the error it makes when you take 1 step is `E`, then the error if you take `N` steps would be around `E / (N^O)`.

Let's compare `euler` and `heun`. `heun` takes twice as long as `euler` per step, and has an order of 2. Now, let's run them for the same amount of time and see what happens to the error:
- `euler` for 10 steps, the error is about `10 / 10` = 1
- `heun` for 5 steps (because it takes 2x as long), the error is about `10 / (5*5)` = 0.4

So in theory, you can get better bang (accuracy) for your buck (time) by using higher order samplers. 

The high stiffness of diffusion DEs makes out-of-the-box high-order samplers do poorly though. Using stiff-resistant techniques is recommended, for example:

- Implicit methods like [`gauss-legendre`](https://en.wikipedia.org/wiki/Gauss%E2%80%93Legendre_method)
- Exponential integrators like `deis` (and most ODE samplers that came after, including the `dpm(pp)` family, `res` (Refined Exponential Solver), etc.)
