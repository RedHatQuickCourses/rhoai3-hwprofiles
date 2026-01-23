# Insights

## January 23, 2026

### Project Structure Understanding
- The project uses Antora documentation framework with modular structure
- Content is organized in `modules/` directory with separate chapters
- Each chapter has its own `nav.adoc` for navigation and `pages/` directory for content
- Root module (`ROOT/`) serves as the entry point for the documentation site

### Hardware Profiles Content Analysis
- Chapter 1 provides comprehensive coverage of hardware profiles from conceptual introduction to advanced strategies
- Content covers both theoretical concepts (governance, ROI) and practical implementation (YAML examples, CLI commands)
- Three main strategies identified: Isolation (Taints/Tolerations), Fair-Share (Kueue), and Efficiency (Time-Slicing/MIG)
- Clear distinction between global and project-scoped profiles for governance

### AsciiDoc Include Directive
- Used `include::chapter1:pages/index.adoc[]` to map root index to chapter 1
- This approach maintains content in original location while allowing root to reference it
- Images directory path needs to be set appropriately when including content from different modules

### README Strategy
- Focused on practical "how-to" execution rather than theoretical concepts
- Organized content for quick reference with clear sections for different use cases
- Included both UI and code-based approaches to accommodate different user preferences
- Added troubleshooting section based on common issues documented in chapter 1 sections
