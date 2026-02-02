# Introduction To Generative Models

A **generative model** learns to create new data similar to the training data. This is in contrast to a **discriminative model** which learns to look at a new data point and predict a label or number. For images, you can imagine that:

- generative model: learns to make new images
- discriminative model: learns to classify given images as cat or dog

In modern day, generative models are usually deep neural networks, including GANs, VAEs, Language Models, and of course diffusion models.

## What Does "Generating" Mean?

"Generating" new data, like images, is formalized as **sampling from a probability distribution $p_\text{data}$.** 

- A probability distribution assigns probabilities to a set of specific objects. For example, a probability distribution for cat images $p_\text{Cat}$ would assign a high probability to cats, but a low probability to dogs, humans, plants, and everything else.
- Sampling means generating a specific object from that distribution according to its assigned probabilities. In the case of $p_\text{Cat}$, sampling would most often produce an image of a cat because it has a high probability, while images of dogs or humans would only appear very rarely, if ever.

???note "Illustrative Example - Dice"
    Imagine rolling a fair die. The result would be a number 1-6, all with equal chance. The probability distribution of the dice roll $p_\text{dice}$ could thus be described as the following table:

    | $\text{roll}$                | $1$       | $2$       | $3$       | $4$       | $5$       | $6$       |
    | ---------------------------- | --------- | --------- | --------- | --------- | --------- | --------- |
    | $p_\text{dice}(\text{roll})$ | $\frac16$ | $\frac16$ | $\frac16$ | $\frac16$ | $\frac16$ | $\frac16$ |

???note "Illustrative Example - Images"
    Imagine 2x2 images with 1 grayscale channel, where the channel value goes from 0-255. Each image can be thought of as a $2\times2\times1$ list of numbers. For example, this could be a black and white image where the top left pixel has brightness 100, bottom right 200, and the rest completely dark:

    $\begin{bmatrix}[100] & [0] \\ [0] & [200]\end{bmatrix}$

    Assigning each image with a probability creates a probability distribution for these grayscale images:

    | $\text{Image}$                | $\begin{bmatrix}[100] & [0] \\ [0] & [200]\end{bmatrix}$ | $\begin{bmatrix}[42] & [0] \\ [0] & [42]\end{bmatrix}$ | ... |
    | ----------------------------- | -------------------------------------------------------- | ------------------------------------------------------ | --- |
    | $p_\text{data}(\text{Image})$ | $\frac1{1234}$                                           | $\frac{11}{1337}$                                      | ... |

    Real world images are obviously more complex, having dynamic widths $W$ and heights $H$, with multiple color channels like RGB. This results in $W\times H\times 3$ list of numbers. However, the principles stay fundamentally the same. 

### Why Modeling $p_\text{data}$ is a Problem

Great! So, just train a neural network to learn $p_\text{data},$ right? Well... it's not so easy.

First, the how. The modern solution is to train a model to take in **Gaussian noise** and output a realistic image. Since the Gaussian is very simple and easy to make in infinite supply, an endless stream of realistic images can be created by feeding the noise into said model. Neat!

Second, the challenge: It's extremely difficult to ensure that the neural network outputs actual probabilities, and not some meaningless numbers. 

Diffusion models sidestep this issue by not trying to model $p_\text{data}$ directly, but an intermediate quantity which can be used to recover $p_\text{data}$ at a reasonable accuracy once learned.

## What is a Loss Function?

At a high level, people always say they "train a model to learn something." Well... how does one quantify how much the model has "learned"?

The **loss (function) $L$**, also called the **cost** or **objective**, is a function people design to measure how well the model performs on a task. Conventionally, a lower loss means the model is performing better. Training a model could also be referred to as minimizing the loss.

???note "Illustrative Example - House Price Prediction"
    Let's say you're trying to predict house prices. A simple and common loss function for this task could be $L=|\text{TruePrice} - \text{PredictedPrice}|^2$, the square of the difference between the true price and the predicted price. As you can imagine, trying to minimize the loss $L$ is the same as trying to reduce the difference between the true price and the prediction, or in other words make the prediction more accurate. 

    ???info "Why Minimize the Square?"
        You may ask, why minimize the difference squared and not just the difference? An intuitive explanation is this: If the difference between the true price and the predicted price is big, then the square will extrapolate it to be bigger. This means we punish the model way harder if it makes a wildly inaccurate prediction.
        
        There are more math-heavy reasons rooted in statistics, the details of which are out of the scope of this article. (For those interested in searchable keywords, minimizing the squared difference - the L2 loss - corresponds to maximizing the likelihood, under the assumption that the random errors are normally distributed.)

### What is Adversarial Loss?

**Adversarial** loss generally refers to when practitioners pit two neural networks against each other.

