# What is CFG?
!!!note "CFG in Flux"
	**Flux was not trained with CFG, thus nothing in this section applies.** The "Guidance" value you can provide to flux is not CFG.

## The CFG Equation

It's simplest to understand how CFG works exactly by directly looking at its equation:

$$
x_\text{cfg} = x_\text{uncond} + \omega(x_\text{cond} - x_\text{uncond})
$$

- $x_\text{uncond}:$ the model prediction without conditioning
- $x_\text{cond}:$ the model prediction with conditioning
- $\omega:$ CFG scale
- $x_\text{cfg}:$ result image

For brevity, $x_\text{uncond}$ and $x_\text{cond}$ will be referred to as `uncond` and `cond` respectively. One can imagine that:

- `uncond` is what the model thinks it should do.
- `cond` is what the model thinks it should do, *given* the guidance.

Let's play with some CFG numbers:

| cfg | `cfg_result`            | effect                                                                                           |
| --- | ----------------------- | ------------------------------------------------------------------------------------------------ |
| 0.1 | `uncond*0.9 + cond*0.1` | Effect of guidance is pretty weak. 90% of the generation process is still decided by `uncond`.   |
| 0.5 | `uncond*0.5 + cond*0.5` | Guidance is on even footing with the unguided conditioning.                                      |
| 1   | `cond`                  | **`uncond` cancels out, leaving only `cond`.**                                                   |
| 2   | `2*cond - uncond`       | **The model actively moves away from `uncond`,** while the effect of guidance increased further. |

When implementing CFG in practice, people also noticed that when `cfg > 1`, the model moves away from `uncond`. Then, can `uncond` be used as some sort of opposite guidance - "Do anything but this?" Yes! This is what became negative prompts.

Now with that in mind, let's rewrite the equation in more familiar terms:
```
cfg_result = negative + (positive - negative) * cfg
```

- `cfg < 1`: Negative prompts would behave like positive prompts.
- `cfg = 1`: Negative prompts have no effect. 
- `cfg > 1`: The model will actively avoid generating anything in the negative prompt.

Note something special about `cfg = 1`. Since the `negative` term cancels out anyway, one can skip calculating it entirely. This is why you'll see **2x iteration speed if you set `cfg = 1`.**

???note "Differences in Notation"
	Some works prefer to base $x_\text{cfg}$ on $x_\text{cond}$ instead, resulting in this equivalent setup:

	$$
	x_\text{cfg}=x_\text{cond}+\omega'(x_\text{cond}-x_\text{uncond})
	$$

	You can check that they're equivalent by plugging in $\omega'=\omega-1.$

	`k-diffusion`, and by extension most popular UIs like Forge and ComfyUI, use the $\omega$ convention.

## CFG and the Data Manifold

!!!info "The Manifold Hypothesis" 
    The **Manifold Hypothesis** suggests that real-world high-dimensional data (like natural images) lie on a well-behaved subspace called a *manifold* inside that high-dimensional space.

Even for a well-trained model, while both $x_\text{uncond}$ and $x_\text{cond}$ would lie on the real data manifold, their linear combination via cfg may not. This is demonstrated in the following interactive visualization:

<div>
    <script type="text/javascript" src="https://cdn.geogebra.org/apps/deployggb.js"></script>
    <script type="text/javascript">
        function perspective(p){
            ggbApplet.setPerspective(p);
        }
        var parameters = {
            "id":"ggbApplet",
            "appName":"geometry",
            "width":650,
            "height":400,
			"showToolBar":false,
			"enableLabelDrags":false,
			"enableShiftDragZoom":false,
			"enableRightClick":false,
            "errorDialogsActive":true,
            "useBrowserForJS":false,
			"filename":"../assets/cfg.ggb",
        };
        var applet = new GGBApplet(parameters, true);
        //  when used with Math Apps Bundle, uncomment this:
        //  applet.setHTML5Codebase('GeoGebra/HTML5/5.0/web3d/');
        window.onload = function() { applet.inject('applet_container'); }
    </script>
    <div id="applet_container"></div>
</div>

In fact, when $\omega\ne0\text{ or }1,$ the result CFG leads to ceases to be a true probability distribution.

When $\omega > 1$, the equation becomes *extrapolation* instead of interpolation; at higher and higher scales, the result strays further and further away from the data manifold and what the model has seen in training, leading to artifacts such as color burn.  
Even for $0\leq\omega\leq1$ though, if the manifold just so happens to be sufficiently curved between $x_\text{cond}$ and $x_\text{uncond},$ it also leads to an off-manifold result, albeit at much less severity.

CFG is therefore not well-grounded in theory; its use is mainly justified by the massive leap in performance it offers in practice.

!!!info "Why *do* We Need CFG?"
	If using CFG at all distorts the result into a fake distribution, why do all modern models still need CFG to generate high fidelity images?  
	While the [original idea](https://arxiv.org/abs/2207.12598) came out in 2022, it's not until 2024 did people begin investigating CFG's inner workings under simplified assumptions, coming up with many intriging results.  
	A [more recent](https://arxiv.org/abs/2505.19210) work finds that current models don't capture class-specific features as well as we'd like, leading to melded blobs at low guidance scales. It argues based on this that maybe the current training regimes still have massive room for improvements.