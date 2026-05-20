# AIMO-3 Silver Medal Solution 

**Competition:** [AI Mathematical Olympiad - Progress Prize 3](https://www.kaggle.com/competitions/ai-mathematical-olympiad-progress-prize-3)  
**Result:** Silver Medal (42.5/50) 
**Model:** GPT-OSS 120B (vLLM, served locally on Kaggle)

---

## context

This notebook is my solution to AIMO-3, built on top of the public baseline
[`gpt-oss-120b-with-tools`](https://www.kaggle.com/code/andreasbis/aimo-3-gpt-oss-120b-with-tools/notebook) by Andreas Bisiadis. I went mostly by intuition and a
few ideas I wanted to test. This one worked out to a silver, so I figured it was worth sharing as-is.

---

## improvements

The baseline already had a solid foundation: run 8 parallel generation attempts
with a tool-use sandbox, take the plurality answer, and return it early if 4+
attempts agree (`early_stop = 4`).

My main additions were **better prompts** and a **3-phase answer selection pipeline**.

### richer prompts

The baseline prompts were  minimal (a couple of sentences each).
I expanded them significantly:

- **System prompt** — gave the model an explicit 5-step problem-solving framework
  (Understand → Explore → Plan → Execute → Verify), mathematical reasoning principles
  (work backwards, check edge cases, look for symmetry), and verification requirements.
- **Tool prompt** — described *when* and *why* to reach for code, rather than just
  saying "you have Python".
- **Preference prompt** — documented the available libraries (`sympy`, `numpy`, `math`)
  with guidance on when to use each (exact symbolic vs numerical verification).

From experience, model tends to respond better when it has a structured mental
framework to follow, rather than just being told to "solve the problem".

### metamorphic Verification

When the 8 generation attempts produce **weak consensus** (the top answer has
fewer than `metamorphic_threshold = 3` votes), instead of just returning the
plurality, I run a verification pass:

> *"The solver thinks the answer is X. Create a simpler version of this problem
> where the answer can be brute-forced. Apply the same method to the simpler case.
> Do they match?"*

The idea (metamorphic testing) is borrowed from my personal approach to problem solving: if you can't
directly verify an output, check that a known transformation of the input produces
a consistent output. Here the "transformation" is: reduce the problem parameters
until brute force is feasible.

The model writes Python that prints:
```
METHOD_RESULT=<value>
BRUTE_FORCE=<value>
VERDICT=CONSISTENT / INCONSISTENT
ORIGINAL_ANSWER=<value>
```

If the method is internally consistent (CONSISTENT), it returns the original answer
with more confidence. If it's INCONSISTENT, the metamorphic answer is flagged as a
candidate — which triggers Phase 3.

### tutor arbitration

When metamorphic testing **disagrees** with the generation consensus, I run a
third call with a "tutor" prompt that gets both sides of the story:

> *"Group A got answer X (N/M votes). Student B's metamorphic test says Y.
> Write Python to independently verify which is right. Pay attention to
> off-by-one errors, boundary conditions, count vs value confusion."*

The tutor's answer is the final output. If the tutor also fails, it falls back
to the generation consensus.

### minor tweaks

- Slightly increased timeouts (`high_problem_timeout` 840→900, `base_problem_timeout`
  240→300, `session_timeout` 900→960) to give the new phases room to run.

---

## honest caveats

- I didn't run proper ablations. I don't know exactly how much each change
  contributed the prompt improvements and the 3-phase pipeline were developed
  together. Reckon most was initial prompts.
- The metamorphic and tutor phases only fire on *weak* consensus cases, so they
  don't affect the majority of problems (where the model agrees strongly). These tended to run a lot of computational space, and did not help tweak much more.


## acknowledgements

Big thanks to Simon and Andreas!
The core infrastructure (vLLM server, Jupyter sandbox, parallel generation, token
streaming) is theirs — I only modified the prompts and the answer selection logic.
