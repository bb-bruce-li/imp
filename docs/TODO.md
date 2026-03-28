# imp — TODO

## Validation Experiments

### Experiment 1: Manual Contract Injection Test

**Goal:** Validate that a cognitive contract actually improves AI interaction quality before building any automation.

**Steps:**
1. Pick 2-3 engineers who struggle with AI (ideally different skill levels)
2. Sit with each one, manually assess their level — what they know, what confuses them, what they don't know they don't know
3. Write a cognitive contract by hand for each person (abstraction level, known concepts, ZPD edge, avoid concepts, pacing style)
4. Inject that contract into their Claude/Cursor system prompt
5. Have them use AI for a week with the contract
6. Ask: did the output quality noticeably improve? Did they feel more in control? Did they learn anything they wouldn't have otherwise?

**Success criteria:**
- Users notice a difference without being told what changed
- Users report feeling more in control of AI interactions
- At least one user demonstrates learning (uses a concept independently that was previously in their ZPD edge)

**If it works:** The product is validated — everything else is automating contract generation.
**If it doesn't:** The problem may not be cognitive mismatch. Reconsider the approach before building infrastructure.
