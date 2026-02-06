# builder-validator

- File ownership is the load-bearing wall. Builder owns implementation files, Validator owns test files.
- The Validator doesn't wait. The whole point is real-time feedback. The message format with severity levels (CRITICAL/WARN/NOTE) gives the Builder a clear signal on what to drop everything for vs. what to note for later.
- Dispute resolution has a hard cap. Two exchanges, then escalate to the Lead who spawns a tiebreaker subagent. Without this, two agents can ping-pong forever and burn tokens arguing.
- The Lead does nothing but coordinate. It doesn't implement, doesn't review. It spawns, creates the task list, breaks ties, and synthesizes at the end. Keeps the roles clean.

## Usage

To use it, drop the builder-validator/ folder into .claude/skills/ in your project, make sure Agent Teams is enabled, and prompt something like "Use the builder-validator skill to implement the auth module." The skill triggers on keywords like "adversarial", "builder-validator", or "red-green."
