# Tutoring Worksheet Generator

This repository contains a Zsh workflow for turning PDF or image lesson files
into a supplementary LaTeX worksheet. The script:

1. validates and copies the input lesson files into a course project;
2. renders every PDF page as a PNG while leaving PNG and JPEG inputs unchanged;
3. builds a prompt containing the output, style-reference, and image paths;
4. invokes either Codex CLI or Gemini CLI;
5. asks the selected agent to create and compile a new worksheet; and
6. stores the draft, build output, rendered pages, and logs in predictable
   locations.

The script never overwrites an existing source file, lesson image directory,
draft `.tex` file, or lesson build directory.

## Requirements

- macOS or another system with Zsh
- Poppler commands `pdfinfo` and `pdftoppm` when processing PDFs
- the standard `file` command when processing PNG or JPEG images
- `latexmk` and a working LaTeX distribution
- at least one supported coding agent:
  - Codex CLI (`codex`)
  - Gemini CLI (`gemini`)

Install Poppler on macOS with Homebrew:

```zsh
brew install poppler
```

Verify the required commands:

```zsh
command -v zsh
command -v pdfinfo
command -v pdftoppm
command -v file
command -v latexmk
command -v codex     # if using Codex
command -v gemini    # if using Gemini
```

Authenticate the selected agent before running the automated workflow.

## Repository layout

The reusable framework is organized as follows:

```text
tutoring/
├── config/
│   ├── common.env
│   └── general.env
├── prompts/
│   └── general.txt
├── scripts/
│   └── import-lessons.zsh
└── README.md
```

Course-specific configurations, prompts, source material, and generated
worksheets are intentionally excluded from Git.

## Configuration

### Shared configuration

`config/common.env` contains settings shared by all courses:

```zsh
DEFAULT_AGENT="codex"
PDF_RENDER_DPI="180"
```

- `DEFAULT_AGENT` is used when `--agent` is omitted.
- `PDF_RENDER_DPI` controls the PNG rendering resolution.

### Course configuration

Each course needs a configuration file. For example:

```zsh
# config/algebra_II.env
COURSE_NAME="algebra_II_murduca"
PROMPT_FILE="${TUTORING_DIR}/prompts/algebra_II.txt"
```

This configuration expects:

```text
tutoring/
└── prompts/
    └── algebra_II.txt
```

A formatting reference is optional and is selected per run with
`--reference`; it no longer belongs in the course configuration.

By default, the script loads `config/general.env`. Select a private
course-specific configuration by setting `TUTORING_CONFIG_FILE` for that run:

```zsh
TUTORING_CONFIG_FILE="$PWD/config/algebra_II.env" \
  ./scripts/import-lessons.zsh --help
```

Use an absolute path when running the script from outside the repository.

The shared configuration can also be replaced with:

```zsh
TUTORING_COMMON_CONFIG_FILE="/absolute/path/to/common.env" \
  TUTORING_CONFIG_FILE="/absolute/path/to/course.env" \
  ./scripts/import-lessons.zsh --help
```

## Command syntax

```text
import-lessons.zsh [--agent <agent_name>] --lesson-id <lesson_id> [--reference <file.tex>] -- <lesson-file> [lesson-file ...]
```

Options:

- `--agent codex|gemini`: selects the coding agent. If omitted, the script
  uses `DEFAULT_AGENT` from `common.env`.
- `--lesson-id ID`: identifies the new lesson and names its generated files.
  It may contain letters, numbers, periods, underscores, and hyphens.
- `--reference FILE`: optionally identifies an existing `.tex` document used
  only as a formatting reference. A relative filename is resolved inside the
  selected course directory. An absolute path is also accepted.
- `--`: ends option parsing. Every remaining argument is treated as a PDF,
  PNG, JPG, or JPEG lesson-file path.
- `-h`, `--help`: prints the usage line.

At least one supported lesson file is required. PDFs expand into rendered page
images. PNG and JPEG files are passed to the agent directly without conversion.
Mixed inputs retain their command-line order.

## Examples

### Run with Gemini

```zsh
TUTORING_CONFIG_FILE="$PWD/config/algebra_II.env" \
  ./scripts/import-lessons.zsh \
    --agent gemini \
    --lesson-id lesson-example \
    --reference lesson-1.tex \
    -- "$HOME/Downloads/lesson-example.pdf"
```

### Run with Codex

