+++
date = '2026-02-22T16:44:48-03:00'
draft = false
title = 'From Hard‑coded Post‑Processing to a Plug‑in Skill System in Frank'
+++

## The problem

Keeping a public record of the day‑to‑day work that actually moves a product forward is hard.  
Frank, a CLI that turns Git commits into Hugo‑ready markdown, does most of the heavy lifting:

1. Pull the commit log.  
2. Group commits by day or week.  
3. Feed the raw diff and the current `README` into an LLM.  
4. Write the LLM answer to a file that Hugo can serve.

The first version of Frank produced “raw” LLM output that was clean enough to publish, but it still smelled like machine‑generated prose: inflated claims, AI‑specific vocabulary, and a laundry list of transitional phrases.  
The original implementation solved this with a single, hard‑coded “humanizer” prompt that rewrote the content after the first LLM pass.

Two problems surfaced:

* **Brittle maintenance** – every tweak to the rewrite logic meant touching the core generator code.  
* **Limited extensibility** – adding a second pass (e.g., a summary or a translation) required another hard‑coded LLM call.

The goal was to replace that monolithic post‑processing step with a **skills architecture** that lets us drop in new Markdown processors without recompiling Frank.

---

## The solution – a skill‑based pipeline

A *skill* is a small Markdown file that contains a system prompt.  
Frank loads all the requested skills from the `skills/` directory, and then runs the LLM **once per skill** in the order defined in the configuration.

### 1. New configuration option

The `.frank.toml` template now contains an optional list:

```toml
# .frank.toml.example
llm_provider = "anthropic"
llm_model = ""
period = "week"

# ordered list of post‑processing skills to run (from skills/ directory)
skills = ["humanizer"]
```

The `Config` struct was extended with a `Skills []string` field, and the TOML loader was rewritten to parse simple string arrays.

#### TOML parsing changes

```go
// internal/config/toml.go
type TOMLConfig struct {
    Values map[string]string
    Skills []string
}

func LoadTOML(path string) (*TOMLConfig, error) {
    // ... read file line by line
    if strings.HasPrefix(val, "[") && strings.HasSuffix(val, "]") {
        if key == "skills" {
            cfg.Skills = parseStringArray(inner)
        }
        continue
    }
    cfg.Values[key] = val
}
```

`parseStringArray` strips quotes and whitespace, yielding a clean slice of skill names.

### 2. Skill loader

```go
// internal/skills/skills.go
type Skill struct {
    Name   string
    Prompt string
}

func Load(dir string, names []string) ([]Skill, error) {
    var out []Skill
    for _, n := range names {
        path := filepath.Join(dir, n+".md")
        b, err := os.ReadFile(path)
        if err != nil {
            return nil, fmt.Errorf("loading skill %s: %w", n, err)
        }
        out = append(out, Skill{Name: n, Prompt: string(b)})
    }
    return out, nil
}
```

The loader is intentionally tiny – it just reads the file and returns a `Skill`.  The prompt is any Markdown‑style text the LLM will receive as a *system* prompt.

### 3. Generator now loops over skills

```go
// internal/generator/blogposts.go
type BlogPostGenerator struct {
    LLM   llm.Provider
    State *state.Store
    Templates *prompts.Templates
    Skills []skills.Skill
    // …
}

func (g *BlogPostGenerator) Generate(ctx context.Context) ([]GenerateResult, error) {
    // …
    for _, c := range commits {
        // 1. LLM generate
        content, err := g.LLM.Generate(ctx, ...)

        // 2. Run each skill in order
        for _, sk := range g.Skills {
            front, body := hugo.SplitFrontmatter(content)
            out, err := g.LLM.Generate(ctx, llm.Request{
                SystemPrompt: sk.Prompt,
                UserPrompt:   body,
                MaxTokens:    4096,
                Temperature:  0.4,
            })
            if err != nil { /* … */ }
            out = hugo.SanitizeLLMOutput(out)
            content = front + "\n" + strings.TrimSpace(out) + "\n"
        }

        // 3. Persist
        // …
    }
}
```

Two key changes:

* The **humanizer** is no longer special‑cased; it’s just another skill.  
* The LLM temperature is lowered to 0.4 for post‑processing, keeping the rewrite deterministic.

### 4. Frontmatter utilities

The original `cmd/update/home.go` contained helper functions for splitting and sanitizing Markdown.  
They were moved to `internal/hugo/frontmatter.go` so that every sub‑command can reuse them.

