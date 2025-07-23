# How Does Order Affect Error?

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
