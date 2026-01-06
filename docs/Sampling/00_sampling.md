# What is Sampling?

**Sampling** has already been explained in other sections in this guide. To recap:

- **[#What](../Models/00_models.md#what-does-generating-mean):** sampling means to pick data out according to a data distribution $p_\text{data},$ which assigns probabilities to data. For example, a cat distribution $p_\text{Cat}$ would assign high probability to cats, and everything else like dogs, humans, scenery, ...
- **[#How](../Models/01_diffusion.md#how-2-diffuse):** For diffusion models, solve the corresponding ODE/SDE which describe paths that take Gaussian noise and gradually transform it into realistic images.

## What is a Sampler?

While not all of them were conceived this way, all diffusion samplers can be viewed as SDE/ODE solvers - or in other words, [numerical integration methods](https://en.wikipedia.org/wiki/Numerical_integration).

## [Deep Dive] Sampling Algorithm

Using the Euler method, one can sample from a diffusion model as follows:

!!!info "Algorithm 1: Diffusion Sampling (Euler-Maruyama Method)"
    **Require:** diffusion model $u_\theta,$ step sizes $[h_0,h_1,...,h_{n-1}]$, diffusion coefficient $\eta_t$

    1. Set $t = 0$
    2. Draw a sample $x_0\sim p_{\text{init}}$
    3. **for** $i=0,...,n-1$ **do**
    4. &emsp;Draw random noise $\epsilon \sim \mathcal N(0,1)$
    5. &emsp;$x_{t+h}=x_t+h_iu_\theta(x_t,t)+\eta_t\sqrt{h_i}\epsilon$
    6. &emsp;Update $t\gets t+h_i$
    7. **end for** ($t$ should be $1$ after the loop)
    8. **return** $x_1$

Where $\mathcal N(0,1)$ is the Standard Gaussian distribution. 

A few notes:

- We recover flow matching / ODE sampling by setting $\eta_t=0.$
- One can adapt the above algorithm to other samplers by changing line 5 to using said samplers' update rules rather than Euler's.

To adapt the above into `k-diffusion`, and by extension popular UIs like forge and comfy, a few terminologies need to shift. Mainly:

1. `k-diffusion` works with a data prediction model $x_\theta$ that would predict the clean image directly, rather than $u_\theta$ which predicts the change needed to get there. 
2. `k-diffusion` directly works in a noise schedule $\sigma_t$ rather than step sizes.

These aren't huge issues, as one can be translated into another without much trouble.

Algorithm 2 mostly follows the conventions in Algorithm 1, and thus you may find it easier to follow the changes; Algorithm 2.1 is a more direct inscription of the actual code.

!!!info "Algorithm 2: `k-diffusion` Sampling (Euler-Maruyama Method)"
    **Require:** data prediction model $x_\theta,$ noise schedule $[\sigma_0,\sigma_1,...,\sigma_n]$, diffusion coefficient $\eta$

    1. Draw a sample $x\sim\mathcal N(0, \sigma_0^2)$
    2. **for** $i=0,...,n-1$ **do**
    3. &emsp;Draw random noise $\epsilon\sim\mathcal N(0,1)$
    4. &emsp;Set $h_\text{down},h_\text{up}\gets\text{find\_step\_size}(\sigma_i,\sigma_{i+1},\eta)$ 
    5. &emsp;Set $u_\theta\gets (x-x_\theta(x,\sigma_i))/\sigma_i$
    6. &emsp;Update $x\gets x+h_\text{down}u_\theta+\eta h_\text{up}\epsilon$
    7. **end for** 
    8. **return** $x$

!!!info "Algorithm 2.1: `k-diffusion` Sampling True-to-the-Code (Euler-Maruyama Method)"

    **Require:** data prediction model $x_\theta,$ noise schedule $[\sigma_0,\sigma_1,...,\sigma_n]$, diffusion coefficient $\eta$

    1. Draw a sample $x\sim\mathcal N(0, \sigma_0^2)$
    2. **for** $i=0,...,n-1$ **do**
    3. &emsp;Draw random noise $\epsilon\sim\mathcal N(0,1)$
    4. &emsp;Set $\sigma_\text{down},\sigma_\text{up}\gets\text{get\_ancestral\_step}(\sigma_i,\sigma_{i+1},\eta)$ 
    6. &emsp;Set $d\gets (x-x_\theta(x,\sigma_i))/\sigma_i$
    7. &emsp;Set $dt\gets \sigma_\text{down}-\sigma_i$
    8. &emsp;Update $x\gets x+d\cdot dt+\sigma_\text{up}\epsilon$
    9. **end for** 
    10. **return** $x$

