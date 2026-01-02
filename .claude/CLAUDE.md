# Project Instructions

AWS Architecture learning scenarios for Solutions Architect certification prep.

## Exam Reference

[AWS Certified Solutions Architect - Associate (SAA-C03) Exam Guide](https://docs.aws.amazon.com/aws-certification/latest/examguides/solutions-architect-associate-03.html)

## Creating Content

### Mermaid Diagrams

Follow [mermaid-style-guide.md](./mermaid-style-guide.md) for consistent styling:

- Use `<br>` for line breaks (never `\n`)
- Always add `color:#000` for text visibility
- Add `linkStyle default stroke:#000,stroke-width:2px`
- Use the color palette (VPC: `#e3f2fd`, Public: `#c8e6c9`, Private: `#fff9c4`, RDS: `#bbdefb`)

### New Scenarios

Follow [SCENARIO-GUIDELINES.md](./SCENARIO-GUIDELINES.md) for creating new learning scenarios:

- Start with simple MVP, evolve naturally
- Focus on the WHY behind each decision
- Map to SAA exam domains
- Use realistic business triggers

**When adding a new scenario, update:**

1. `docs/_sidebar.md` - Add navigation links
2. `docs/README.md` - Add scenario card to Home page

## Project Structure

```
docs/                    # Docsify site (GitHub Pages)
├── scenarios/
│   └── techbooks/      # TechBooks scenario
└── index.html

.claude/                 # AI context and guides
├── CLAUDE.md
├── mermaid-style-guide.md
└── SCENARIO-GUIDELINES.md
```

## Formatting

Run `npm run format` to format markdown files with Prettier.
