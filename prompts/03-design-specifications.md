# Prompt 3: Design Specifications & Branding Updates

**Role:** Product Designer / Frontend Architect

**Context:**
The site currently has DevOps-specific branding ("DevOps Notes", "Learning Kubernetes, one concept at a time"). We need to make it **general-purpose** so it can host any technical content, not just DevOps/Kubernetes.

---

## Task: Rebrand Site as General-Purpose Technical Blog

### Files to Modify:

1. `/Users/tien.nguyen6/Desktop/Cake/nttien/xxntti3n.github.io/_config.yml`
2. `/Users/tien.nguyen6/Desktop/Cake/nttien/xxntti3n.github.io/index.md`
3. `/Users/tien.nguyen6/Desktop/Cake/nttien/xxntti3n.github.io/_layouts/default.html`

---

## Branding Requirements:

### 1. Site Name & Description

**Before (DevOps-specific):**
```yaml
title: DevOps Notes
description: Learning Kubernetes, one concept at a time
```

**After (General-purpose):**
```yaml
title: Tien's Blog
description: Technical guides, tutorials, and explorations in software engineering
```

### 2. Color Scheme

Keep the current clean aesthetic but make it more general:

```css
:root {
  /* Keep these - they work well */
  --bg-primary: #fafafa;
  --bg-secondary: #ffffff;
  --text-primary: #0d1117;
  --text-secondary: #3b4252;
  --accent: #FF6719;

  /* New - for general branding */
  --category-tech: #3b82f6;      /* Blue for tech posts */
  --category-tutorial: #8b5cf6;   /* Purple for tutorials */
  --category-guide: #10b981;       /* Green for comprehensive guides */
  --category-exploration: #f59e0b;  /* Amber for explorations */
}
```

### 3. Logo/Icon

**Current:** "DN" icon (DevOps Notes)

**New Options:**
1. **"TN"** - Simple initials
2. **Full name** - "Tien's Blog" in nice font
3. **Symbol** - Code bracket icon `</>`

---

## Homepage Design:

### Current Homepage Structure
```html
<!-- Currently uses home.html with blog grid -->
```

### New Homepage Structure (Guide-First)

```html
<!-- Hero: Feature the Complete Guide -->
<section class="hero">
  <h1>Technical Guides & Tutorials</h1>
  <p>Deep dives into software engineering, cloud infrastructure, and development best practices.</p>

  <div class="hero-cta">
    <a href="/kubernetes-complete-guide" class="btn-primary">
      Read the Kubernetes Guide
      <svg>→</svg>
    </a>
  </div>
</section>

<!-- Featured: Complete Kubernetes Guide -->
<section class="featured-section">
  <h2 class="section-title">Featured Guide</h2>
  <article class="featured-guide">
    <div class="guide-cover">
      <!-- Optional: K8s diagram or abstract visual -->
      <svg class="guide-icon">K8s</svg>
    </div>
    <div class="guide-info">
      <h3>Kubernetes Explained: What Really Happens Under The Hood</h3>
      <p class="guide-excerpt">A complete guide covering architecture, pods, services, networking, security, storage, and autoscaling. 15,000 words of production-ready knowledge.</p>
      <div class="guide-meta">
        <span class="guide-date">Feb 2025</span>
        <span class="guide-time">75 min read</span>
        <span class="guide-tag">Complete Guide</span>
      </div>
      <a href="/kubernetes-complete-guide" class="guide-link">
        Read Complete Guide →
      </a>
    </div>
  </article>
</section>

<!-- Topics Grid: Quick Access -->
<section class="topics-section">
  <h2 class="section-title">Browse by Topic</h2>
  <div class="topics-grid">
    <a href="/kubernetes-complete-guide#overview" class="topic-card">
      <div class="topic-icon">🎯</div>
      <h4>Kubernetes</h4>
      <p>Complete guide from basics to production</p>
    </a>

    <!-- Future topics - placeholder -->
    <a href="#" class="topic-card topic-card--placeholder">
      <div class="topic-icon">🐳</div>
      <h4>Docker</h4>
      <p>Container deep-dives coming soon</p>
    </a>

    <a href="#" class="topic-card topic-card--placeholder">
      <div class="topic-icon">☁️</div>
      <h4>Cloud</h4>
      <p>AWS, GCP, Azure patterns and guides</p>
    </a>

    <a href="#" class="topic-card topic-card--placeholder">
      <div class="topic-icon">🔧</div>
      <h4>DevOps</h4>
      <p>CI/CD, pipelines, automation</p>
    </a>
  </div>
</section>

<!-- Subscribe Section -->
<section class="subscribe-hero">
  <div class="subscribe-content">
    <h2>Stay Updated</h2>
    <p>Get new guides and tutorials delivered to your inbox weekly.</p>
    <form class="subscribe-form-large">
      <input type="email" placeholder="your@email.com" required>
      <button type="submit">Subscribe</button>
    </form>
    <p class="subscribe-terms">
      By subscribing, you agree to receive emails. Unsubscribe anytime.
    </p>
  </div>
</section>
```

---

## Post Categories (for Future Content):

Create a taxonomy system for organizing posts:

```yaml
# In post front matter
category: guide          # Complete comprehensive guide
category: tutorial       # Step-by-step tutorial
category: explanation     # Concept explanation
category: exploration     # Research/exploration
category: snippet         # Quick code snippet

# Tags remain for topic-specific
tags: [kubernetes, security, rbac]
```

