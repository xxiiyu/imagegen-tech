# Exploring Internals

ComfyUI is great, except when it's not. Documentation is especially lacking: If what you're looking for isn't some generic issue that has been answered 10 times before, you're most likely out of luck.

This section hopes to alleviate some of that, by actually digging into the internals and explaining what the nodes do. Through better understanding comes better informed decisions, like when/why should X node be used in this situation.

## Recommended Usage

Though it is not done, I assume this section will quickly become messy and unorganized. I will try my best to make each item easily searchable.

If you're looking for a specific node, I thus recommend not going through each page, but simply use the search function located at the top right of the website.

## Methodology

I make extensive use of:

- **`Power Puter (rgthree)`:** Allows you to run basically arbitrary python. I set the output type to `*` most of the time for digging purposes.
- **`Display Any (rgthree)`:** Allows you to display basically anything.
- **`🔧 Debug Tensor Shape (comfyui-essentials)`:** A convenience node for showing tensor shapes, though the functionality can technically be achieved with `Power Puter` too.

If you want to dig into ComfyUI nodes yourself I highly recommend giving the above nodes a try. Obviously, this also requires you to know python.

## Ramble

You can ignore this section, I just want to vent.

There's nothing of substance here.

...

You're still here? Oh well. Enjoy seeing me crash out I guess.

But seriously. How is it so bad.

"Hmm. I wonder what `ModelSamplingAuraFlow` does.  
...[ComfyUI Docs](https://docs.comfy.org/built-in-nodes/overview)... Nope, no page on that... The other [wiki](https://comfyui-wiki.com/en)? Nothing either.  
Let's check the internet."

- **[`RunComfy`](https://www.runcomfy.com/comfyui-nodes/ComfyUI/ModelSamplingSD3):**  
> The `ModelSamplingAuraFlow` node is designed to enhance the sampling process of AI models, specifically tailored for the AuraFlow model. This node allows you to apply advanced sampling techniques to your model, which can significantly improve the quality and efficiency of the generated outputs. By leveraging the `patch_aura` method, it adjusts the sampling parameters to optimize the model's performance, ensuring smoother and more accurate results. This node is particularly beneficial for AI artists looking to fine-tune their models with precise control over the sampling process, ultimately leading to more refined and high-quality artistic outputs.

"Oh, uhh... cooolll... doesn't help me understand a thing though. What about here?"

- **[`ComfyAI.run`](https://comfyai.run/documentation/ModelSamplingAuraFlow):**  
> The ModelSamplingAuraFlow ComfyUI node is a versatile tool designed for advanced users aiming to fine-tune models through dynamic sampling. It allows users to apply a shift parameter to models, catering to various AI model development and experimentation needs. This node can be accessed and run through ComfyUI, whether on a local setup or a cloud-based service, providing flexibility and scalability for different deployment scenarios.

"Well... let's check somewhere else?"

- **[`InstaSD`](https://www.instasd.com/comfyui/custom-nodes/comfyui/modelsamplingsd3):**  
> The `ModelSamplingAuraFlow` node in ComfyUI is designed to perform specific sampling functions related to the AuraFlow model within the broader framework of ComfyUI's modular visual AI engine. This node enables users to integrate AuraFlow's sampling techniques into their custom workflows, benefiting from its unique approach to model sampling.

Like, I just can't.  

Why do both official wikis only talk about the nodes that everyone already knows what they do?  

Are the unofficial websites serious with these "explanations" they give out? Can they write nothing but flowery yet useless sentences? Do they collectively share 1 singular LLM braincell?  

My god.