```go
// internal/hugo/frontmatter.go
func SplitFrontmatter(content string) (frontmatter, body string) {
    parts := strings.SplitN(content, "+++", 3)
    if len(parts) == 3 {
        frontmatter = "+++" + parts[1] + "+++\n"
        body = strings.TrimSpace(parts[2])
        return
    }
    return "", content
}

func StripFrontmatter(content string) string { … }

func SanitizeLLMOutput(s string) string {
    if start := strings.Index(s, "```"); start >= 0 {
        after := s[start+3:]
        if nl := strings.Index(after, "\n"); nl >= 0 {
            after = after[nl+1:]
        }
        if end := strings.LastIndex(after, "```"); end >= 0 {
            return strings.TrimSpace(after[:end])
        }
        return strings.TrimSpace(after)
    }
    return strings.TrimSpace(s)
}
```

Because the same logic is now shared, the home‑page update command can safely strip any accidental frontmatter that sneaks back in after an LLM pass.

---

## Concrete example – the humanizer skill

The humanizer is a Markdown file that contains a long system prompt.  It instructs the LLM to rewrite the article so it reads like a native human author and to eliminate typical AI artifacts.

```text
Rewrite the following post so it sounds like a human wrote it, not like an AI.

Look for these patterns and fix them:

**Content patterns**
- Significance inflation: remove words like “transformative”, “revolutionary”, …
- Notability emphasis: don't call things “remarkable”, “noteworthy”, …
- Promotional language: cut “vibrant”, “bustling”, “nestled”, …
- Superficial analysis: replace vague “‑ing” constructions …
- Vague attribution: replace “many experts say”, “it is widely believed” …

**Language patterns**
- AI vocabulary: replace “delve”, “landscape”, …
- Copula avoidance: replace “serves as”, “functions as” …
- Negative parallelism: rewrite “not merely X but Y” …
- Conjunctive overuse: reduce “moreover”, “furthermore”, …
- Formulaic transitions: avoid “when it comes to”, “it’s worth noting that”, …

**Style patterns**
…
```

Frank simply feeds the post body to the LLM with this prompt as the system context.  The resulting text is cleaned with `SanitizeLLMOutput` to drop any stray code fences.  The output is then concatenated with the original frontmatter and written to a Hugo file.

---

## How this benefits other developers

| Benefit | Why it matters |
|---------|----------------|
| **Extensibility** | Add new Markdown processors (e.g., a *summary* skill, a *translation* skill) by dropping a `.md` file into `skills/` and listing it in `.frank.toml`. No code changes, no rebuild. |
| **Maintainability** | The skill loader is a tiny, testable function. The generator no longer contains hard‑coded LLM calls. If the humanizer prompt needs a tweak, edit the Markdown file instead of touching Go code. |
| **Consistency** | All post‑processing passes use the same LLM provider and temperature configuration, avoiding duplicated logic across the codebase. |
| **Reusability** | The frontmatter helpers are now shared across commands. Any future feature that needs to strip or sanitize LLM output can import `internal/hugo`. |
| **Documentation** | The `.frank.toml` example now includes the `skills` field, making the architecture visible to users immediately. |
| **Open‑source friendliness** | By moving prompt logic into Markdown, contributors can tweak language without touching the binary. |

---

## Adding a new skill – a quick walkthrough

Suppose you want to add a *summary* skill that generates a short TL;DR after the post body.

1. Create `skills/summary.md`:

    ```text
    Summarize the article in three sentences, capturing the main point without AI jargon.
    ```

2. Update `.frank.toml`:

    ```toml
    skills = ["humanizer", "summary"]
    ```

3. Run `frank generate blog-posts`.  
   The generator will first apply the *humanizer*, then the *summary*, producing a final Markdown file that contains the human‑written content followed by a clean summary.

No Go code changes, no rebuild, no CI changes. This is the power of a skill system.

---

## Reflections on the design journey

The transition from a hard‑coded post‑processing step to a generic skill pipeline illustrates a classic software‑engineering trade‑off: **coupling vs. flexibility**.  
Initially, the humanizer was the only post‑processor; keeping it tightly coupled to the generator made sense.  But as the idea of adding more passes became attractive, the cost of modification skyrocketed.

By abstracting the prompt into a small, external file, we achieved:

* **Separation of concerns** – the generator orchestrates, the skill files define behavior.  
* **Testability** – unit tests can load a skill and verify its prompt content without invoking an LLM.  
* **Documentation** – the prompts live next to the code that uses them, making it easier to audit.  
* **Version control** – prompts are versioned with the repository, allowing rollback if a prompt causes regressions.

The change also required a modest overhaul of the configuration loader.  Parsing a TOML array is a tiny addition to the `LoadTOML` function, but it unlocks a more expressive configuration language.

---

## Takeaway

For teams that want to automate the publication of their development narrative, the **skill‑based approach** is a game changer:

* **Add a new post‑processing step** in minutes.  
* **Keep the core generator lean** and focused on orchestration.  
* **Expose prompts as first‑class citizens** in version control, enabling collaboration beyond the core developers.

Frank’s evolution from a single hard‑coded rewrite to a pluggable skill system demonstrates how a small architectural tweak can make a tooling project far more robust, maintainable, and community‑friendly.  If you’re building a similar pipeline, consider treating every LLM pass as a *skill* – it pays off in code quality and developer experience.
