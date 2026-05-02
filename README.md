# Research projects carried out by AI tools

Each directory in this repo is a separate research project carried out by an LLM tool - usually [Claude Code](https://www.claude.com/product/claude-code). Every single line of text and code was written by an LLM.

This setup is inspired by Simon Willison's [simonw/research](https://github.com/simonw/research) repo. See his post [Code research projects with async coding agents like Claude Code and Codex](https://simonwillison.net/2025/Nov/6/async-code-research/) for more details on how this workflow works.

I try to include prompts and links to transcripts in the PRs that added each report, or in the commits.

*Times shown are in UTC.*

<!--[[[cog
import os
import re
import subprocess
import pathlib
from datetime import datetime, timezone

# Model to use for generating summaries
MODEL = "github/gpt-4.1"

# Get all subdirectories with their first commit dates
research_dir = pathlib.Path.cwd()
subdirs_with_dates = []

for d in research_dir.iterdir():
    if d.is_dir() and not d.name.startswith('.'):
        # Get the date of the first commit that touched this directory
        try:
            result = subprocess.run(
                ['git', 'log', '-1', '--format=%aI', '--', d.name],
                capture_output=True,
                text=True,
                timeout=5
            )
            if result.returncode == 0 and result.stdout.strip():
                # Most recent commit that touched this directory
                date_str = result.stdout.strip()
                commit_date = datetime.fromisoformat(date_str.replace('Z', '+00:00'))
                subdirs_with_dates.append((d.name, commit_date))
            else:
                # No git history, use directory modification time
                subdirs_with_dates.append((d.name, datetime.fromtimestamp(d.stat().st_mtime, tz=timezone.utc)))
        except Exception:
            # Fallback to directory modification time
            subdirs_with_dates.append((d.name, datetime.fromtimestamp(d.stat().st_mtime, tz=timezone.utc)))

# Print the heading with count
print(f"## {len(subdirs_with_dates)} research projects\n")

# Sort by date, most recent first
subdirs_with_dates.sort(key=lambda x: x[1], reverse=True)

for dirname, commit_date in subdirs_with_dates:
    folder_path = research_dir / dirname
    readme_path = folder_path / "README.md"
    summary_path = folder_path / "_summary.md"

    date_formatted = commit_date.astimezone(timezone.utc).strftime('%Y-%m-%d %H:%M')

    # Get GitHub repo URL
    github_url = None
    try:
        result = subprocess.run(
            ['git', 'remote', 'get-url', 'origin'],
            capture_output=True,
            text=True,
            timeout=2
        )
        if result.returncode == 0 and result.stdout.strip():
            origin = result.stdout.strip()
            # Convert SSH URL to HTTPS URL for GitHub
            if origin.startswith('git@github.com:'):
                origin = origin.replace('git@github.com:', 'https://github.com/')
            if origin.endswith('.git'):
                origin = origin[:-4]
            github_url = f"{origin}/tree/master/{dirname}"
    except Exception:
        pass

    # Extract title from first H1 header in README, fallback to dirname
    title = dirname
    if readme_path.exists():
        with open(readme_path, 'r') as f:
            for readme_line in f:
                if readme_line.startswith('# '):
                    title = readme_line[2:].strip()
                    break

    if github_url:
        print(f"### [{title}]({github_url}#readme) ({date_formatted})\n")
    else:
        print(f"### {title} ({date_formatted})\n")

    # Check if summary already exists
    if summary_path.exists():
        # Use cached summary
        with open(summary_path, 'r') as f:
            description = f.read().strip()
            if description:
                print(description)
            else:
                print("*No description available.*")
    elif readme_path.exists():
        # Generate new summary using llm command
        prompt = """Summarize this research project concisely. Write just 1 paragraph (3-5 sentences) followed by an optional short bullet list if there are key findings. Vary your opening - don't start with "This report" or "This research". Include 1-2 links to key tools/projects. Be specific but brief. No emoji."""
        try:
            result = subprocess.run(
                ['llm', '-m', MODEL, '-s', prompt],
                stdin=open(readme_path),
                capture_output=True,
                text=True,
                timeout=60
            )
        except (FileNotFoundError, subprocess.TimeoutExpired) as e:
            import sys
            print(f"*No description available — llm invocation failed ({e}).*")
            sys.stderr.write(f"warning: llm unavailable for {dirname}: {e}\n")
            print()
            continue

        if result.returncode != 0 or not result.stdout.strip():
            import sys
            stderr_excerpt = (result.stderr or '').strip().splitlines()[-1] if result.stderr else ''
            sys.stderr.write(
                f"warning: llm summary generation failed for {dirname} "
                f"(rc={result.returncode}): {stderr_excerpt}\n"
            )
            # Fall back to a placeholder; do NOT cache so a fix retries next run.
            print("*No description available — auto-summary unavailable.*")
        else:
            description = result.stdout.strip()
            print(description)
            # Save to cache file
            with open(summary_path, 'w') as f:
                f.write(description + '\n')
    else:
        print("*No description available.*")

    print()  # Add blank line between entries

# Add AI-generated note to all project README.md files
# Note: we construct these marker strings via concatenation to avoid the HTML comment close sequence
AI_NOTE_START = "<!-- AI-GENERATED-NOTE --" + ">"
AI_NOTE_END = "<!-- /AI-GENERATED-NOTE --" + ">"
AI_NOTE_CONTENT = """> [!NOTE]
> This is an AI-generated research report. All text and code in this report was created by an LLM (Large Language Model)."""

NOT_AI_GENERATED = "<!-- not-ai-generated --" + ">"

for dirname, _ in subdirs_with_dates:
    folder_path = research_dir / dirname
    readme_path = folder_path / "README.md"

    if not readme_path.exists():
        continue

    content = readme_path.read_text()

    # Skip files marked as not AI-generated
    if NOT_AI_GENERATED in content:
        continue

    # Check if note already exists
    if AI_NOTE_START in content:
        # Replace existing note
        pattern = re.escape(AI_NOTE_START) + r'.*?' + re.escape(AI_NOTE_END)
        new_note = f"{AI_NOTE_START}\n{AI_NOTE_CONTENT}\n{AI_NOTE_END}"
        new_content = re.sub(pattern, new_note, content, flags=re.DOTALL)
        if new_content != content:
            readme_path.write_text(new_content)
    else:
        # Add note after first heading (# ...)
        lines = content.split('\n')
        new_lines = []
        note_added = False
        for i, line in enumerate(lines):
            new_lines.append(line)
            if not note_added and line.startswith('# '):
                # Add blank line, then note, then blank line
                new_lines.append('')
                new_lines.append(AI_NOTE_START)
                new_lines.append(AI_NOTE_CONTENT)
                new_lines.append(AI_NOTE_END)
                note_added = True

        if note_added:
            readme_path.write_text('\n'.join(new_lines))

]]]-->
## 2 research projects

### [LLM / Agentic Memory Systems — A Conceptual Survey](https://github.com/vertexcover-io/research/tree/master/llm-memory-systems#readme) (2026-05-02 11:45)

*No description available — auto-summary unavailable.*

### [Integrating a Python library into a TypeScript project](https://github.com/vertexcover-io/research/tree/master/python-typescript-integration#readme) (2026-05-01 09:02)

A survey of approaches for calling a Python library from a TypeScript project, using [scenedetect](https://github.com/Breakthrough/PySceneDetect) as a representative stress test (CPU-heavy, OpenCV-dependent, large binary inputs). Nine integration patterns are compared — subprocess, [FastAPI](https://fastapi.tiangolo.com/) sidecar, gRPC, job queues, native embedding, [Pyodide](https://pyodide.org/), serverless, JS-equivalent replacement, and porting — with detailed limitations covering cold-start cost, the GIL, payload size, proxy timeouts, and deployment complexity. The report ends with a decision matrix mapping situations to recommended approaches.

Key takeaways:
- Long-running subprocess workers eliminate Python import overhead for repeated calls; one-shot subprocess is fine for scripts.
- A FastAPI sidecar is the typical production answer, but watch out for the GIL blocking async endpoints and proxy timeouts on long jobs.
- Pyodide is not viable for libraries with C extensions like OpenCV.
- Always check whether a JS equivalent (e.g. `ffmpeg` scene filter) gets you 80% of the way before standing up a Python service.

<!--[[[end]]]-->

---

## Updating this README

This README uses [cogapp](https://nedbatchelder.com/code/cog/) to automatically generate project descriptions.

### Automatic updates

A GitHub Action automatically runs `cog -r -P README.md` on every push to main and commits any changes to the README or new `_summary.md` files.

### Manual updates

To update locally:

```bash
# Install dependencies
pip install -r requirements.txt

# Run cogapp to regenerate the project list
cog -r -P README.md
```

The script automatically:
- Discovers all subdirectories in this folder
- Gets the first commit date for each folder and sorts by most recent first
- For each folder, checks if a `_summary.md` file exists
- If the summary exists, it uses the cached version
- If not, it generates a new summary using `llm -m <!--[[[cog
print(MODEL, end='')
]]]-->
github/gpt-4.1
<!--[[[end]]]-->` with a prompt that creates engaging descriptions with bullets and links
- Creates markdown links to each project folder on GitHub
- New summaries are saved to `_summary.md` to avoid regenerating them on every run

To regenerate a specific project's description, delete its `_summary.md` file and run `cog -r -P README.md` again.