---

## Design System:

### Typography Scale

| Element | Font | Size | Weight | Line-Height |
|---------|-------|-------|--------|-------------|
| Page title | Spectral | 48-56px | 700 | 1.1 |
| Section heading | Spectral | 32-36px | 700 | 1.2 |
| Subheading | Spectral | 24-28px | 600 | 1.3 |
| Body | Spectral | 18-20px | 400 | 1.75 |
| Caption/UI | Inter | 14-15px | 500 | 1.5 |
| Code | SF Mono | 14-15px | 400 | 1.6 |

### Spacing System

```css
/* Space Scale - based on 8px grid */
--space-xs: 8px;    /* Small gaps, padding */
--space-sm: 16px;   /* Card padding, list items */
--space-md: 24px;   /* Section spacing */
--space-lg: 48px;   /* Component separation */
--space-xl: 80px;   /* Major sections */
```

### Component Specifications

#### Buttons
```css
.btn-primary {
  padding: 14px 28px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border: none;
  border-radius: 30px;
  font-weight: 600;
  transition: all 0.3s ease;
  box-shadow: 0 4px 16px rgba(102, 126, 234, 0.3);
}
.btn-primary:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 24px rgba(102, 126, 234, 0.4);
}
```

#### Cards
```css
.topic-card {
  background: var(--bg-secondary);
  border: 1px solid var(--border-light);
  border-radius: 12px;
  padding: 24px;
  transition: all 0.3s ease;
  text-decoration: none;
  color: inherit;
}
.topic-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.1);
  border-color: var(--accent);
}
.topic-card--placeholder {
  opacity: 0.6;
  cursor: not-allowed;
}
```

---

## Responsive Breakpoints:

```css
/* Mobile First */
.base { /* Default styles for < 480px */ }

@media (min-width: 481px) { /* Small phones */ }
@media (min-width: 481px) { /* Large phones */ }
@media (min-width: 769px) { /* Tablets */ }
@media (min-width: 1025px) { /* Desktop */ }
@media (min-width: 1441px) { /* Large desktop */ }
```

**Mobile Optimizations:**
- Single column layout
- Stacked navigation
- Full-width hero
- Inline TOC instead of sidebar
- Touch-friendly tap targets (44px minimum)

---

## Accessibility Requirements:

### WCAG 2.1 AA Compliance
- Color contrast ratio ≥ 4.5:1 for normal text
- Color contrast ratio ≥ 3:1 for large text (18px+)
- All interactive elements 44x44px minimum
- Keyboard navigation support
- Focus indicators visible
- Skip to content link
- Proper heading hierarchy
- Alt text for images

### ARIA Implementation
```html
<!-- Skip Link -->
<a href="#main-content" class="skip-link">Skip to main content</a>

<!-- Navigation -->
<nav role="navigation" aria-label="Main navigation">
<nav>

<!-- TOC -->
<aside role="complementary" aria-label="Table of contents">
<aside>

<!-- Main Content -->
<main role="main" id="main-content">
<main>
```

---

## Social / Open Graph:

### Meta Tags
```html
<!-- Add to default.html head -->
<meta property="og:type" content="article">
<meta property="og:title" content="{{ page.title }}">
<meta property="og:description" content="{{ page.excerpt | truncate: 160 }}">
<meta property="og:image" content="{{ site.url }}/assets/og-image.png">
<meta property="og:url" content="{{ site.url }}{{ page.url }}">

<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:creator" content="@yourhandle">
```

---

## Testing Checklist:

### Branding Tests
- [ ] Site title says "Tien's Blog" not "DevOps Notes"
- [ ] Logo displays as "TN" or chosen icon
- [ ] No DevOps-exclusive language in descriptions
- [ ] Color scheme works for any topic

### Homepage Tests
- [ ] Hero section displays correctly
- [ ] Featured guide card is prominent
- [ ] Topic cards grid displays properly
- [ ] Subscribe form is visible and functional
- [ ] All links navigate correctly

### Responsive Tests
- [ ] Mobile (< 480px): Single column, readable
- [ ] Tablet (481px-768px): Optimized 2-column
- [ ] Desktop (769px+): Full layout
- [ ] Large (1441px+): Centered content max-width

### Design Tests
- [ ] Typography scale is consistent
- [ ] Spacing follows 8px grid
- [ ] Buttons have hover states
- [ ] Cards have hover/elevation states
- [ ] Colors meet contrast requirements

---

## Deliverables:

1. **Updated `_config.yml`** - General-purpose branding
2. **Updated `index.md`** - New homepage structure
3. **Updated `default.html`** - New meta tags, updated header/footer
4. **CSS variables** - New color system with categories
5. **Future-proof taxonomy** - Category system for upcoming content

---

## Expected Output:

A beautifully designed, general-purpose homepage that:
- Features the complete Kubernetes guide prominently
- Has space for future topics (Docker, Cloud, DevOps)
- Uses consistent design system
- Is fully responsive
- Is accessible (WCAG AA)
- Doesn't alienate non-DevOps readers

The site should feel like a **personal technical blog** that happens to have a comprehensive Kubernetes guide, not a "DevOps blog" that happens to have Kubernetes content.
