# Score Matching

!!!warning "UNDER CONSTRUCTION"

## Loss Functions for Diffusion

For diffusion models, there are 3 main types of [#loss](./00_models.md#what-is-a-loss-function)es commonly in use today, each representing a different prediction type. Oversimplifying the equations for intuition, these are:

- $\epsilon:$ Train the model to predict the noise. $L=|\epsilon_t - \text{model}(x_t, t)|^2$ 
- $x_0:$ Train the model to predict the clean image. $L=|x_0 - \text{model}(x_t, t)|^2$
- $v:$ Train the model to predict the "angular velocity." $L=|v_t - \text{model}(x_t, t)|^2$
    - The angular velocity is defined as $v=\alpha_t\epsilon - \sigma_t x_0,$ where $\alpha_t, \sigma_t$ are numbers that depend on which step $t$ you're on. The later in the sampling process = the closer you are to finishing generation = the smaller $t$, the bigger $\alpha_t$ is and the smaller $\sigma_t$ is. Vice versa.
    - You can imagine that as at the very start of sampling, $v$ prediction injects a lot of noise. As the image gets cleaner, the process also starts injecting less and less noise.
    - On the other hand, you can also just imagine $v$ prediction as a mixture of $\epsilon$ prediction and $x_0$ prediction.

While in theory, all 3 of these could lead to a perfectly viable diffusion model, there are practical concerns to each of these.

## Further Learning

- [IE University Lecture (Slides)](https://julioasotodv.github.io/ie-c4-466671-diffusion-models/)
- **Depth First**'s two parter on diffusion (Video):
    - [Part 1](https://www.youtube.com/watch?v=1pgiu--4W3I)
    - [Part 2](https://www.youtube.com/watch?v=Fk2I6pa6UeA)
- [Score Matching (Video)](https://www.youtube.com/watch?v=B4oHJpEJBAA)
