# Frontend Build Prompt: Substack-Style Blog for xxntti3n.github.io

## Project Overview

Build a Jekyll-based blog frontend for `https://xxntti3n.github.io/` that replicates the clean, typography-focused design of Substack publications (specifically inspired by `https://luminousmen.substack.com/`).

### Current State
- **Platform**: Jekyll 3.10 hosted on GitHub Pages
- **Current Theme**: Minima (default Jekyll theme)
- **Content**: Kubernetes/DevOps tutorials (13 posts in `_posts/`)
- **Site Title**: "DevOps Notes"
- **Site Description**: "Learning Kubernetes, one concept at a time"

### Reference Design Analysis
Based on `https://luminousmen.substack.com/`:

**Key Design Elements:**
1. **Hero Section**: Large header image/banner, publication name, tagline
2. **Subscribe Box**: Prominent email subscription form with terms notice
3. **Post Layout**: Card-based or list-based post previews
4. **Typography**: Clean, readable serif for body text, sans-serif for UI elements
5. **Minimal UI**: Focus on content over chrome
6. **Responsive**: Mobile-first design

---

## Requirements

### 1. Homepage (`/`)

**Layout Structure:**
```
[Sticky Header]
    [Logo/Title]          [About] [Archive] [RSS]

[Hero Section]
    [Banner Image - Optional]
    [Site Title: "DevOps Notes"]
    [Description: "Learning Kubernetes, one concept at a time"]
    [Stats: Articles count, Launch date]

[Subscribe Section - Prominent]
    [Email Input] [Subscribe Button]
    "By subscribing, you agree to..."

[Featured Post - Optional]
    [Latest post highlighted]

[Recent Posts Grid/List]
    Post Card 1
    Post Card 2
    ...

[Footer]
    Copyright, Social Links
```

**Homepage Features:**
- Dynamic post listing from Jekyll's `site.posts`
- Reading time estimation for each post
- Publication stats (total articles, date range)
- Email subscription form (can integrate with Buttondown, Substack, or similar)
- Social links (GitHub, Twitter/X, LinkedIn)

### 2. Article Layout (`/blog/:year/:month/:day/:title/`)

**Layout Structure:**
```
[Sticky Header - Reading Progress Bar]

[Article Header]
    [Title]
    [Subtitle - Optional]
    [Author Info] [Date] [Reading Time]

[Article Content]
    [Drop cap on first letter]
    [Properly styled headings, code blocks, tables, quotes]

[Article Footer]
    [Author Bio]
    [Share Buttons]
    [Next/Previous Post Navigation]

[Related Posts - Optional]

[Subscribe CTA]
    "Enjoyed this article? Subscribe for more..."
```

**Article Features:**
- Reading progress bar at top
- Drop cap on first paragraph
- Author bio section
- Table of contents for long posts
- Syntax-highlighted code blocks (using Prism.js - already included)
- Social share buttons
- Related posts suggestions
- Subscribe CTA after article

### 3. Typography System

**Font Families:**
```css
--font-serif: 'Spectral', Georgia, serif;  /* For article body, titles */
--font-sans: 'Inter', system-ui, sans-serif;  /* For UI, nav, meta */
--font-mono: 'SF Mono', 'Monaco', monospace;  /* For code */
```

**Type Scale:**
- Article body: 18-20px, line-height 1.7-1.8
- Article title: 36-48px
- Section headings: 24-32px
- UI elements: 13-15px

### 4. Color System (Substack-inspired)

```css
:root {
  --bg-primary: #fafafa;      /* Main background */
  --bg-secondary: #ffffff;     /* Card/container background */
  --text-primary: #0d1117;     /* Main text */
  --text-secondary: #3b4252;   /* Secondary text */
  --text-tertiary: #6e7681;   /* Meta text */
  --accent: #FF6719;           /* Brand accent (orange) */
  --link-color: #0969da;       /* Links */
  --border: #e5e9ef;           /* Borders */
}
```

### 5. Components

#### Post Card
- Title, excerpt, date, reading time
- Hover effects (subtle lift, shadow)
- Category/tag badge

#### Subscribe Box
- Email input + submit button
- Optional: Integration with email service (Buttondown, ConvertKit, etc.)
- Terms of service notice

#### Navigation
- Sticky header with blur backdrop
- Smooth scroll behavior
- Mobile hamburger menu

#### Code Blocks
- Syntax highlighting (Prism.js)
- Language label
- Copy button (optional)

### 6. Responsive Design

**Breakpoints:**
- Mobile: < 480px
- Tablet: 481px - 768px
- Desktop: 769px - 1024px
- Large: > 1024px

