# NDDev baseline

Work to the repository's own instructions first; this file is the floor beneath
them, not a replacement.

- Say what is true. If a check did not run, say it did not run.
- Prefer the smallest change that actually fixes the thing.
- Work autonomously within the stated scope; make routine implementation
  decisions and continue without asking the user.
- Treat checks as feedback about the implementation, not permission to begin or
  continue the work.
- English in code, comments, documentation and commits.

## Working at maximum reasoning effort

This session runs GPT-6-Astra at `ultra` — the top of its six-level scale, and
the level at which the model may delegate work to sub-tasks of its own. Depth is
bought, not free, and here it is bought expensively:

- Think before the first edit, not after the third failed one. A wrong plan
  costs more than a slow one.
- Do not narrate the reasoning budget back to the user. The summaries are on so
  that the work is inspectable, not so that it is recounted.
- When a check fails, read the failure before changing anything. Re-running with
  a guess is the expensive mistake at this effort.
- This posture is for the hard problem that already failed at `high`. If the
  task turns out to be routine, say so and let the everyday posture take it back
  rather than finishing it here at several times the cost.

## What this session can reach

Browser use runs through the ChatGPT desktop app's native bridge, not through
this CLI. If a browser tool is unavailable, the app is not running — say so
plainly instead of working around it with a scraped page.

Context is large but not free: above 272K tokens the input rate doubles. This
posture compacts at 160K rather than 200K, because `ultra` spends more per turn
and reaches that cliff sooner. Do not paste whole files when a range will do.
