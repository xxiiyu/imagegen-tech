# What is Guidance?

In early diffusion models, generation was basically rolling dice (seeds). Assume you train a model on a dataset of elves and bunnies. While you can ask it to generate an image, you can't tell it to specifically generate an elf or a bunny. The only thing you can do is keep changing seeds, until one just so happen to appear.

**Guidance** is the mechanism that solves this. It is the umbrella term for the techniques used to **steer the model's generating process toward a specific target.**

Guidance can take many forms, from text prompts, to other input images, a hypernetwork such as ControlNet, and more.

<div class="grid cards" markdown>

-	[Classifier-Free Guidance](./CFG/00_cfg.md)

</div>
