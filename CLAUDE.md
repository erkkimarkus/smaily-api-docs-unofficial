# CLAUDE.md


## Linear discipline (Smaily process bridge)

This repository is engineering truth — STATUS/DECISIONS/docs stay canonical here. Linear is the coordination and visibility layer. Full process: Outline → Processes → "Agent-Driven Development"; compact guide: Linear document "Linear workflow guide for AI agents" (attached to MGMT-5). Three rules for every agent working here:

1. **Anchor before work.** A Linear project must exist before substantive work starts — create it or link to it (minutes, not hours). This repo has no standing Linear project — anchor documentation work to the Linear project of the plugin or engine it concerns (see the *Smaily E-Commerce native integrations* and *Campaign Intelligence* initiatives).

2. **One-way doors interrupt.** Before any irreversible or expensive-to-undo commitment — persistent data schema, public/integration API contracts (e.g. `RECENGINE_API_CONTRACT.md`), releases reaching real users or stores, anything touching deliverability/reputation, pricing, or legal/consent — halt, file a Linear issue in the project with the evidence and proposed action, and wait for Erkki's approval. Reversible work proceeds at full speed without asking.

3. **Scribe pass at session end.** Before finishing a working session, distill it into Linear: post an honest project status update (onTrack/atRisk/offTrack), promote new backlog items to Linear issues, update the project's SDD document if architecture moved, close completed issues. Use the `/linear-project` skill if available, otherwise the Linear MCP tools directly.

Never duplicate repo documents into Linear — summarize and link. Linear content is written in English.
