---
name: llm-council
description: Run a question through an LLM Council. Five independent advisors answer separately, then anonymously review and rank each other's answers, then a chairman synthesizes one final answer. Use when the user types /llm-council, or asks for a council, multiple perspectives, a peer-reviewed answer, or a second opinion on a hard decision, tradeoff, or ambiguous question.
---

# LLM Council

Adapted from karpathy/llm-council. The original is a web app that sends a query to several LLMs via OpenRouter. This skill keeps its three-stage logic (first opinions, anonymized review and ranking, chairman synthesis) but runs inside Claude Code using five independent subagents. Diversity comes from five advisor roles, not from five different model vendors.

The question to put to the council: $ARGUMENTS

If that is empty, ask the user for the question and stop.

## Rules that apply to every stage

- Stage 1 and Stage 2 agents MUST be separate subagents (Task/Agent tool), launched in parallel, each with a fresh context. Never show one advisor another advisor's output during Stage 1.
- Pass each subagent only what this file says it gets. Do not add your own opinion or hints about the answer.
- If a subagent fails or returns nothing, drop it and continue with the rest (the original does the same). If no advisor responds in Stage 1, report the failure and stop.
- If subagents are unavailable, tell the user the independence guarantee is lost, then ask whether to proceed sequentially anyway.

## Stage 1: First opinions

Launch five subagents in parallel. Each receives the user's question plus exactly one role lens below. Each answers the question directly and independently.

Prompt template for each advisor:

```
You are one member of a five-person advisory council. Your lens: <ROLE NAME> - <ROLE LENS>.
Answer the question below as accurately and insightfully as you can, using your lens to
shape what you look for. Do not mention your role name or that a council exists. Do not
hedge for the sake of it; commit to a position and give your reasoning.

Question:
<the user's question>
```

The five roles:

1. **Pragmatist** - what actually works in practice; cost, effort, and what to do first.
2. **Skeptic** - what could be wrong; hidden assumptions, failure modes, missing evidence.
3. **Domain Expert** - the technically deepest and most accurate answer; correct terminology and facts.
4. **Contrarian** - the strongest credible alternative to the obvious answer; what the consensus overlooks.
5. **Long-Term Strategist** - second-order effects, reversibility, and how this looks in a year or more.

Collect the five answers as `stage1_results`: a list of `{role, response}`.

## Stage 2: Anonymized review and ranking

1. Assign labels `Response A` through `Response E` to the Stage 1 answers in order. Keep a private mapping `label_to_role`. Never reveal roles or the mapping to reviewers.
2. Launch five fresh reviewer subagents in parallel (one per advisor, matching the original where every council member reviews). Each receives the same prompt:

```
You are evaluating different responses to the following question.

Question: <the user's question>

Here are the responses from different advisors (anonymized):

Response A:
<text>

Response B:
<text>

...

Your task:
1. First, evaluate each response individually. For each, explain what it does well
   and what it does poorly.
2. Then, at the very end of your reply, provide a final ranking.

IMPORTANT: Your final ranking MUST be formatted EXACTLY as follows:
- Start with the line "FINAL RANKING:" (all caps, with colon)
- Then list the responses from best to worst as a numbered list
- Each line: number, period, space, then ONLY the label (e.g., "1. Response C")
- Do not add any text after the ranking section

Judge on accuracy and insight.
```

3. Parse each reviewer's output: find the `FINAL RANKING:` section and extract labels in order from lines like `1. Response C`. If that format is missing, fall back to the order in which `Response X` labels appear after "FINAL RANKING". If nothing parses, discard that reviewer's ranking.
4. Compute aggregate rankings: for each label, average its position across all parsed rankings. Map labels back to roles. Sort ascending (lower average is better). Record the number of rankings counted for each.

## Stage 3: Chairman synthesis

You (the main agent) act as Chairman. Do not delegate this stage.

Build your context from:

- The original question.
- All Stage 1 answers, each labeled with its role (the chairman sees identities, as in the original).
- All Stage 2 reviews and rankings.
- The aggregate ranking table.

Then write one comprehensive final answer that reflects the council's collective wisdom. Explicitly weigh where advisors agreed, where they disagreed, and what the peer rankings suggest. Do not simply pick the top-ranked answer; if a lower-ranked answer contains a point the reviewers undervalued, say so and include it.

## Output format to the user

1. **Final answer** (the chairman synthesis), first and prominent.
2. **Council ranking**: a short table of role, average rank, and rankings counted.
3. **Where the council disagreed**: two to four lines.
4. Offer to show any advisor's full Stage 1 answer or any reviewer's critique on request. Do not dump all of them by default.

## Notes

- No API keys or OpenRouter account are needed. This skill uses only the model Claude Code is already running on.
- All five advisors share the same underlying model, so their independence is role-based and context-based, not model-based. Say so if the user asks why the answers may correlate.
- The original also generates a short conversation title with a cheap model. That is a UI feature of the web app and is intentionally omitted.