For example, in image genereating GANs, two models are simutaneously trained at once - the generator $G$, and the discriminator / adversary $A$:

- $G$ tries its best to create realistic images and fool $A$.
- $A$ tries its best to distinguish between real and generated images. 

This is a sound approach, and GANs have been SOTA in terms of generative modeling. It comes with its own problem though, most prominently that it's very hard to balance $G$ and $A$. For example:

- $G$ only learns how to exploit $A$'s defects, creating "images" that trick $A$ but are completely unnatural.
- $G$ only learns a few types of images that $A$ is less certain about, destroying output variety. (**Mode Collapse**)
- $A$ is too good at discerning real vs. generated that makes it impossible for $G$ to learn from gradient descent.
- $G$ and $A$ end up in an infinite XYZ cycle. $G$ learns to generate only X, so $A$ learns to classify that as generated; $G$ then learns to only generate Y, so $A$ classifies all Y as generated, repeat.

Several training rules, different types of losses, regularization techniques... have been proposed *just* to attempt to solve this problem in GANs.

---

## Deep Dives

### $p_\text{data}$ Model Specification

A generative model is trained to take a data point from an initial probability distribution $p_\text{init},$ and output a data point from a target probability distribution $p_\text{data}.$

In sections above, the Gaussian was $p_\text{init},$ while the distrubtion underlying realistic images was $p_\text{data}.$ However, the entire framework is more general: it is theoretically possible to e.g. train a model that takes cats from $p_\text{Cat}$ and transform them into dogs from $p_\text{Dog}.$

Diffusion models learn a transformation that would warp the entire $p_\text{init}$ into the shape of $p_\text{data}$ as it steps through time $t.$ Specifically, they learn **velocity fields**: given a mid-transform $p_t,$ diffusion models learn in *which directions* and by *how much* $p_t$ should change in order to get closer to $p_\text{data}.$ 

Stepping through $t=0\to1$ and taking "snapshots" of what the current mid-transform distrubution $p_t$ looks like creates a collection of $p_t$ that together trace a path - a **probability path** - which interpolates between $p_\text{init}=p_0$ and $p_\text{data}=p_1.$

During training, practitioners would *pre-define* a probability path during training, such as $x_t = (1-t)x_0 + tx_1,$ where $x_0,x_t,x_1$ are samples from $p_0,p_t,p_1$ respectively. This means that during training, practitioners don't need to actually simulate the process, which would be very costly, and can calculate $x_t$ directly.

!!!note "Differences in Notation"
    Earlier works derived diffusion from another framework (Markov Chain Monte Carlo, MCMC), and you thus may see conflicting conventions.

    For example, some refer to the transformation $p_\text{init}\to p_\text{data}$ as the **reverse process,** preferring a **time reversal** convention where instead, this process steps *backwards* through **discrete time:** $t=1000,999,998,...,0.$

    Some conventions from then still linger to this day, such as data prediction being written as $x_0$-prediction and not $x_1$-prediction.

### $p_\text{data}$ Model Challenges

Since the model should output probabilities, there are some contraints that it should follow:

1. **Outputs should be positive.** Negative probability doesn't make sense.
1. **All possible outputs should sum to 1.** This is because the output should represent chances that a specific outcome occurs; However, the chance that there be one outcome (*any* outcome, don't care what it is) should be 100%, or 1.

1\. is not hard. For example, one can simply do $e^{-\text{output}},$ where $\text{output}$ is the raw neural network output, to transform it into all strictly positive values.

2\. is the main issue. One natural idea is to first find what these outputs sum to, $Z,$ then just divide the model output value by $Z.$ Combining with the above, that results in $\frac{e^{-\text{output}}}{Z}.$ 

However, as there are infinitely many possible inputs to the model, there are infinitely many possible outputs. It's impossible to sum over infinitely many arbitrary numbers in a finite amount of time.

Here, $Z$ is called the **normalizing constant,** as in it is the number that would normalize the sum of outputs to 1 when divided.

People have come up with ways around this of course. Approaches prior to diffusion can be split into 3 categories, each with their own downsides. Referencing [this talk](https://www.youtube.com/watch?v=wMmqCMwuM2Q) / [blog](https://yang-song.net/blog/2021/score/) by Yang Song, one of the pioneers of modern diffusion, these are:

1. **Approximate $Z$:** This is *still* very hard and expensive to calculate.
2. **Restrict the Architecture:** This limits the freedom in designing the neural network. Examples include autoregressive models.
3. **Model the Generation Process Only:** Skip trying to model $p_\text{data}$ accurately, just model a process that can generate new data. This usually involves using adversarial loss, which is highly unstable and hard to train with, plus [the result might deviate from $p_\text{data}$](https://arxiv.org/abs/1706.08224). Examples include GANs.
