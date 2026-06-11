# n8n-problem-12-ai-output-reviewer-approval-gate
n8n automation workflow that reviews AI-generated content using a two-phase evaluation engine — deterministic rule checks followed by an LLM quality judge (Groq Llama 3.1) — calculates a confidence score, auto-sends high-confidence outputs, routes borderline drafts to a Slack human approval queue, rejects low-quality 
