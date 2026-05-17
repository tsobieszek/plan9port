# CLAUDE.md

`rc` scripts run as acme commands to drive the font/emphasis feature.
See `README.md` for what each one does.

For the 9P control surface (`/mnt/acme`), ctl/nctl verbs, the event
protocol, and the absolute rules ("never write `dirty`/`clean` to
`ctl`", "never mutate a dirty window"), read `../llm/acme/SKILL.md`.
For the `rc` shell idioms these scripts use (lists, `~`, free carets,
heredoc rules, the p9p `awk`/`sed`/`grep` flavours), read
`../llm/rc/SKILL.md`. These scripts are the worked examples that
document references.

## Reminder

`mkfile` lists every script explicitly in `$SCRIPTS`. Whenever an acme
command is added to or removed from this directory, update that list (and
the `README.md` tables) accordingly.

Each new command should be `chmod +x`.

