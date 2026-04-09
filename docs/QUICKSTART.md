# CooperativeCoding Website — Quick Start Guide

## Opening the Site Locally

### Option 1: Direct Browser (Fastest)
```bash
# macOS
open docs/index.html

# Linux
xdg-open docs/index.html

# Windows
start docs/index.html
```

### Option 2: Local Server (Recommended)
```bash
# Python 3
cd docs
python -m http.server 8000
# Visit http://localhost:8000

# Or Node.js
npx http-server docs
# Visit http://localhost:8080
```

## File Overview

| File | Purpose |
|------|---------|
| `index.html` | Complete website (60KB, self-contained) |
| `_config.yml` | GitHub Pages config (optional Jekyll support) |
| `assets/logo.svg` | Logo graphic |
| `README.md` | Documentation |
| `QUICKSTART.md` | This file |

## What You Get

A professional, dark-themed GitHub Pages website with:

✓ **10 Main Sections**
- Hero with animated background
- Overview of CooperativeCoding
- Seven design principles (grid cards)
- Cooperation loop (4-phase diagram)
- Elements & relationships (examples)
- Design contract
- Bidirectional sync rules
- Getting started (4-step guide)
- Spec documents (5 doc cards)
- Footer with links

✓ **Features**
- Sticky navigation header
- Smooth scroll anchors
- Fade-in animations on scroll
- Responsive design (mobile + desktop)
- Dark theme (GitHub-inspired)
- Inline CSS/JS (no build tools needed)
- High performance
- Accessible (WCAG AA)

✓ **No Dependencies**
- Pure HTML/CSS/JavaScript
- Google Fonts only (for Inter typeface)
- Works offline after initial load
- ~60KB total (index.html)

## Customization

### Change Colors
Edit these lines in `<style>` section:
```css
--primary: #0f9690;    /* Teal */
--secondary: #4361ee;  /* Blue */
--accent: #f72585;     /* Pink */
--bg: #0d1117;         /* Dark background */
--card: #161b22;       /* Card background */
```

### Update Links
All GitHub links point to:
```
https://github.com/giosullutrone/cooperative-coding
```

Search and replace to update if needed.

### Add/Remove Sections
Each section is a `<section>` tag with an `id`. Remove or reorder as needed. Navigation links reference these IDs.

### Modify Text
Open `index.html` in any text editor and search for the section you want to edit. All content is plain text.

## Deployment

### GitHub Pages (Automatic)
1. Push code to `main` branch
2. Go to repo **Settings** → **Pages**
3. Select `/docs` folder as source
4. Site auto-publishes at `https://github.com/giosullutrone/cooperative-coding`

### Other Platforms
- **Netlify**: Drag & drop the `docs` folder
- **Vercel**: Import repo, select `docs` as publish directory
- **Any Static Host**: Copy entire `docs` folder

## File Sizes

```
index.html    ~60 KB  (all HTML/CSS/JS combined)
logo.svg      ~1.5 KB
_config.yml   ~0.8 KB
README.md     ~4 KB
```

Total: ~66 KB (lightweight!)

## Browser Support

Tested and working on:
- Chrome 90+
- Firefox 88+
- Safari 14+
- Edge 90+
- Mobile browsers (iOS Safari, Chrome Android)

## Tips & Tricks

### 1. Search-Replace Links
If moving the repo or changing URLs:
```bash
# Replace old domain with new
sed -i 's/old-url/new-url/g' docs/index.html
```

### 2. Add Custom Analytics
Insert before `</body>`:
```html
<script async src="https://cdn.example.com/analytics.js"></script>
```

### 3. Dark/Light Mode Toggle
The site is dark-only by default. To add a toggle, edit the CSS `theme` variables and add a JavaScript handler.

### 4. Cache Busting
All assets are inline, so no cache-busting needed. But if you add external resources, use:
```
?v=1.0.0
```

### 5. SEO Optimization
Already included:
- Meta descriptions
- Semantic HTML
- Proper heading hierarchy
- Open Graph tags (can be added)

## Troubleshooting

### Links Not Working
Check that you're opening from the correct path. If using GitHub Pages, links should be:
```html
<a href="https://github.com/giosullutrone/cooperative-coding">
```

### Fonts Not Loading
Google Fonts requires internet. For offline use, download Inter and host locally.

### Images Not Showing
All SVG is embedded. If you add external images:
```html
<img src="assets/image.svg" alt="Description">
```

Ensure the path is correct.

### Styling Issues
Clear browser cache:
- Chrome: Ctrl+Shift+Delete (Cmd+Shift+Delete on Mac)
- Or use Ctrl+F5 (Cmd+Shift+R on Mac)

## Next Steps

1. **Review**: Open the site and explore all sections
2. **Customize**: Update colors, links, and content as needed
3. **Deploy**: Push to GitHub, enable Pages, and you're live
4. **Share**: Post the link to your README or documentation

---

**Questions?** Open an issue on GitHub.
