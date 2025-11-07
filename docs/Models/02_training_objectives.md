# Training Diffusion Models

!!!note "Recap: [#How Diffusion Generates Images](./01_diffusion.md#how-2-diffuse)"
    A diffusion model learns "paths" that begin at some initial point $x_1$ which is pure noise, and ends at a data point $x_0$ from $p_\text{data}$ (for example a cat image from $p_\text{CatImages}$). By "following" these learned paths, gaussian noise can be transformed into cat images.
    
    Mathematically, these paths are described by differential equations (DE), coming in two types: ordinary (ODE) and stochastic (SDE). To follow a path is to solve the corresponding DE.

## Flow and Score Matching Loss

A diffusion model usually is designed to have two inputs, a noisy image $x$ and the timestep $t.$ To train the diffusion model, there are two common options, **flow matching** or **score matching**:

| Aspect | Flow Matching | Score Matching |
| ------ | ------------- | -------------- |
| Type   | ODE           | SDE            |
| Loss   | $\|u_\text{predicted}(x,t)-u_\text{true}(x,t)\|^2$ | $\|s_\text{predicted}(x,t)-s_\text{true}(x,t)\|^2$ |
| Predicted Quantity | Think of $u$ as the "velocity." Given your position ($x$) and how far along the path you are ($t$), $u$ is how fast and in which direction to walk to eventually get to the path's end. | $s$ is the **score,** defined as the gradient of the log of the probability distribution $s(x,t)=\nabla_x\log p_t(x,t).$ Note that technically SDE needs both $u$ and $s$ to work. 
| Under Practical Conditions...\* | $u_\text{true}(x,t)=x_0-x_1,$ which is very simple and stable to train on. (Also relating to something called the Optimal Transport in literature) | You don't actually need to train two models. One option is to make the neural network output two things ($u$ and $s$) at the same time. Also see below. |
| Models | Most modern models use this, like `SD 3.X`, `Flux`, `Lumina`, `Qwen-Image`, `Wan`, etc. | The original SD releases are noise predictors, like `SD 1.X`, `SDXL`, etc.

\*Stil under the same practical conditions: 

- $u$ and $s$ can be converted between each other. 
    - This means you can e.g. use SDE sampling with a flow matching model by doing the correct math conversion; A sampler designed for one can be used on the other, etc.
    - Another way to bypass needing two models for score matching, is only train a $u$ predictor then turn it into $s$ with math afterwards, or vice versa.
- A score matching model is equivalent to a **noise prediction** model which predicts the noise to remove from images. Thus, these models are also called **denoising diffusion** models.

Since the "practical conditions" are almost always met in practice, and basically all models on the market satisfy them, I'll be assuming they're true from here on. 

!!!info "What are these practical conditions?"
    I've skipped the details in the main text since it's math heavy, and you can assume that they're satisfied most of the time anyway, but the conditions are that:
    
    The models have Gaussian probability paths, which mathematically have the form of $\mathcal N(\alpha_t z; \beta_t^2I_d),$ where $\alpha, \beta$ are noise schedulers (monotonic, continuously differentiable, and $\alpha_1=\beta_0=0$ and $\alpha_0=\beta_1=1$).

## Hurdles in Score Matching

### Score Matching to Noise Prediction

When an image is close to being clean, score matching loss becomes numerically unstable and training breaks. Remember that I'm assuming practical conditions, then the score matching loss becomes the following (simplified):

$$
L=\underset{\substack{\downarrow \\ \text{Near 0 when} \\ \text{image is} \\ \text{almost clean}}}{\color{red}\frac1{\beta_t^2}}|\beta_ts_\text{predicted}(x,t)+\epsilon|^2\Rightarrow\text{divide by 0 error}
$$

Thus, a "score matching" model is very often reparameterized (basically, changed) and trained on a different but still mathematically equivalent objective. 

DDPM drops the red part of the original loss, and reparameterizes the score matching model into a noise prediction model (**$\epsilon$-pred, eps-pred**). eps-pred saw widespread adoption afterwards.

$$L=|\epsilon_\text{predicted}(x,t)-\epsilon|^2$$

### Noise Prediction to (Tangential) Velocity Prediction

eps-pred becomes a problem again in few-steps sampling. At the extreme of 1 step, we're trying to generate a clean image from pure noise, however the eps-pred model *only* predicts the *noise to remove* from an image. Removing noise from pure noise results in... nothing. Empty. Oops, that's bad.

That's the problem the authors of [this](https://arxiv.org/pdf/2202.00512) work faced. They propose a few reparameterizations that fix this, the most influential of which being **(tangential) velocity prediction (v-pred):**  

$$L=|v_\text{predicted}(x,t)-v|^2,\quad v=\alpha_t\epsilon - \beta_t x_0$$

!!!note "Tangential Velocity $v$ and Velocity $u$"
    You might remember that there was also a "velocity" $u,$ that being what the flow matching models predict. On the other hand, v-pred is also often also called velocity prediction. How do they relate to each other?

    Confusingly, they really don't. v-pred comes from an angular parameterization, where you can find a visual in the same paper on [page 15](https://arxiv.org/pdf/2202.00512#page=15). 

Essentially, this loss is saying that at high noise levels, the v-pred model should focus on trying to make an image, and at low noise levels it should instead focus on removing the remaining noise.

### "Current Schedules are Bad"

It was [found](https://arxiv.org/pdf/2305.08891) that for most noise schedules at the time, $x_1$ which is supposed to be pure gaussian noise still left a bit of the data in it, resulting the model learning very low frequency information when it shouldn't have. 

It's likely that models suffering from flawed training schedules learned the average value of the channels correlates with the average brightness of the clean image. Then since when sampling we start with pure gaussian noise (whose channel averages are 0), models always generated images with around the same brightness.

Fixing this was pretty simple, just make $x_1$ actually pure noise when training. They also recommend switching to v-pred, since again eps-pred can't learn from pure noise images.
