# CooperativeCoding Website

This is the GitHub Pages website for **CooperativeCoding**, an open specification for human-AI collaborative software design.

## Quick Start

1. Open `index.html` in your browser to view the site locally.
2. All assets are in the `assets/` folder.
3. The site is self-contained (no external dependencies except Google Fonts).

## File Structure

```
docs/
├── index.html           # Main website (complete HTML/CSS/JS)
├── _config.yml          # GitHub Pages configuration
├── README.md            # This file
└── assets/
    └── logo.svg         # CooperativeCoding logo
```

## Deployment

### GitHub Pages

To enable GitHub Pages for this site:

1. Go to your repository's **Settings** → **Pages**
2. Select `main` branch, `/docs` folder as the source
3. The site will be published at: `https://github.com/giosullutrone/cooperative-coding/docs`

The `_config.yml` is minimal and only necessary if you later decide to use Jekyll. For now, the plain HTML site works perfectly as-is.

### Local Testing

```bash
# Just open the file
open docs/index.html

# Or use a simple Python server
cd docs
python -m http.server 8000
# Visit http://localhost:8000
```

## Site Sections

1. **Hero** — Logo, title, tagline, version badge, CTAs
2. **Overview** — What is CooperativeCoding
3. **Principles** — Seven collaborative design principles
4. **Cooperation Loop** — Four phases of human-AI collaboration
5. **Data Model** — Node kinds, edges, stereotypes
6. **Status Lifecycle** — Node statuses and transitions
7. **Sync** — Bidirectional sync rules
8. **Getting Started** — Four-step introduction
9. **Spec Documents** — Links to specification files
10. **Footer** — License, links, attribution

## Design

- **Color Scheme**: Dark theme (GitHub-inspired)
  - Background: `#0d1117`
  - Cards: `#161b22`
  - Borders: `#30363d`
  - Primary Text: `#e6edf3`
  - Secondary Text: `#8b949e`
  - Accents: Teal `#0f9690`, Blue `#4361ee`, Pink `#f72585`

- **Typography**: Inter (via Google Fonts) with system font fallbacks
- **Responsive**: Mobile-friendly (tested at 320px, 768px, 1200px+)
- **Animations**: Smooth scrolling, fade-in on scroll, hover effects
- **No Dependencies**: Plain HTML/CSS/JS, no build tools required

## Customization

### Add SVG Diagrams

SVG diagrams can be referenced in the HTML:

```html
<svg viewBox="0 0 400 280" xmlns="http://www.w3.org/2000/svg">
  <!-- SVG content -->
</svg>
```

Or saved as external files and referenced:

```html
<img src="assets/diagram.svg" alt="Description">
```

### Update Content

All content is in the HTML. To edit:

1. Open `index.html` in a text editor
2. Find the relevant section (they're clearly commented)
3. Update the text, links, or structure
4. Save and refresh your browser

### Modify Colors

Search for color values in the CSS:
- `#0d1117` — Main background
- `#161b22` — Cards
- `#0f9690` — Primary accent (teal)
- `#4361ee` — Secondary accent (blue)
- `#f72585` — Tertiary accent (pink)

## Accessibility

- Semantic HTML structure
- Proper heading hierarchy (h1 → h2 → h3)
- Color contrast meets WCAG AA standards
- Readable font sizes (minimum 16px on mobile)
- Smooth scroll behavior
- Navigation links skip to sections

## Performance

- No external JavaScript libraries
- CSS animations use GPU acceleration (`transform`, `opacity`)
- Inline CSS and JS (no extra HTTP requests)
- Optimized SVG diagrams
- No unused CSS

## Browser Support

- Chrome/Edge 90+
- Firefox 88+
- Safari 14+
- Mobile browsers (iOS Safari, Chrome Android)

## License

CC-BY-4.0 (Creative Commons Attribution 4.0)

## Contributing

To contribute updates to the website:

1. Edit `index.html` or related files
2. Test locally by opening in your browser
3. Submit a PR with your changes

For content updates, see the specification documents in the main repository.

---

**CooperativeCoding** is an open specification. Questions? Open an issue on GitHub.
