---
name: kb-to-skill
description: Converts a Knowledge Base article into a skill.
---

## Convert KCS Article to Skill

Retrieve the KB article as json using the id:

```bash
ID=5564771
curl -s https://api.access.redhat.com/support/search/kcs?q=$ID | jq .
```

Do not try to retrieve any other data for the KCS article.

The example skill would look like [ETCD_SKILL.md](ETCD_SKILL.md)

Refer to these guides as definitive for creating skills.

https://github.com/anthropics/skills/blob/main/skills/claude-api/SKILL.md 
https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md 

Output the skill into the root folder.
