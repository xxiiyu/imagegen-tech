# Training Diffusion Models

!!!note "Recap: [#How Diffusion Generates Images](./01_diffusion.md#how-2-diffuse)"
    A diffusion model learns things called **probability paths,** that begin at some initial point $x_1=\epsilon$ which is pure noise, and ends at a data point $x_0$ from $p_\text{data}$ (for example a cat image from $p_\text{CatImages}$). By "following" these learned paths, Gaussian noise can be transformed into cat images.
    
    Mathematically, these paths are described by differential equations (DE), coming in two types: ordinary (ODE) and stochastic (SDE). To follow a path is to solve the corresponding DE.

## Flow and Score Matching Loss

Depending on if you wish to model the transformation from noise to image as an ODE or SDE, we can arrive at 2 objectives: **flow matching** and **score matching.** A diffusion model takes 2 inputs, a noisy image $x_t$ and the timestep $t,$ and tries to minimize the chosen loss.

| Aspect             | Flow Matching                                                                                                                                                                                    | Score Matching                                                                                                                                                                              |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Type               | ODE                                                                                                                                                                                              | SDE                                                                                                                                                                                         |
| Loss               | $\|u_\text{predicted}(x,t)-u_\text{true}(x,t)\|^2$                                                                                                                                               | $\|s_\text{predicted}(x,t)-s_\text{true}(x,t)\|^2$                                                                                                                                          |
| Predicted Quantity | $u$ is the **velocity.** Given your position ($x$) and how far along the path you are ($t$), $u$ is how fast and in which direction to walk to eventually get to the path's end (a clean image). | $s$ is the **score,** defined as the gradient of the log of the probability distribution $s(x,t)=\nabla_x\log p(x,t).$ $s$ points in the direction where the image is most likely to exist. |
| Models             | Most modern models use this, like `SD 3.X`, `Flux`, `Lumina`, `Qwen-Image`, `Wan`, etc.                                                                                                          | The original SD releases are noise predictors, like `SD 1.X`, `SDXL`, etc.                                                                                                                  |

In practice, because we're trying to model the transformation from (gaussian) noise to clean image, this has many positive implications. Important implications include:

1. $u$ and $s$ are **mathematically equivalent** and can be **converted from one to another.**
1. **For flow matching:** This can simplify the training target down to $u_\text{true}(x,t)=x_0-\epsilon,$ or in other words, the difference between a sample of pure noise $\epsilon$ and a completely clean image $x_0.$ This is very easy and stable to train on.
1. **For score matching:** Since you can calculate $u$ from $s$, this means we only need to train the model to learn the score $s;$ Whereas in the most general case you need both for simulating SDE.  
Additionally, a score matching model is equivalent to a **noise prediction** model which predicts the noise to remove from images. Thus, these models are also called **denoising diffusion** models.
1. While flow matching/score matching models are trained to learn ODE/SDEs respectively, due to their equivalence, **you can use both ODE and SDE samplers on the either of them.**

???note "Oversimplifications, Differing Notation, and More Details" 
    The above section is oversimplified for clarity, and uses a specific notation that may be different to other literature.

    1. $x_0-x_1$ is not *the* flow matching target, but *a* target out of many possible choices. It is however the most common, and the one based on Optimal Transport.
    2. Some works may use a reversed time notation, where the "start" is the clean image and the "end" is the pure noise. Early works deriving diffusion from Markov Chains also may use timesteps $t$ from $T=1000\to0$ rather than $0\to1.$  
    In any case, the general idea of learning paths whose 2 ends are data and noise, and walking this path to transform between the 2 stay the same.

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

### ["Current Schedules are Bad"](https://arxiv.org/pdf/2305.08891)

Most training noise schedules at the time failed to make $x_1$ pure Gaussian noise, leaving behind some residual data. This caused models to learn low-frequency information that they shouldn't have, such as there being a relationship between a clean image’s average brightness and $x_1$'s channel averages.

As a result, during sampling, since we always start from true Gaussian noise (which has channel averages of zero), the model consistently generate images with similar brightness (the "uniform lighting curse").

The fix was straightforward: We ensure at the final timestep, $x_1$ is actually pure noise. Or in technical jargon, we use a **Z**ero **T**erminal **S**ignal-to-**N**oise **R**atio (**ZTSNR**) schedule. 

The authors also recommend using v-prediction instead of epsilon-prediction, since the latter can't learn from pure-noise inputs.

They also find that a "trailing" schedule is more efficent than others. In common UIs, that means switching from the `normal` schedule to the `sgm_uniform` schedule.

!!!info "Why is it called ZTSNR?"
    It's a mouthful, but it isn't that difficult to understand once you break the words apart.

    - "Terminal" simply means "final."
    - "Signal-to-Noise Ratio" (SNR) is just the ratio between the information and the noise, $\frac{\text{signal}}{\text{noise}}.$ We want pure noise, so that means there's 0 signal. $\frac0{\text{noise}}=0.$

    So "Zero Terminal Signal-to-Noise Ratio" simply means at the last (terminal) timestep, there is no signal and only noise (SNR = 0).