```zsh
TUTORING_CONFIG_FILE="$PWD/config/algebra_II.env" \
  ./scripts/import-lessons.zsh \
    --agent codex \
    --lesson-id transformations-review \
    --reference lesson-1.tex \
    -- "$HOME/Downloads/transformations-review.pdf"
```

### Process several PDFs as one lesson

```zsh
TUTORING_CONFIG_FILE="$PWD/config/algebra_II.env" \
  ./scripts/import-lessons.zsh \
    --agent gemini \
    --lesson-id chapter-1-review \
    -- "$HOME/Downloads/day-1.pdf" \
       "$HOME/Downloads/day-2.pdf" \
       "$HOME/Downloads/day-3.pdf"
```

### Process PDF and image files together

```zsh
TUTORING_CONFIG_FILE="$PWD/config/algebra_II.env" \
  ./scripts/import-lessons.zsh \
    --agent codex \
    --lesson-id mixed-lesson \
    -- "$HOME/Downloads/page-1.png" \
       "$HOME/Downloads/handout.pdf" \
       "$HOME/Downloads/page-4.jpg"
```

This produces the lesson order `page-1.png`, every rendered page of
`handout.pdf`, and then `page-4.jpg`.

Use `"$HOME/Downloads/file.pdf"` for paths that may contain spaces. Do not
write `"~/Downloads/file.pdf"` unless relying on the script's special handling
of a leading literal `~/`; ordinarily, Zsh does not expand a quoted tilde.

## Input lesson files are copied

After all inputs pass validation, the script copies them into:

```text
<course-directory>/sources/
```

The original files remain in Downloads or their other original location.

On macOS, the terminal or application running the script must have permission
to read Downloads. If `pdfinfo` reports `Operation not permitted`, either:

- grant the application access in **System Settings > Privacy & Security**; or
- move the PDF into the tutoring repository before running the script.

## Generated project layout

For `--lesson-id lesson-example`, the course directory will contain:

```text
<course-directory>/
├── sources/
│   ├── lesson-example.pdf
│   └── photographed-page.png
├── drafts/
│   └── lesson-example.tex
└── generated/
    ├── page-images/
    │   └── lesson-example/
    │       ├── source-01-page-1.png
    │       └── source-01-page-2.png
    ├── build/
    │   └── lesson-example/
    │       ├── lesson-example.pdf
    │       └── LaTeX intermediate files
    └── logs/
        ├── lesson-example-render.log
        ├── lesson-example-gemini.log
        └── lesson-example-codex-last-message.txt
```

The exact log names depend on the selected agent. Codex additionally writes
its final message to a separate `-last-message.txt` file.

## Agent behavior

The generated runtime prompt supplies:

- `DRAFT_TEX`: the exact destination for the editable LaTeX source;
- `STYLE_REFERENCE`: an optional existing `.tex` file used only for
  formatting, or `none provided` when `--reference` is omitted;
- `LESSON_BUILD_DIR`: the destination for compiled output; and
- `PAGE_IMAGES`: the rendered PDF pages and unchanged source images in lesson
  order.

Codex receives every lesson image as an explicit image attachment. Gemini
receives their absolute paths in the prompt and runs from the course directory.

Agent terminal output is redirected to the lesson's agent log. If the command
fails, inspect the path printed by the script.

## Common errors

### Requested style reference is missing

```text
Error: Style reference not found: .../lesson-1.tex
```

Check the value passed to `--reference`. Relative paths are resolved from the
course directory. Omit `--reference` to run without a formatting example.

### Selected agent is unavailable

```text
Error: The selected agent CLI is not installed or not on PATH
```

Install and authenticate that CLI, or select the other supported agent.

### PDF is invalid or unreadable

```text
Error: Invalid or unreadable PDF
```

Run `pdfinfo` on the file directly. An `Operation not permitted` message on
macOS indicates a privacy permission problem rather than necessarily a
damaged PDF.

### Image is invalid or mislabeled

```text
Error: Invalid or mislabeled image file
```

The file extension and detected content must agree. Supported image extensions
are `.png`, `.jpg`, and `.jpeg`.

### Output already exists

The script intentionally stops if the source destination, page-image
directory, draft, or lesson build directory already exists. Choose a new
`--lesson-id`, rename the source file, or manually review the existing output
before deciding whether it should be removed.

## Git policy

The repository uses an allowlist-style `.gitignore`: reusable scripts,
generic configuration, generic prompts, and documentation are tracked, while
course-specific material and generated artifacts remain local. When adding a
new reusable file, explicitly add a matching negation rule to `.gitignore`.