**Mobile Considerations:**
- Single column layout
- Collapsed navigation
- Touch-friendly tap targets
- Optimized font sizes

---

## Technical Requirements

### File Structure
```
/
├── _layouts/
│   ├── default.html      # Base layout
│   ├── home.html         # Homepage layout
│   └── article.html     # Article/post layout
├── _includes/
│   ├── header.html
│   ├── footer.html
│   ├── subscribe.html
│   └── post-card.html
├── _posts/              # Existing content (13 posts)
├── _sass/               # Styles (optional, can use inline styles)
├── assets/
│   ├── css/
│   └── js/
├── index.md
└── _config.yml
```

### Jekyll Configuration

**Required in `_config.yml`:**
```yaml
# Site settings
title: DevOps Notes
description: Learning Kubernetes, one concept at a time
baseurl: ""
url: "https://xxntti3n.github.io"

# Build settings
markdown: kramdown
theme: minima  # Can be removed if building custom
plugins:
  - jekyll-feed
  - kramdown-parser-gfm

# Collections
collections:
  posts:
    output: true
    permalink: /blog/:year/:month/:day/:title/

# Defaults
defaults:
  - scope:
      path: ""
      type: "posts"
    values:
      layout: "article"
```

### JavaScript Features

**Required:**
- Reading progress bar
- Reading time calculator
- Scroll effects (header shadow on scroll)
- Back to top button
- Smooth scrolling for anchor links

**Optional:**
- Table of contents generator
- Code block copy button
- Newsletter form submission

### External Libraries (Already Used)
- **Prism.js** - Syntax highlighting (already included)
- **Mermaid.js** - Diagrams (already included)

---

## Design Deliverables

### 1. Layouts

**`_layouts/default.html`**
- Base HTML structure
- Head tags, fonts, CSS
- Header and footer includes
- Content yield

**`_layouts/home.html`**
- Hero section
- Subscribe CTA
- Featured post
- Post grid/list

**`_layouts/article.html`**
- Article header with meta
- Content with typography styling
- Author bio
- Next/prev navigation
- Subscribe CTA

### 2. Includes

**`_includes/header.html`**
- Sticky navigation
- Logo/title
- Nav links
- Progress bar (article pages)

**`_includes/footer.html`**
- Copyright
- Social links
- Sitemap links

**`_includes/subscribe.html`**
- Email form
- Terms notice
- Success state

**`_includes/post-card.html`**
- Reusable post card component
- Accepts post object as parameter

### 3. Styling

**Approach Options:**
1. **Inline styles in layouts** - Simplest for Jekyll
2. **SCSS in `_sass/`** - Better organization
3. **External CSS in `assets/css/`** - More traditional

**Recommendation**: Use SCSS in `_sass/` for maintainability.

---

## Content Migration

### Front Matter Requirements

All posts should have:
```yaml
---
layout: article
title: "Post Title"
date: YYYY-MM-DD
tags: [tag1, tag2, tag3]
author: DevOps Engineer
excerpt: "Brief description for previews..."
---
```

### Existing Posts
- 13 Kubernetes posts already exist in `_posts/`
- Verify all have proper front matter
- Add excerpts if missing

---

## Functional Requirements

### 1. RSS Feed
- Use `jekyll-feed` plugin (already in Gemfile)
- RSS link in footer/header

### 2. SEO
- Proper meta tags
- Open Graph tags for social sharing
- Structured data (JSON-LD)

### 3. Performance
- Minimal external dependencies
- Optimized font loading
- Lazy load images (if any)

### 4. Accessibility
- Semantic HTML
- ARIA labels where needed
- Keyboard navigation
- Color contrast ratios (WCAG AA)

---

## Testing Checklist

### Visual Testing
- [ ] Homepage displays correctly on mobile, tablet, desktop
- [ ] Article pages render properly
- [ ] All typography styles applied correctly
- [ ] Code blocks highlighted properly
- [ ] Mermaid diagrams render correctly
- [ ] Images responsive (if any added)

### Functional Testing
- [ ] Navigation links work
- [ ] Post cards link to correct articles
- [ ] Next/prev post navigation works
- [ ] RSS feed generates at `/feed.xml`
- [ ] Reading progress bar updates on scroll
- [ ] Back to top button appears after scrolling
- [ ] Subscribe form validates email input

### Cross-Browser Testing
- [ ] Chrome/Edge (Chromium)
- [ ] Firefox
- [ ] Safari (if available)
- [ ] Mobile browsers

### Performance Testing
- [ ] Lighthouse score > 90
- [ ] First Contentful Paint < 2s
- [ ] No layout shift issues

