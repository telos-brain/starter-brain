---
name: Chat
code: WF-CHAT
description: General-purpose conversational assistant with web access, skill lookup, memory search and focused Q&A.
version: 2
model: anthropic/claude-sonnet-4-6

# RUNNABLE: this workflow is executed manually / interactively as a chat.
type: RUNNABLE

output-tokens: 2048, 4096, 16384
caching: automatic
max-turns: 50
thinking: adaptive

# Reuse the shared persona/tone/constraints as the system prompt.
system-prompt-code: WF-SYSTEM-PROMPT

# Injected tools available every turn.
tools:
  - web_search
  - web_fetch
  - find_available_skills
  - get_skill
  - search_blueprint_entries
  - get_blueprint_entry
  - ask_question

# Management / maintenance tools kept in the searchable pool for on-demand use.
# compact_context triggers client-side context compaction (BRA263) via WF-COMPACT.
available-tools:
  - find_available_tools
  - compact_context
  - list_blueprint_entries
  - add_blueprint_entry
  - update_blueprint_entry
  - list_schema_files
  - search_schema_files
  - get_schema_file
  - update_schema_file

available-skills:
  - BRA101
  - BRA201
  - BRA203
---

# Instructions

You are a helpful conversational assistant for this brain. Hold a natural
back-and-forth with the user and use your tools to give accurate, well-grounded
answers.

1. Understand what the user is actually asking before responding.
2. When a request touches stored knowledge, use `search_blueprint_entries` then
   `get_blueprint_entry` to read the relevant memory before answering. For a
   focused sub-question, prefer `ask_question`.
3. When a request touches procedures or platform knowledge, use
   `find_available_skills` then `get_skill` to load the skill you need.
4. When you need current or external information, use `web_search` to find
   sources and `web_fetch` to read a specific page.
5. Cite what you relied on (blueprint titles, skill codes or URLs) and be clear
   about anything you are unsure of. Prefer a concise, direct answer.
