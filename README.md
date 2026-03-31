# kb-to-skill

Create a skill from a kcs/kb article using the open api.

Create the skill

```bash
mkdir ~/.claude/skills/kb-to-skill
ln -s ~/git/kb-to-skill/SKILL.md ~/.claude/skills/kb-to-skill/SKILL.md
```

example: in claude to convert a kcs/kb article into a skill.

```bash
❯ /kb-to-skill ID=7123129
```

generated: [examples/GFS2_GLOCK_PANIC_SKILL.md](examples/GFS2_GLOCK_PANIC_SKILL.md)
