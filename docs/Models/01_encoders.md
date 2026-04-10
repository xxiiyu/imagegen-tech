# Text Encoders

!!!warning "UNDER CONSTRUCTION"

**Text Encoders** take strings from short tags like `cat`, `dog` to entire sentences or paragraphs like ``

## Contrastive Language-Image Pre-training (CLIP)

Contrary to popular usage, "[CLIP](https://arxiv.org/abs/2103.00020)" is not a model but a training procedure to make models embed both text and images into the same embedding space. Though when used to refer to one model, it's probably `OpenCLIP` by `openai`.

CLIP-trained models like `OpenCLIP` have deceptively good concept separation. This makes it possible to "up/down weigh" specific parts of the prompt, e.g. `a girl holding an (apple:2.0)`, and have diffusion models using CLIP as encoder to focus more on the apple. 

CLIP training results in models which behave like [Bag-of-Words (BoW)](https://en.wikipedia.org/wiki/Bag-of-words_model) - that is to say, a model which doesn't distinguish between word order. Details can be found [here](https://arxiv.org/pdf/2210.01936), which additionally shows it also may cause poor attribution among other deficiencies, along with how one may alleviate this during training.

## LLM-As-Encoders

Modern diffusion models have largely shifted to using off-the-shelf LLMs such as `Gemma`, `Qwen`, etc. as encoders. The idea is LLMs are trained a massive corpus which should give them deep understanding of language semantics.

This is the main reason why modern models can be prompted with complex natural language. However, not that this still isn't perfect, and "free-styling" whatever may not always work. An example is on certain models, `Knife made of water` may not work yet `Water shaped like a knife` does.

Additionally, word embeddings in LLMs are highly entangled, so `(prompt) [weighing]` has little to no intended effects.
