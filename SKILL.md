---
name: kb-to-skill
description: Converts a Red Hat Knowledge Base (KCS) article into a Claude Code skill. Use this whenever a user provides a KB article ID, KCS solution number, or asks to turn a knowledgebase article into a skill. Also trigger when users mention converting Red Hat solutions, KCS articles, or support articles into reusable troubleshooting skills.
---

## Convert a KCS Article to a Skill

This skill fetches a Red Hat KCS article by ID and produces a well-structured Claude Code skill from its content. The output skill should be action-oriented — short imperative context followed by runnable commands — not a copy-paste of the KB article.

### Step 1: Fetch the article

Retrieve the KB article JSON using the ID provided by the user. The ID argument is passed as `ID=<number>`.

```bash
curl -s https://api.access.redhat.com/support/search/kcs?q=$ID | jq .
```

Do not attempt to fetch from any other endpoint — the `/rs/solutions/` API is decommissioned. The search API returns the issue description, tags, product, component, and metadata. Resolution, root cause, and diagnostic fields are subscriber-only and will show as `"subscriber_only"` — work with what's available and supplement with domain knowledge.

### Step 2: Extract the key fields

From the JSON response, pull out:

- **`publishedTitle`** — becomes the basis for the skill name
- **`issue`** — the symptom description, log signatures, and stack traces
- **`tag` / `issueTag`** — keywords for the skill description
- **`product`** / **`component`** — scoping (e.g., RHEL 8/9, kernel, cluster)
- **`view_uri`** — link back to the original KB article

If the search returns multiple results, use the one whose `id` matches the requested ID exactly.

### Step 3: Write the skill

Follow the skill-creator guidelines for structure and writing style. Read the examples in `examples/` for the target format — particularly `ETCD_SKILL.md` for the action-oriented pattern to follow.

#### Frontmatter

```yaml
---
name: short-kebab-case-name
description: What it does + when to trigger. Be pushy — include related symptoms, log messages, and adjacent terms so the skill triggers broadly. Reference the KB article URL.
---
```

The description is the primary trigger mechanism. Include specific error strings, function names, and component names that a user might paste or mention. Explain when to use the skill, not just what it does.

#### Body structure

Organize the skill body to match the troubleshooting workflow:

1. **Signatures / Symptoms** — the log patterns, error messages, or stack traces that identify this issue. Present these as matchable patterns, not prose. A user troubleshooting this problem will paste logs — make it easy to confirm the match.

2. **Diagnose** — imperative steps with runnable commands. Each command gets a one-line explanation of *why* you're running it, not just what it does. Keep it lean.

3. **Root Cause** — 2-3 sentences explaining the underlying issue. Enough context to understand the fix, not a kernel internals lecture.

4. **Resolve** — the fix, with commands. If it's a kernel/package update, include the update command for each RHEL version. If a rolling restart is needed, explain why (quorum, service availability) and show the procedure.

5. **Workaround** (if applicable) — interim steps when the fix can't be applied immediately.

6. **Related** — errata links, related KB articles, product/component info.

#### Writing style

- Use imperative form: "Check the kernel version" not "The kernel version can be checked by"
- Explain the *why* briefly: "Reboot nodes one at a time — rebooting all at once loses quorum"
- Keep commands copy-pasteable with clear placeholder conventions (`<nodename>`, `<fsname>`)
- Don't pad with generic advice — every line should earn its place
- Don't duplicate commands that do the same thing (`mount | grep` and `cat /proc/mounts | grep` — pick one)

### Step 4: Save the output

Save the generated skill to the project root as `<DESCRIPTIVE_NAME>_SKILL.md`.

### Examples

Reference these generated skills in `examples/` for the target quality and format:

- `examples/ETCD_SKILL.md` — etcd health checks and defragmentation on OpenShift
- `examples/GFS2_GLOCK_PANIC_SKILL.md` — GFS2 glock kernel panic diagnosis on RHEL HA clusters
