# What is a Model?

A **generative model** learns to create new data similar to the training data. This is in contrast to a **discriminatory model** which learns to look at a new data point and predict a label or number. For images, you can imagine that:

- generative model: learns to make new images
- discriminatory model: learns to classify it as cat or dog / predict auction price

In modern day, generative models are usually deep neural networks, including GANs, VAEs, and of course diffusion models.

## What Does "Generating" Mean?

Researchers mathematically formalize the idea of generating data as sampling from a **probability distribution $p_\text{data}$.** A probability distribution basically associates a probability with each possible outcome of an event.

!!!note "Illustrative Example"
    Imagine rolling a fair die. The possible outcomes are one of the 6 faces, with a number 1-6 on that side. The probability of each outcome is equal. The probability distribution of the dice roll $p_\text{dice}$ could thus be described as the following table:

    | $\text{roll}$ | $1$ | $2$ | $3$ | $4$ | $5$ | $6$ |
    | - | - | - | - | - | - | - | 
    | $p_\text{dice}(\text{roll})$ | $\frac16$ | $\frac16$ | $\frac16$ | $\frac16$ | $\frac16$ | $\frac16$ |

It's simple enough to come up with $p_\text{data}$ for rolling dice or flipping coins. 

A mathematically simple and commonly used distribution is the **Gaussian**, or the **normal distribution**, denoted as $\mathcal N(\mu, \sigma^2),$ where $\mu, \sigma$ are variables that change its behavior.  
The unit Gaussian $\mathcal N(0, 1)$ is also the **noise distribution** that people talk about in diffusion (usually).

For real world data however, the underlying $p_\text{data}$ is so complex that it's close to impossible to know what it is. Thus, the goal of generative modeling is to allow us to sample from $p_\text{data},$ when $p_\text{data}$ itself could be extremely complicated.

### $p_\text{data}$ for Images

Imagine 2x2 images with 1 grayscale channel, where the channel value goes from 0-255. Each image can be thought of as a $2\times2\times1$ list of numbers. For example, this could be an image: 

$\begin{bmatrix}[100] & [0] \\ [0] & [200]\end{bmatrix}$

We can also assign each image with a probability, thus creating a probability distribution for images:

| $\text{Image}$ | $\begin{bmatrix}[100] & [0] \\ [0] & [200]\end{bmatrix}$ | $\begin{bmatrix}[42] & [0] \\ [0] & [42]\end{bmatrix}$ | ... |
| - | - | - | - |
| $p_\text{data}(\text{Image})$ | $\frac1{1234}$ | $\frac{11}{1337}$ | ... |

Real world images are obviously more complex, having way bigger widths and heights, and having multiple color channels like RGB. However, the principles stay fundamentally the same. 

Now, let's say we try to generate cats. One can imagine that this probability distribution, $p_\text{CatImages},$ assigns a higher probability to natural cat images, and low probability for other images like dogs. Thus, if we can sample from $p_\text{CatImages},$ that's the same as being able to generate new, high quality cat images.

## What is a Loss Function?

At a high level, we always say we "train a model to learn something." Well... how do we quantify how much the model has "learned"?

The **loss (function)**, also called the **cost** or **objective**, is a function people design to measure how well the model performs on a task. Conventionally, a lower loss means the model is performing better. Training a model could also be referred to as minimizing the loss.

!!!note "Illustrative Example"
    Let's say you're trying to predict house prices. A simple and common loss function for this task could be $L=|\text{TruePrice} - \text{PredictedPrice}|^2$, the square of the difference between the true price and the predicted price. As you can imagine, trying to minimize the loss $L$ is the same as trying to reduce the difference between the true price and the prediction, or in other words make the prediction more accurate. 

!!!info "Why Minimize the Square?"
    You may ask, why minimize the difference squared and not just the difference? An intuitive explanation is this: If the difference between the true price and the predicted price is big, then the square will extrapolate it to be bigger. This means we punish the model way harder if it makes a wildly inaccurate prediction.
    
    There are more math-heavy reasons rooted in statistics, the details of which are out of the scope of this article. (For those interested in searchable keywords, minimizing the squared difference - the L2 loss - corresponds to maximizing the likelihood.)

## References
