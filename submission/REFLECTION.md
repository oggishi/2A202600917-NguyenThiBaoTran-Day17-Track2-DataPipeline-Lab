# Reflection — Day 17 (≤ 200 words)

Answer briefly, in your own words. This is graded on reasoning, not length.

1. **The flywheel.** Day 13 emitted agent traces; today you turned them into an
   eval set and DPO pairs that Day 22 will train on. Which step in
   `traces → Bronze → datasets` would break most silently in production if you
   got it wrong — and how would you detect it?

2. **Decontamination.** Your run dropped 2 of 3 preference pairs because their
   prompts were in the eval set. What concretely goes wrong if you *skip* this
   step and train on those pairs? How would the lie show up in your metrics?

3. **Point-in-time.** The naive join leaked a future `lifetime_spend` into the
   training row. Describe one feature in a system you know that would be
   dangerous to join without an `ASOF`/point-in-time guard.

4. **Graph vs vector.** From `kg_demo.py`, name one question the knowledge graph
   answers well that flat chunk retrieval (`embed.py`) would struggle with, and
   one where the graph is overkill.

_Write your answers below._

1. The quietest break would be span flattening: if child spans or `gen_ai.*`
   fields are dropped, the pipeline still runs but cost, latency, and outcomes
   become wrong. I would detect it with trace/span count checks, null-rate checks
   on important attributes, and sampled trace reconstruction.

2. Skipping decontamination trains on the same prompts used for evaluation. The
   model can memorize those answers, so eval accuracy looks better than real
   generalization. The lie would show up as strong eval metrics but weaker
   performance on fresh prompts or paraphrased holdouts.

3. A churn model should not join a user's "number of support tickets closed"
   from after the prediction date. Without an ASOF guard, the model sees future
   dissatisfaction resolution and learns from information production would not
   have at scoring time.

4. The graph answers multi-hop questions like which warehouse supports a
   returnable product through its accessory relationship. Flat vector retrieval
   may split those facts across chunks. The graph is overkill for direct policy
   lookup, such as asking whether widgets are returnable.
