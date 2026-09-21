# Onboarding Review Methodology

A focused framework for reviewing a CLI tool or agent's first-10-minutes onboarding experience. Used when the primary question is "where would a new user get confused, stuck, or give up?"

## Six Focus Areas

1. **Onboarding flow gaps** — What does a new user see first? Is there a clear path from install → first interaction? Are entry points documented and discoverable?

2. **Jargon or confusing terminology** — Are provider names, config keys, and command names understandable to someone who hasn't read the docs? Do tool names match user mental models?

3. **Missing error recovery** — What happens when the user makes a mistake? Does Ctrl+C during a wizard exit cleanly? Are there fallback paths for non-interactive contexts? Does the tool tell the user how to recover?

4. **Empty states and cold-start problems** — What does the UI show when there's no data yet? Is there guidance for the blank slate, or does the user face a void with no hints?

5. **Documentation that doesn't match the actual experience** — Does the README describe what `hermes setup` actually does? Are flags like `--portal` and `--quick` discoverable from the tool itself or only from external docs?

6. **Too many steps for basic tasks** — How many prompts/commands does it take to go from fresh install to first successful interaction? Are there shortcuts that aren't documented?

## Output Format

- Max 15 findings, severity-ranked (CRITICAL/HIGH/MEDIUM/LOW)
- Every finding cites file:line references
- Include evidence (what the user would actually see/experience)
- End with top-3 highest-impact-per-line fixes

## Key Heuristics

- Run the tool with a fresh HOME (`HOME=/tmp/fhX`) to simulate first-time user
- Check what happens when no provider/API key is configured — does the tool crash, hang, or guide the user?
- Check if `Ctrl+C` during interactive setup exits cleanly or leaves partial state
- Check if `--help` on key commands surfaces all important flags
- Check if the README mentions flags/commands that the tool itself doesn't document in its help text
- Check empty/cold-start states in the TUI — what does a new user see with no history?
