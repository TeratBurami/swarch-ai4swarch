**Language:** [ภาษาไทย](05-reflection.md) | English

# Individual reflection (≤ 150 words)

**Name / Student ID:** Terat Burami / 6970143621
**My role on the review board:** Architect (online role-play)

After Part A and Part B: which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

I'd trust AI with the reading-heavy work: recovering dependencies from manifests, listing options, and checking an ADR against the requirements. In Task 1 every dependency I checked was correct, and its review quickly found the event-loss and personal-data risks.

I'd keep the final decision and anything that depends on context outside the files for humans. The best example from our board: the product owner said marketing had already promised customers they'd see their points the moment they order. My ADR, which the AI had approved with changes, awarded points "within 1 minute", so it would have broken a public promise. The AI couldn't know this, because it was only on the role card. Same for the SRE's 3-week deadline and lack of production RabbitMQ experience. AI is a good analyst, but someone still has to ask the people who know what isn't written down.
