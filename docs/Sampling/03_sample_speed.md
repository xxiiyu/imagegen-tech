# How Fast is My Sampler?

The most computationally expensive thing you can do when diffusing is running the neural network. Thus, **the iteration speed of a sampler is mostly determined by how many times it needs to run the neural network per step.** Running the neural network is also sometimes called "calling the model," doing a "model call," etc.

This is also why you don't see people in research comparing sampling methods in number of steps, but rather Number of Function Evaluations (NFE), which is how many times the model was run. It wouldn't make practical sense to compare say `euler` and `heun` both at 20 steps, because the latter would have twice the NFE and run for twice as long.

!!!note "Steps is Not Time Spent"
    Be on the lookout for potentially misleading statements, such as: "`dpmpp_sde` is great for low steps sampling." While technically true, `dpmpp_sde` does 2 model calls per step, which means that in the time `dpmpp_sde` runs for 5 steps, you could've run say `dpmpp_2m` for 10 steps.  
    If you want to make speed-quality comparisons, prefer measuring in NFE over step count. Make sure you don't accidentally compare something like `dpmpp_2s_ancestral` with `dpmpp_2m` at the same amount of steps.
