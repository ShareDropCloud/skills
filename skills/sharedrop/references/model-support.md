# Model support for the sharedrop skill

For the person choosing a model or host. No Sharedrop task needs this file.

## Recommendation

Claude Sonnet 5.5 (`claude-sonnet-5-5`) on Claude Code. Tested fallback: Claude Opus 5.5
(`claude-opus-5-5`). Configured in `SKILL.md` frontmatter as `model: claude-sonnet-5-5`
(a Claude Code field, not part of the portable Agent Skills format).

**Status: provisional for this version.** The results below were measured on the previous
version of the skill (the 6 October 2026 audit rebuild) against CLI 1.11.0. This version
changes the revise rule, the pre-check and the verification step to match CLI 1.12.0;
its eval rerun is pending, and the recommendation is confirmed or changed when it lands.

## Support matrix

| Host | Model | Status | Evidence |
|---|---|---|---|
| Claude Code 2.1.290 and 2.1.292 | Sonnet 5.5 | Tested on the previous version, recommended | 98.0% of 546 checks and expectations over 90 runs |
| Claude Code 2.1.290 and 2.1.292 | Opus 5.5 | Tested on the previous version, fallback | 98.4% of 546 checks and expectations over 90 runs |
| Claude Code | Haiku, other models | Not tested | |
| Codex | any | Not tested; ignores `model:` | |
| claude.ai and other MCP-only hosts | any | Not tested; model chosen in the host | |

Results are from round 2 of the audit evaluation: 30 cases, 3 repeats per model, run on
6 and 7 October 2026 through `claude -p`, no effort or thinking flags. Both models passed;
the 0.4 point gap is inside Sonnet's own repeat spread (1.1 points). Sonnet cost about
half as much per run ($0.11 against $0.21, notional) and ran about 20% faster (23.8 s
against 29.3 s), so it is the recommendation. Not run: a no-skill baseline, Haiku, any
host other than Claude Code, an interactive Claude Code session.

Full record: the skill audit of 6 October 2026 (`audits/sharedrop-2026-10-06/`, files
`04-evaluation-report.md` and `05-decision.md`), kept by the skill's maintainers.

## Does `model:` switch the model?

Not in the runs made so far. On Claude Code 2.1.292 (7 October 2026, headless
`claude -p`, three read-only runs: with `--model claude-opus-5-5`, with no `--model`, and
with the skill as a project skill instead of a plugin), the host read the field (the Skill
result reported `model: claude-sonnet-5-5`) but Opus 5.5 served every turn after the skill
loaded. Interactive sessions were not tested.

Until a run shows the switch, choose Sonnet yourself: `claude --model claude-sonnet-5-5`,
or `/model` in a session. Other hosts: use the host's own model setting. An organisation's
model restrictions can block either route.
