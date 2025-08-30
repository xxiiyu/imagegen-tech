# What is a Model?

A **generative model** learns to create new data similar to the training data. This is in contrast to a **discriminatory model** which learns to look at a new data point and predict a label or number. For images, you can imagine that:

- generative model: learns to make new images
- discriminatory model: learns to classify it as cat or dog / predict auction price

In modern day, generative models are usually deep neural networks, including GANs, VAEs, and of course diffusion models.

## What Does "Generating" Mean?

Researchers mathematically formalize the idea of generating data as sampling from a **probability distribution $p_\text{data}$.** A probability distribution is just something that assigns each possible outcome of an event with the chance of it happening. More specifically, $p_\text{data}$ is a function that takes in a data point and gives you how likely it is to occur.

!!!note "Illustrative Example"
    Imagine rolling a fair die. The possible outcomes are one of the 6 faces, with a number 1-6 on that side. The probability of each outcome is equal. The probability distribution of the dice roll $p_\text{dice}$ could thus be described as the following table:

    | $\text{roll}$ | $1$ | $2$ | $3$ | $4$ | $5$ | $6$ |
    | - | - | - | - | - | - | - | 
    | $p_\text{dice}(\text{roll})$ | $\frac16$ | $\frac16$ | $\frac16$ | $\frac16$ | $\frac16$ | $\frac16$ |

Let's say we try to generate cats. One can imagine there that there exists an underlying probability distribution, $p_\text{CatImages},$ which assigns a higher probability to natural cat images, and low probability for other images like dogs. If we can sample from $p_\text{CatImages},$ that's the same as being able to generate new, high quality cat images.

The goal of generative modeling can now be formalized as to allow us to sample from $p_\text{data},$ when $p_\text{data}$ itself could be extremely complicated.

### $p_\text{data}$ for Images

Imagine 2x2 images with 1 grayscale channel, where the channel value goes from 0-255. Each image can be thought of as a $2\times2\times1$ list of numbers. For example, this could be a black and white image where the top left pixel has brightness 100, bottom right 200, and the rest completely dark:

$\begin{bmatrix}[100] & [0] \\ [0] & [200]\end{bmatrix}$

We can also assign each image with a probability, thus creating a probability distribution for images:

| $\text{Image}$ | $\begin{bmatrix}[100] & [0] \\ [0] & [200]\end{bmatrix}$ | $\begin{bmatrix}[42] & [0] \\ [0] & [42]\end{bmatrix}$ | ... |
| - | - | - | - |
| $p_\text{data}(\text{Image})$ | $\frac1{1234}$ | $\frac{11}{1337}$ | ... |

Real world images are obviously more complex, having way bigger widths and heights, and having multiple color channels like RGB, resulting in $W\times H\times 3$ list of numbers. However, the principles stay fundamentally the same. 

### Why Modeling $p_\text{data}$ is a Problem

Great! So, just train a neural network to learn $p_\text{data},$ right? Well... it's not so easy. To see why, let's try designing a neural network that represents a probability distribution. 

Let's call our neural network $\text{NN}(\text{Input}),$ and let's call the distribution that $\text{Input}$ comes from $p_\text{init}.$ $p_\text{init}$ can pretty much be anything; You could for example train a neural network to turn cat images into dog images if you really wanted to.

In practice, $p_\text{init}$ is usually the **gaussian** distribution, aka the **noise** distribution. The benefit is that the gaussian is very simple, and we know how generate infinite samples from it. So if our neural network learns to turn $p_\text{init}\to p_\text{data},$ *and* we know how to generate infinite samples of $p_\text{init},$ we can now generate infinite samples of $p_\text{data}.$ Neat!

!!!note "Assumptions"
    Since it's almost always the case that $p_\text{init}$ is the gaussian, I'll be making this assumption in this guide from here on.

Since our outputs should be probabilities, these outputs should all be positive, as negative probability doesn't really make sense, right? A conventional way to do this is to transform the output using the exponential function, i.e. turn it into $e^{-\text{NN}(\text{Input})}.$ The details are mathy, but the important part is that $e^{-\text{NN}(\text{Input})}$ is always positive.

