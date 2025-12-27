# Data Types

Before looking at individual nodes, it's helpful to know how the data they work with looks like, and how they behave.

Also refer to Comfy docs:

- [Datatypes](https://docs.comfy.org/custom-nodes/backend/datatypes)
- [Images, Latents, and Masks](https://docs.comfy.org/custom-nodes/backend/images_and_masks)

## Basic Types

|                  | `int`                      | `float` / `double`   | `string`                    | `bool`        |
| ---------------- | -------------------------- | -------------------- | --------------------------- | ------------- |
| Semantic Meaning | Whole numbers like `12345` | Decimals like `3.14` | Text like `"I like trains"` | True or False | A collection of items |

The most complex out of these is `float` - besides being a specific type, it also encompasses the idea of the "floating point format/number", which is a smart scheme to store a very wide range of numbers with a fixed bit budget.

Usually, `float` actually means a 32-bit floating point number, while `double` means a 64-bit floating point number. You may also see some people / programming languages use other naming, e.g. `float16` `float32` `float64` where the number is how many bits.

!!!info "Floating Point Numbers"
    A number stored in the floating point format is split into 2 parts, the "mantissa" and the "exponent," in base 2 because computers.  
    For demonstration, I'll show what that may look like using our every day normal numbers (in other words, base 10):  
    <big> $12345=\underbrace{1.2345}_{\text{mantissa}}\times\underbrace{10}_{\text{base}}\!\!\!\!\!\!^{\overbrace{4}^\text{exponent}}$ </big>  
    Assume we have a fixed 32-bit budget. To represent massive / tiny numbers, you can allocate more of those 32 bits to the exponent, giving it greater range at the cost of accuracy (usually a worthwhile trade).

**JavaScript,** the programming language powering ComfyUI (and most of the internet's) interactivity, uses `double` to store all numbers.  This leads to potentially surprising behavior. For example, a `double` starts to lose the ability to distinguish between $n$ and $n+1$ at $n=2^{53}=9007199254740992.$  
To see this in action, take a `Int (utils > primitive)` node and try inputting `9007199254740993`. Alternatively, input `9007199254740992` and try to increase it by pressing the arrow.

## Tensors

**Tensors** probably won't be the direct input or output of nodes, but understanding them will nevertheless prove immensely helpful imo.

|                 | 0D Tensor     | 1D Tensor                          | 2D Tensor             | 3D Tensor          | 4D Tensor         |
| --------------- | ------------- | ---------------------------------- | --------------------- | ------------------ | ----------------- |
| Also known as   | scaler        | vector                             | matrix                |
| Explanation     | 1 number      | a list of numbers                  | a grid of numbers     | a cube of numbers  |
| ComfyUI example | `seed`, `cfg` | `sigmas` (represents noise levels) | pooled `conditioning` | raw `conditioning` | `image`, `latent` |

In other words, a tensor is simply a multi-dimensional array of numbers (including 0D, a single number).

<!--
### Broadcasting

For many operations on 2 tensors, they need to be **[broadcastable.](https://numpy.org/doc/stable/user/basics.broadcasting.html#general-broadcasting-rules)** Here's an example of how to check that:

Assume you have two tensors `M` and `N` with the shape `3 x 4 x 5 x 6` and `2 x 5 x 1`:

- Align them from the tail:
```
3 x 4 x 5 x 6
    2 x 5 x 1
```
- Fill the missing dimensions with 1
```
3 x 4 x 5 x 6
1 x 2 x 5 x 1
```
- Now check each dimension pair - all pairs must be **equal** or one of them must be **1** for two tensors to be broadcastable:
```
        v equal
            v one of them is 1
3 x 4 x 5 x 6
1 x 2 x 5 x 1
    ^ not equal nor 1!!!
^ one of them is 1
```
for this example, the 2nd dimension pair (4, 2) aren't equal, nor is one of them 1, so `M` and `N` aren't broadcastable and many operations will fail on them.
-->

## Images, Latents and Masks

**`image`** and **`latent`** are both 4D tensors, while **`mask`** is a 3D tensor. The semantic meanings of the dimensions are:

- `N (batch size)`: How many images/latents there are.
- `C (channels)`: How many channels. For example, an image may have 3 channels, RGB.
- `H (height)`
- `W (width)`


`latent`s are in the "channel first" format - `NCHW`, meaning the first dimension is `N`, the second is `C`, the third is `H`, and the last is `W`.

`image`s are in "channel last" - `NHWC`.

`mask` doesn't have the channel dimension - `NHW`.

### "Frequency"

For something more familiar like audio, the frequency is the **number of vibrations** with respect to **time**. Higher frequency = more vibrations = sounds like higher pitch.

For `image`s (and by extension, `latent`s), it's the **change in pixel values** with respect to **distance (X Y coordinates)**. 

- High frequency = rapid change in color/brightness/etc in a small area; Corresponds to **fine details**.
- Low frequency = gradual change in color/brightness/etc in a large area; Corresponds to image **structure**.

## Common Errors

**KSampler - Input type (double) and bias type (float) should be the same:**

- Try switching your Preview method from `TAESD` to something else. 
