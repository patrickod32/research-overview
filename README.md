# Patrick — Research

This repo highlights my independent AI Safety research. Professionally I am an experienced fullstack developer currently leading a AI team at the architect level in the finance domain. I have a background in quant development, fullstack development, app dev, project management and am a former physicist. My physics focus was nanotech and lithography.

My research interests are mechanistic interpretability, AI control and Alignment. My approach is to use findings from pre/post training recipes and repurpose them for safety use cases. An example of this is my use of a [parameter golf](https://github.com/openai/parameter-golf) architecture for AI safety. In an autoresearch loop I distilled a 20GB SAE pipeline into a 5.3MB probe. The probe learnt 2000 concepts to detect from text and the top 5 concepts it detects fired in the teacher SAE 95% of the time. I spend a lot of my research time studying technical pretraining reports as I think this is a key factor in understanding how to best design interpretability experiments.

## Articles

- **[Don’t Waste a Good Swarm Attack and the Colluder’s Dilemma](reports/swarm-attacks-and-the-colluders-dilemma.md)**  
  “This is the Colluder's Dilemma: keep your colony small and less effective at causing harm or make it larger and more detectable to the harness.”

- **[Competitive AI Safety is the loss function to make sure AI goes well](reports/competitive-ai-safety.md)** · [Code: seq2feature](https://github.com/patrickod32/seq2feature)  
  “AI safety needs a loss function and Competitive AI safety can deliver it.”

- **[The AI Safety Puzzle Everyone Avoids: How To Measure Impact, Not Intent.](reports/measuring-impact-landed-writes.md)** · [Code and paper: landed_writes](https://github.com/patrickod32/landed_writes)  
  “It is a literal tracking of who wrote what and so doesn't reveal the why.”

- **[The End of the Digital Age Mirrors its Beginning](reports/the-end-of-the-digital-age.md)**  
  “The Digital Age began with us learning how to write circuits. It will end with us learning to read them.”

*Each article includes a link to its original publication on LessWrong.*