Next, recall that $p_\text{data}$ gives you the chance of a specific outcome of an event. Thus, we should expect that the *sum* of all these chances - that there being *an* outcome, *any* outcome - be 1. To do this, we can divide our outputs by a magical constant $Z$. Our output is now $\frac{e^{-\text{NN}(\text{Input})}}Z.$ $Z$ is called the **normalizing constant** in literature.

To recap:

- Neural network outputs $\text{NN}(\text{Input})$ which should be similar to $p_\text{data},$ where $\text{Input}$ comes from some initial distribution $p_\text{init},$ usually the gaussian (the noise distribution).
- Probabilities should be positive, so we instead use $e^{-\text{NN}(\text{Input})}$ as the output
- The sum of probabilities should be 1, so we instead use $\frac{e^{-\text{NN}(\text{Input})}}Z$ as the output

The problem lies in $Z,$ which is next to impossible to calculate for a general neural network. People have come up with ways around this of course. Approaches prior to diffusion can be split into 3 categories, referencing [this talk](https://www.youtube.com/watch?v=wMmqCMwuM2Q) / [blog](https://yang-song.net/blog/2021/score/) by Yang Song, one of the pioneers of modern diffusion:

1. **Approximate $Z$:** Calculating $Z$ exactly is hard, so just use an approximation. One problem is this approximation is *still* expensive to calculate.
2. **Restrict the Neural Network:** Restrict the neural network to specific architectures so $Z$ can be calculated. Obviously this limits our freedom in designing the neural network. Examples include autoregressive models.
3. **Model the Generation Process Only:** Skip trying to model $p_\text{data}$ entirely, just model a process that can generate new data. This usually involves using adversarial loss, which is highly unstable and hard to train with, plus [the result might deviate from $p_\text{data}$](https://arxiv.org/abs/1706.08224). Examples include GANs.

## What is a Loss Function?

At a high level, we always say we "train a model to learn something." Well... how do we quantify how much the model has "learned"?

The **loss (function) $L$**, also called the **cost** or **objective**, is a function people design to measure how well the model performs on a task. Conventionally, a lower loss means the model is performing better. Training a model could also be referred to as minimizing the loss.

!!!note "Illustrative Example"
    Let's say you're trying to predict house prices. A simple and common loss function for this task could be $L=|\text{TruePrice} - \text{PredictedPrice}|^2$, the square of the difference between the true price and the predicted price. As you can imagine, trying to minimize the loss $L$ is the same as trying to reduce the difference between the true price and the prediction, or in other words make the prediction more accurate. 

!!!info "Why Minimize the Square?"
    You may ask, why minimize the difference squared and not just the difference? An intuitive explanation is this: If the difference between the true price and the predicted price is big, then the square will extrapolate it to be bigger. This means we punish the model way harder if it makes a wildly inaccurate prediction.
    
    There are more math-heavy reasons rooted in statistics, the details of which are out of the scope of this article. (For those interested in searchable keywords, minimizing the squared difference - the L2 loss - corresponds to maximizing the likelihood, under the assumption that the random errors are normally distributed.)

### What is Adversarial Loss?

**Adversarial** loss generally refers to when practitioners pit two neural networks against each other.

For example, in image genereating GANs, two models are simutaneously trained at once - the generator $G$, and the discriminator / adversary $A$:

- $G$ tries its best to create realistic images and fool $A$.
- $A$ tries its best to distinguish between real and generated images. 
This is a sound approach, and GANs have been SOTA in terms of generative modeling. It comes with its own problem though, most prominently that it's very hard to balance $G$ and $A$. For example:

- $G$ only learns how to exploit $A$'s defects, creating "images" that trick $A$ but are completely unnatural.
- $G$ only learns a few types of images that $A$ is less certain about, destroying output variety.
- $A$ is too good at discerning real vs. generated that makes it impossible for $G$ to learn from gradient descent.
- $G$ and $A$ end up in an infinite XYZ cycle. $G$ learns to generate only X, so $A$ learns to classify that as generated; $G$ then learns to only generate Y, so $A$ classifies all Y as generated, repeat.

Several training rules, different types of losses, regularization techniques... have been proposed *just* to attempt to solve this problem in GANs.
