# Mermaid Diagram Style Guide

Reference for consistent diagram styling across all TechBooks architecture documents.

## Key Rules

1. **Line breaks:** Use `<br>` instead of `\n`
2. **Text contrast:** Always add `color:#000` for black text on light backgrounds
3. **Quotes:** Wrap node labels in quotes when using `<br>`: `["Label<br>SubLabel"]`
4. **Arrow color:** Use `linkStyle default stroke:#000,stroke-width:2px` for black arrows

## Color Palette

| Purpose               | Fill Color | With Text                 | Example          |
| --------------------- | ---------- | ------------------------- | ---------------- |
| **VPC / Container**   | `#e3f2fd`  | `fill:#e3f2fd,color:#000` | Light blue       |
| **Public Subnet**     | `#c8e6c9`  | `fill:#c8e6c9,color:#000` | Light green      |
| **Private Subnet**    | `#fff9c4`  | `fill:#fff9c4,color:#000` | Light yellow     |
| **Database / RDS**    | `#bbdefb`  | `fill:#bbdefb,color:#000` | Blue             |
| **Warning / Problem** | `#ffcdd2`  | `fill:#ffcdd2,color:#000` | Light red        |
| **Success / Good**    | `#c8e6c9`  | `fill:#c8e6c9,color:#000` | Light green      |
| **Neutral / Future**  | `#f5f5f5`  | `fill:#f5f5f5,color:#000` | Light gray       |
| **Highlight**         | `#fff3e0`  | `fill:#fff3e0,color:#000` | Light orange     |
| **Info**              | `#e8f5e9`  | `fill:#e8f5e9,color:#000` | Very light green |

## Style Syntax Examples

### Basic node with color

```mermaid
flowchart LR
    A["Node Label"] --> B["Another Node"]
    style A fill:#c8e6c9,color:#000
    style B fill:#bbdefb,color:#000
```

### Multi-line labels

```mermaid
flowchart TB
    EC2["EC2 Instance<br>t3.micro<br>Public Subnet"]
    style EC2 fill:#fff9c4,color:#000
```

### Dashed border (for future/placeholder items)

```mermaid
flowchart LR
    Future["Reserved for<br>Future Use"]
    style Future fill:#f5f5f5,color:#000,stroke-dasharray: 5 5
```

### Subgraph styling

```mermaid
flowchart TB
    subgraph VPC["VPC: 10.0.0.0/16"]
        subgraph Public["Public Subnet"]
            EC2["EC2"]
        end
    end
    style VPC fill:#e3f2fd,color:#000
    style Public fill:#c8e6c9,color:#000
```

### Arrow/Link styling

```mermaid
flowchart LR
    A["Source"] --> B["Destination"]
    B --> C["Another"]

    %% Style all links to be black
    linkStyle default stroke:#000,stroke-width:2px
```

**Styling specific links (by index):**

```mermaid
flowchart LR
    A --> B
    B --> C
    C --> D

    %% Links are 0-indexed in order of appearance
    linkStyle 0 stroke:#000,stroke-width:2px
    linkStyle 1 stroke:#f00,stroke-width:2px
    linkStyle 2 stroke:#00f,stroke-width:2px
```

**Link style options:** | Property | Example | Description | |----------|---------|-------------| |
`stroke` | `stroke:#000` | Line color | | `stroke-width` | `stroke-width:2px` | Line thickness | |
`stroke-dasharray` | `stroke-dasharray:5,5` | Dashed line |

## AWS Component Colors

| Component      | Recommended Style         |
| -------------- | ------------------------- |
| VPC            | `fill:#e3f2fd,color:#000` |
| Public Subnet  | `fill:#c8e6c9,color:#000` |
| Private Subnet | `fill:#fff9c4,color:#000` |
| EC2            | `fill:#fff9c4,color:#000` |
| RDS            | `fill:#bbdefb,color:#000` |
| S3             | `fill:#c8e6c9,color:#000` |
| Load Balancer  | `fill:#e1f5fe,color:#000` |
| Security Group | `fill:#e8f5e9,color:#000` |
| Error/Problem  | `fill:#ffcdd2,color:#000` |

## Template

Copy this template for new diagrams:

```mermaid
flowchart TB
    User((User)) --> Internet((Internet))
    Internet --> IGW[Internet Gateway]

    subgraph VPC["VPC: 10.0.0.0/16"]
        subgraph AZ1["Availability Zone 1"]
            subgraph Public["Public Subnet"]
                EC2["EC2 Instance"]
            end
            subgraph Private["Private Subnet"]
                RDS["RDS Database"]
            end
        end
    end

    IGW --> Public
    EC2 --> RDS

    style VPC fill:#e3f2fd,color:#000
    style Public fill:#c8e6c9,color:#000
    style Private fill:#fff9c4,color:#000
    style EC2 fill:#fff9c4,color:#000
    style RDS fill:#bbdefb,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```
