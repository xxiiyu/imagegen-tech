# "Accuracy / Control"

You've probably seen something along the lines of this in other SD guides:

> `DPM++` is better \[than `Euler`\] for precision and control.

Armed with the perspective that samplers = equation solvers, it's clearer what "precision," "control," and "accuracy," among other similar descriptions really mean - solving the diffusion equation more accurately. 
In math terms, we'd say that the solver is "higher **order.**" You can just remember it as "high order = more complex and accurate (in theory)."

However, we are not looking to solve equations, we're looking to generate nice images. **Numerical accuracy does not directly equate to good-looking images.**

For example, despite the popularity of `euler(_ancestral)`, you may be surprised to find that it's actually the simplest and most inaccurate DE solver. The errors manifest as blurring the output and create a **soft/dreamy visual style** that many find pleasing. 
On the other hand, inaccuracies may also mean **small detail is lost** - distant buildings merging together, hair strands forming blobs, etc. In severe cases, this *might* lead to *worse prompt adherence* overall (the errors are so big that it visibly deviates from your prompt; there's also the factor of the model simply not understanding you, though).

DEs that represent the process of diffusion are often very "[stiff](https://en.wikipedia.org/wiki/Stiff_equation)," especially at high CFG - this increases numerical instability and in practice:
1. makes `adaptive` solvers take many tiny steps, which in turn makes image gen take very long
2. means that higher order solvers might do worse than expected or completely break, especially with low step count

This may be why the community favorites - `euler_ancestral` and `dpmpp_2m` - are "only" first-order and second-order DE solvers respectively but still perform very well.
(And `dpm(pp)` side-steps the stiffness issue using clever math tailored for diffusion equations.)

!!!info
    Did you know that as [originally formulated](https://arxiv.org/pdf/2006.11239), a diffusion model has to take *1000 steps* to generate an image? What we're doing now all seems like "low step count" compared to that!