---

## Implementation Phases

### Phase 1: Structure (Priority: High)
1. Create `_layouts/default.html` with base structure
2. Create `_layouts/home.html` with hero and post listing
3. Create `_layouts/article.html` with article content
4. Create header and footer includes

### Phase 2: Styling (Priority: High)
1. Implement typography system
2. Style homepage components
3. Style article components
4. Add responsive breakpoints

### Phase 3: Features (Priority: Medium)
1. Reading progress bar
2. Reading time calculator
3. Subscribe box
4. Social sharing buttons

### Phase 4: Polish (Priority: Low)
1. Animations and transitions
2. Dark mode support (optional)
3. Author bio section
4. Related posts

---

## Reference Inspiration

**Primary Reference:**
- https://luminousmen.substack.com/

**Design Principles:**
- Content-first approach
- Clean typography
- Minimal chrome
- Fast loading
- Mobile-responsive

**Similar Sites to Study:**
- https://seths.blog/
- https://newslettersubscribe.substack.com/
- https://jvns.ca/

---

## Handoff Notes

### For the Implementing Agent:

1. **Start with the structure** - Build layouts first, then style
2. **Use existing content** - Work with the 13 existing posts
3. **Preserve functionality** - Keep Mermaid and Prism.js working
4. **Test locally** - Use `bundle exec jekyll serve` to preview
5. **Deploy to GitHub Pages** - The site auto-deploys on push to main

### Git Workflow:
```bash
# Install dependencies
bundle install

# Start local server
bundle exec jekyll serve

# View at http://localhost:4000
```

### Files to Modify/Create:
- `_layouts/default.html` - NEW or UPDATE
- `_layouts/home.html` - UPDATE (already exists)
- `_layouts/article.html` - UPDATE (already exists as `post.html`)
- `_includes/*.html` - NEW includes
- `_sass/*.scss` - NEW or UPDATE
- `_config.yml` - VERIFY configuration

---

## Success Criteria

The build is complete when:

1. **Homepage** displays all posts in a clean, Substack-like layout
2. **Article pages** have proper typography, reading progress, and navigation
3. **Responsive** design works on mobile and desktop
4. **Subscribe** box is prominent and functional
5. **Code blocks** are syntax-highlighted
6. **Performance** score is acceptable (Lighthouse > 90)
7. **All existing posts** render correctly without broken links

---

## Questions to Clarify Before Starting

1. **Email Service**: Which service to use for subscriptions? (Buttondown, Substack, Mailchimp?)
2. **Analytics**: Should Google Analytics or similar be added?
3. **Comments**: Do you want comments? (Disqus, giscus, etc.)
4. **Author Info**: What author name/bio to display?
5. **Social Links**: Which social platforms to link?
6. **Branding**: Keep the "DN" icon or create something new?
7. **Colors**: Any preference for accent color? (Currently orange: #FF6719)

---

## Mockup/Wireframe Reference

```
┌─────────────────────────────────────────────────────────────┐
│  [DN] DevOps Notes                  [About] [Archive]      │  ← Sticky Header
├─────────────────────────────────────────────────────────────┤
│                                                             │
│           ┌─────────────────────────────────────┐            │
│           │                                     │            │
│           │       Learning Kubernetes             │            │  ← Hero Section
│           │     one concept at a time           │            │
│           │                                     │            │
│           └─────────────────────────────────────┘            │
│                                                             │
│    ┌─────────────────────────────────────────────────┐       │
│    │  Get weekly DevOps tips in your inbox          │       │  ← Subscribe
│    │  [________________] [Subscribe]                │       │
│    └─────────────────────────────────────────────────┘       │
│                                                             │
│  FEATURED                                                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Kubernetes Overview: What is K8s?                  │   │
│  │ A complete guide to understanding Kubernetes...      │   │
│  │ Feb 11, 2025 · 8 min read                          │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  RECENT ARTICLES                                           │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐          │
│  │ Pods       │  │ Service    │  │ Deployment │          │
│  │ ...        │  │ ...        │  │ ...        │          │
│  └────────────┘  └────────────┘  └────────────┘          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Additional Resources

**Jekyll Documentation:**
- https://jekyllrb.com/docs/

**GitHub Pages:**
- https://docs.github.com/en/pages

**Substack Design Analysis:**
- Typography: Serif headings, clean sans UI
- Whitespace: Generous margins and padding
- Focus: Content takes center stage
- Minimal UI: Navigation is secondary to reading

---

*Document Version: 1.0*
*Created: 2025-02-12*
*For: xxntti3n.github.io*
