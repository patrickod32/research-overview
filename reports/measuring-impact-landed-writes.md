# The AI Safety Puzzle Everyone Avoids: How To Measure Impact, Not Intent.

By [Patrick0d](https://www.lesswrong.com/users/patrick0d) · 2025-07-22

Originally published on [LessWrong](https://www.lesswrong.com/posts/3KTgeXBfhvRKfL5kf/the-ai-safety-puzzle-everyone-avoids-how-to-measure-impact).

\- Using Landed Writes
----------------------

We do not know the algorithm that transformers follow to compute an answer. This was the original goal with artificial intelligence - we can't easily improve the human brain, so we will build a computer that can think like a human. Then it can learn instantly, have a perfect memory and follow fair, ethical logic. 

This is not how AI has panned out in the past few years. **Trillions** of weights, **billions** of dollars spent on energy and data centres, **millions** of GPUs, **thousands** of researchers, **hundreds** of models and a **dozen** CEOs calling the shots. But when a LLM is given the prompt '2+2=' it fires hundreds of millions of activations and hopefully says '4'.

We have made progress. We have found circuits, features for semantic concepts, in-model behaviours and more. But almost 3 years after the ChatGPT moment we still can't say exactly how they work! At the most literal level: *linearly added writes by heads and neurons are added to a residual stream and normalised via scaling, several times, and then compared to the values of known words to get a confidence value for the model' prediction of the next word*. I am proposing a new approach to help us understand a little bit more about how models arrive at an answer. It is not semantic, and is causally naive, but attempts to track model action in the models own frame of reference. And this method says in a literal sense, "Which heads and neurons pointed us to this specific word".

### **How does a LLM pick its next word?** 

When a neuron or attention head helps generate a new token by writing a value to a residual stream. This can be viewed in many models as a single 2048d or 4096d coordinate in hidden space. The model stores known words in embedding space and at the end of a forward pass it checks what known word in embedding space the residual is 'closest' to. The residuals location is based on where the neurons and heads (Llama 8b has about 130k of these units) moved the residual to via their concatenated and scaled writes. We can feed the model a sentence and get a probability of each potential answer based on the residual's position. "*Paris is the capital of*" may be the prompt, and the model might have different confidence guesses of "*France*", "*London*" or maybe some other grammatical guesses of "*:*" or "*a*". These are all words the model knows and the model scores every single known word based on its similarity to this residual that has been built, the single 4096d coordinate.

**But here's the critical part: these writes don't directly influence the model's output.** After collection, RMSNorm scales these values dramatically - and the scaling varies by layer depth.

Based on actual measurements from Mistral 7B: if a typical small write of +0.001 happens in attention layer 2, it gets amplified by an average factor of 176×, becoming +0.176. But if that same +0.001 write happens in layer 31, it only gets scaled by 5.8×, becoming +0.0058. **That's a 30× difference in impact for the same initial write, depending solely on which layer it comes from.**

![](https://39669.cdn.cke-cs.com/rQvD3VnunXZu34m86e5f/images/2ed80d7f30fc05fa2434dfac6fcfb9400583da37d69f6067.png)

This isn't a quirk - it's systematic across models:

*   **Early layers (0-3)**: Massive amplification (Mistral: up to 176×, Llama models: 10-16×)
*   **Middle layers**: Moderate scaling (2-20×)
*   **Late layers**: Compression or near-unity (0.55-6.8×)

Experiments show that 98.8% of all writes get amplified by more than 2×, with 81.2% amplified by more than 10×. Most interpretability tools completely miss this. They attribute model behaviour to the pre-norm writes, not the post-norm values that actually influence the output. It's like measuring what someone **intended** to say rather than what was **actually** heard.

We are instead trying to simply say: what units actually were the contributors to the residual coordinates that influenced the final token's selection - accounting for how their contributions were scaled. Then afterwards we know where to look. A set of heads and neurons literally wrote the answer! After we know which ones we can better reason about why they wrote it.

There's a problem with simply tracking these writes though. After a group of neurons or heads write to the residual stream these writes are collected and *normalised*. And the normalisation depends on what writes are present at that time. All values per coordinate are scaled by the RMS. Kind of like "**how far have all of these writes moved the 4096 coordinate from the origin, and let's make sure none of them are too small or too big in relation to the rest**".

Each coordinate is scaled by a factor determined by:

*   The shared root-mean-square (RMS) of the residual at that token.
*   A learned, fixed gain parameter γ unique to each coordinate.

Yet, most interpretability tools work around this crucial step, attributing model behaviour to unscaled writes. There's nothing wrong with current tools as the new approach I am suggesting is causally naive. It is a literal tracking of who wrote *what* and so doesn't reveal the *why*. I think we should combine this approach with others since we need all the help we can get to figure out how these models truly work.

### Introducing Landed Writes

A **landed write** is simply the value a model component, attention head or neuron, contributes to the residual - scaled exactly as the model itself sees it:

    landed_i = (write_i / σ) · γ_i

This scaling is exactly what the model uses internally. Tracking landed writes means attributing logits to what the model actually computed, not just what it intended to write. Since we know each write will be scaled by the norm, if we measure a contribution before scaling it reveal the unit's intent to influence the residual. If we measure the scaled value instead we can see what a unit definitionally contributed to the residual. The attribution is linear up until the norm. It is trivial (in words but needs to be done carefully in code) to say what units wrote what. But instead of attributing what a unit **tried** to write, we can attribute it based on the write it **landed**. Using a realistic example: two neurons in layer 2 combine to write +0.0006 and +0.0004 to leave a coordinate's magnitude at +0.0010. So all I am proposing is that we look at how this single coordinate *was used by the component to produce an output* – in layer 2 the normalized read results in an effective scale of 176×, so the output of the component is +0.1760. Then we attribute +0.1060 and +0.0700 to the respective units, based on their share of the pre-norm total.

    Toy Example:
    BEFORE NORMALIZATION: 
    ==================== 
    Layer 17, Coordinate 2847 
    Neuron A writes: →→→→→ +0.5 
    Neuron B writes: →→→ +0.3 
    Neuron C writes: →→ +0.2 
    Neuron D writes: +0.0 
    ------- 
    Pre-norm total: +1.0 
    RMSNORM SCALING: 
    =============== 
    RMS of full residual vector = 0.05 
    1/RMS = 20.0 γ₂₈₄₇ (learned weight) = 0.2 
    Scaling factor = (1/RMS) × γ = 20.0 × 0.2 = 4.0 
    COMPONENT OUTPUTS BASED ON NORMALIZED READ (Landed Writes): 
    ==================================== 
    Neuron A landed: →→→→→→→→→→→→→→→→→→→→ +2.0 (0.5 × 4.0) 
    Neuron B landed: →→→→→→→→→→→→ +1.2 (0.3 × 4.0) 
    Neuron C landed: →→→→→→→→ +0.8 (0.2 × 4.0) 
    Neuron D landed: +0.0 (0.0 × 4.0) 
    ------- 
    Post-norm total: +4.0 
    Traditional attribution: "Neuron A contributed 0.5" 
    Landed write attribution: ‘Neuron A produced 2.0 as its output to the residual’ ← This is what affects the model’s next computations!

### Key Findings

My experiments with models like LLaMA-3.1-8B and Mistral-7B revealed some surprising results:

*   **Extreme Sparsity**: Logit predictions often rely heavily on just 11-90 coordinates out of thousands, indicating very sparse and specialized internal representations.
*   **Significant Scaling Effects**: Early-layer coordinates typically get amplified dramatically, while late-layer coordinates are usually compressed, significantly reshaping predictions.
*   **Stability Across Prompts**: Each coordinate’s scaling factor stays consistent across different prompts, making landed writes more predictable - this is surprising but does not interfere with landed writes.

### Why It Matters

Systematic scaling effects have a large impact on the models output. Many units writing to a single coordinate? Many units doing unusually large or small writes? Few writes to usually busy coordinates? All of these can drastically change the value a head or neuron contributes to the residual stream compared to what that value is turned into after normalisation.

### Practical Benefits

*   **Accuracy**: Exactly tracks how the model internally reshapes contributions.
*   **CHEAP!**: One memory intensive forward pass or two light ones - one to find the critical coords and one to record scaled writes to them. No gradient tracking or SAE training needed.
*   **Interpretability**: Clearly reveals critical coordinates behind predictions.
*   **Simplicity**: Easy to implement with minimal overhead using forward hooks.

### Drawbacks

*   **Model APIs**: Some model apis will make it difficult to fairly track each linear operation as it can bundle operations for efficiency. But all landed writes can be captured with correct tooling.
*   **Causally Naive**: It does not say *why* a unit did a write. Or reveal more complicated operations like competing writes in a layer. These likely happen and we don't measure their impact as they have a causal impact. It just says which writes chose each logit and which units literally landed those writes.
*   **Only a first order analysis**: Alone it may not reveal too many interesting features such as complex coalitions or internal computations. But combined with a SAE or other modern approaches it could shed new light on where to look and reveal some new model mechanics.

### Next steps

This is simple in theory but takes careful software engineering in practice. If we can make landed writes a first class interpretability citizen we can look for more patterns and behaviours with a single forward pass. First order interpretability one token at a time on a CPU is in sight! In my experience this has been trivial for MLP-to-landed-write flows, but extra hooks are needed for heads when using the HuggingFace Transformers library. Maybe a simple `landed_writes="true"` flag is something we can contribute to some OSS libraries. **What about training an SAE on landed writes only**? Will this miss causal effects and lower performance? Or potentially only keep useful computation? This might be a new lens on distillation. In the same way a distilled model can shed some low value activations that don't impact logits, maybe a model trained on landed writes will only contain the most important parts of the model.

Can we track landed writes as a way to see how much computation actually happens in the models reference frame per layer? If we track entropy, Fisher information, informational capacity or geometric relationships we might reveal landed write systemic behaviour.

By zeroing pre norm values and tracking post norm landed writes can we detect any behaviour that changes the landed write landscape (their local causal effect) - via sigma gaming such as writing to coordinates only inflate or suppress sigma to change the way other writes land? Can we compare landed writes with pre norm writes to see what writes contribute causally and what writes are first order pushes towards a specific logit? Does the crowd effect alone determine what lands or are behaviours more competitive and strategic? We might be only a few steps away from finding the primitive ops of a transformer as we can now focus on the *atomic units of computation*, landed writes, and we can untangle what gives rise to them.

Do multiple features or circuits land the same writes? Landed writes open up a mechanically faithful low cost measure of logit selection and a new way of probing model behaviour. We have a new quantity to measure in the noise - *landed writes*, and we can now take one step closer to building a full causal map of an LLM.

### Try It Yourself

Implementing landed writes is straightforward. Just:

1.  Capture pre-normalization writes from components.
2.  Track RMSNorm’s σ and γ values at each layer.
3.  Attribute logits to these scaled landed writes.

If more people find them useful we can add them as a first order pass for model attribution in popular libraries such as HF Transformers and even pytorch. 

  
Experimental code (with results in the notebooks) and full paper at [https://github.com/patrickod32/landed_writes/](https://github.com/patrickod32/landed_writes/)

If someone is willing to endorse me I can add the paper to arxiv. I am also looking for a **lab to join as a researcher** so I can continue to make progress on solving interpretability. No, no, please. Form an orderly line. One at a time please. The experiments and end to end investigations I conducted to write this paper costed about $10 of gpu time so you know I'm pragmatic at least.

Implementation note:

If in models like llama 3.1 8b when we normalise a residual, that value is temporary and only viewed in its normalised state by the component, one row of heads or one mlp, and then a update is computed based on the normalised value. Then this update is added to the original unnormalised residual. This means the landed write changes per layer and so we would need to attribute based on read time value - perhaps using a sliding sensitivity window over coordinates to gauge what coordinates of the residual are read to give rise to another write. This is tricky! But the core points of scaled writes and logit sparsity still stands. It is not a panacea but we can see that other methods don't capture this effect in their computations either.

Pseudo code diagram:

    HOOKS NEEDED:
    ============
    
    def setup_hooks(model):
        
        for layer_idx in range(model.n_layers):
            
            # 1. ATTENTION OUTPUT HOOK
            # HOOK PER HEAD AND USE EACH HEADS LINEAR PROJECTION TO SEE WHAT COORDS
            # IT WRITES TO
            hook(model.layers[layer_idx].attn.o_proj, 
                lambda: capture_write("attn", layer_idx))
                
            # 2. MLP DOWN PROJECTION HOOK  
            hook(model.layers[layer_idx].mlp.down_proj,
                lambda: capture_write("mlp", layer_idx))
                
            # 3. PRE-NORMALIZATION HOOK
            hook(model.layers[layer_idx].norm1.input,
                lambda: capture_pre_norm_residual(layer_idx))
                
            # 4. POST-NORMALIZATION HOOK
            hook(model.layers[layer_idx].norm1.output,
                lambda: capture_post_norm_residual(layer_idx))
                
            # 5. CAPTURE NORM PARAMETERS
            hook(model.layers[layer_idx].norm1.forward,
                lambda: capture_sigma_gamma(layer_idx))
    
    TRACKING LOGIC:
    ==============
    
    writes_tracker = {
        "layer_X": {
            "attn": tensor([...]),     # Pre-norm attention writes
            "mlp": tensor([...]),      # Pre-norm MLP writes  
            "sigma": float,            # RMS normalization factor
            "gamma": tensor([...]),    # Learned weights per coordinate
            "pre_norm": tensor([...]), # Total pre-norm residual
            "post_norm": tensor([...]) # Total post-norm residual
        }
    }
    
    LANDED WRITE CALCULATION:
    ========================
    
    for each layer:
        for each component (attn/mlp):
            scaling_factor = (1 / sigma) * gamma
    		Note: this is the output of the component based on the normalized residual
    		it reads. The residual stream itself is updated by this unscaled output 							the scaling happens during read-time normalization, not when the write is applied.
            
            # Attribution:
            # Component wrote: pre_norm_write
            # Component landed: landed_write ← THIS IS WHAT MATTERS!
    
    EXAMPLE TRACE:
    =============
    Layer 2, Coord 1483:
      Attn writes:    +0.0006  →  scales by 176×  →  lands: +0.106
      MLP writes:     +0.0004  →  scales by 176×  →  lands: +0.070
      Skip conn:      +0.0001  →  scales by 176×  →  lands: +0.018
      --------------------------------------------------------
      Total landed:   +0.194 (vs pre-norm total of +0.0011)
