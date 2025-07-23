# "Accuracy / Control"

You've probably seen something along the lines of this in other SD guides:

> `DPM++` is better \[than `Euler`\] for precision and control.

Armed with the perspective that samplers = equation solvers, it's clearer what "precision," "control," and "accuracy," among other similar descriptions really mean - solving the diffusion equation more accurately. 
In math terms, we'd say that the solver is "higher **order.**" You can just remember it as "high order = more complex and accurate (in theory)."

However, we are not looking to solve equations, we're looking to generate nice images. **Numerical accuracy does not directly equate to good-looking images.**

For example, despite the popularity of `euler(_ancestral)`, you may be surprised to find that it's actually the simplest and most inaccurate DE solver. The errors manifest as blurring the output and create a **soft/dreamy visual style** that many find pleasing. 
On the other hand, inaccuracies may also mean **small detail is lost** - distant buildings merging together, hair strands forming blobs, etc. In severe cases, this *might* lead to *worse prompt adherence* overall (the errors are so big that it visibly deviates from your prompt; there's also the factor of the model simply not understanding you, though).

DEs that represent the process of diffusion are often very "[stiff](https://en.wikipedia.org/wiki/Stiff_equation)," especially at high CFG - this increases numerical instability and in practice:
1. makes `adaptive` solvers take many tiny steps, which in turn makes imagegen take very long
2. means that higher order solvers might do worse than expected or completely break, especially with low step count

This may be why the community favorites - `euler_ancestral` and `dpmpp_2m` - are "only" first-order and second-order DE solvers respectively but still perform very well.
(And `dpm(pp)` side-steps the stiffness issue using clever math tailored for diffusion equations.)

!!!info
    Did you know that as [originally formulated](https://arxiv.org/pdf/2006.11239), a diffusion model has to take *1000 steps* to generate an image? What we're doing now all seems like "low step count" compared to that!

## How Does Order Affect Error?

!!!note "Oversimplification"
    I'll be brushing over many details and oversimplifying things for ease of understanding here. For more accurate information on this topic, see [Truncation error](https://en.wikipedia.org/wiki/Truncation_error_(numerical_integration)).

The "order" of a DE solving method measures how much the errors scale down when you decrease the step size - or equivalently, increase the number of steps.

Let's assume for simplicity, that the error of any sampler taking 1 step is 10. As in, by some measure, the difference between the truth and the answer produced by the solver is 10.

Now, let's take `euler`, `heun`, and `bosh3`, which have an order of 1, 2, 3 respectively, and look at the error at various steps:

| steps | `euler` | `heun` | `bosh3`
| - | - | - | -
| 2 | `10 / 2` = 5 | `10 / (2*2)` = 2.5 | `10 / (2*2*2)` = 1.25
| 3 | `10 / 3` ≈ 3 | `10 / (3*3)` ≈ 1.1 | `10 / (3*3*3)` ≈ 0.37
| 4 | `10 / 4` = 2.5 | `10 / (4*4)` = 0.625 | `10 / (4*4*4)` ≈ 0.156
| `n` | `10 / n` | `10 / (n*n)` | `10 / (n*n*n)` 

In general, if the order of a sampler is `O`, and the error it makes when you take 1 step is `E`, then the error if you take `N` steps would be around `E / (N^O)`, where `N^O` means `N` multiplied by itself `O` times, a.k.a exponentiation.

Let's compare `euler` and `heun`. `heun` takes twice as long as `euler` per step, and has an order of 2. Now, let's run them for the same amount of time and see what happens to the error:
- `euler` for 10 steps, then the error is about `10 / 10` = 1
- `heun` for 5 steps (because it takes 2x as long), then the error is about `10 / (5*5)` = 0.4

So, *in theory*, you can get a more accurate result in the same amount of time - or equivalently, get an as accurate result in less time - using higher order samplers, though this only holds true if the extra time needed to achieve higher order isn't too high. 

The high stiffness that most diffusion DEs exhibit also makes high-order samplers do really poorly in low steps (or even just in general). 
