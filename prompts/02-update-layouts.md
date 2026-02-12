# Prompt 2: Create Substack-Style Layout for Single Guide

**Role:** Frontend Developer / UI Engineer

**Context:**
We're creating a Substack-style single article page (reference: https://luminousmen.substack.com/p/spark-caching-explained-what-really). The combined content will be created in `kubernetes-complete-guide.md`. Now we need the layout to display it beautifully.

---

## Task: Create Guide Layout with Substack Styling

### Files to Create:

#### 1. Create: `_layouts/guide.html`

**Location:** `/Users/tien.nguyen6/Desktop/Cake/nttien/xxntti3n.github.io/_layouts/guide.html`

**Purpose:** Layout for long-form single articles with table of contents

---

### Layout Specifications:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{% if page.title %}{{ page.title }} | {% endif %}{{ site.title }}</title>

  <!-- FONTS -->
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Spectral:ital,wght@0,400;0,500;0,600;0,700;1,400&family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">

  <!-- SYNTAX HIGHLIGHTING -->
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/prism/1.29.0/themes/prism-tomorrow.min.css">
  <script src="https://cdnjs.cloudflare.com/ajax/libs/prism/1.29.0/prism.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/prism/1.29.0/components/prism-yaml.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/prism/1.29.0/components/prism-bash.min.js"></script>

  <!-- MERMAID DIAGRAMS -->
  <script type="module">
    import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
    mermaid.initialize({
      startOnLoad: true,
      theme: 'base',
      themeVariables: {
        primaryColor: '#fafafa',
        primaryTextColor: '#0d1117',
        primaryBorderColor: '#e5e9ef',
        lineColor: '#FF6719',
        secondaryColor: '#f5f7f9',
        tertiaryColor: '#ffffff',
        fontSize: '14px'
      }
    });
  </script>

  <style>
    /* CSS VARIABLES */
    :root {
      --bg-primary: #fafafa;
      --bg-secondary: #ffffff;
      --bg-tertiary: #f5f5f5;
      --text-primary: #0d1117;
      --text-secondary: #3b4252;
      --text-tertiary: #6b7280;
      --accent: #FF6719;
      --accent-hover: #e55a15;
      --border: #e5e7eb;
      --border-light: #f3f4f6;
      --code-bg: #1e1e1e;
      --quote-bg: #fff8f0;
      --quote-border: #FF6719;
      --link-color: #0969da;
      --font-serif: 'Spectral', Georgia, serif;
      --font-sans: 'Inter', -apple-system, sans-serif;
      --font-mono: 'SF Mono', 'Monaco', monospace;
      --max-width: 740px;
      --toc-width: 280px;
    }

    /* BASE STYLES */
    html { scroll-behavior: smooth; font-size: 16px; }
    body {
      margin: 0;
      padding: 0;
      font-family: var(--font-serif);
      line-height: 1.75;
      color: var(--text-primary);
      background: var(--bg-primary);
      -webkit-font-smoothing: antialiased;
    }

    /* PROGRESS BAR */
    .progress-bar {
      position: fixed;
      top: 0;
      left: 0;
      height: 3px;
      background: linear-gradient(90deg, #FF6719 0%, #764ba2 100%);
      width: 0%;
      z-index: 1000;
      transition: width 0.15s ease-out;
    }

    /* STICKY HEADER */
    .page-header {
      position: sticky;
      top: 0;
      z-index: 100;
      background: rgba(255, 255, 255, 0.95);
      backdrop-filter: blur(20px);
      border-bottom: 1px solid var(--border);
    }
    .header-content {
      max-width: 1200px;
      margin: 0 auto;
      padding: 14px 24px;
      display: flex;
      align-items: center;
      justify-content: space-between;
    }
    .site-title {
      font-family: var(--font-sans);
      font-size: 16px;
      font-weight: 600;
      color: var(--text-primary);
      text-decoration: none;
    }

    /* HERO SECTION */
    .article-hero {
      text-align: center;
      padding: 80px 24px 60px;
      max-width: 740px;
      margin: 0 auto;
    }
    .article-hero h1 {
      font-size: clamp(36px, 5vw, 56px);
      font-weight: 700;
      line-height: 1.1;
      margin: 0 0 16px;
      letter-spacing: -0.02em;
    }
    .article-hero-meta {
      display: flex;
      align-items: center;
      justify-content: center;
      gap: 24px;
      font-family: var(--font-sans);
      font-size: 14px;
      color: var(--text-tertiary);
    }
    .article-hero-author {
      display: flex;
      align-items: center;
      gap: 12px;
    }
    .author-avatar {
      width: 48px;
      height: 48px;
      border-radius: 50%;
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      display: flex;
      align-items: center;
      justify-content: center;
      color: white;
      font-weight: 600;
      font-size: 14px;
    }

    /* MAIN LAYOUT */
    .guide-container {
      max-width: 1200px;
      margin: 0 auto;
      display: grid;
      grid-template-columns: var(--toc-width) 1fr;
      gap: 48px;
      padding: 0 24px 80px;
    }

    /* TABLE OF CONTENTS */
    .toc-sidebar {
      position: sticky;
      top: 100px;
      height: fit-content;
    }
    .toc-title {
      font-family: var(--font-sans);
      font-size: 11px;
      font-weight: 600;
      text-transform: uppercase;
      letter-spacing: 1px;
      color: var(--text-tertiary);
      margin: 0 0 16px;
    }
    .toc-list {
      list-style: none;
      padding: 0;
      margin: 0;
    }
    .toc-list li {
      margin-bottom: 4px;
    }
    .toc-list a {
      font-family: var(--font-sans);
      font-size: 14px;
      color: var(--text-secondary);
      text-decoration: none;
      display: block;
      padding: 8px 12px;
      border-radius: 6px;
      transition: all 0.15s ease;
      border-left: 2px solid transparent;
    }
    .toc-list a:hover,
    .toc-list a.active {
      background: var(--bg-tertiary);
      border-left-color: var(--accent);
      color: var(--text-primary);
    }
    .toc-list .toc-number {
      color: var(--text-tertiary);
      font-size: 12px;
      margin-right: 8px;
    }

    /* ARTICLE CONTENT */
    .article-content {
      font-size: 19px;
      line-height: 1.75;
    }
    .article-content > p:first-of-type::first-letter {
      font-size: 3.5em;
      float: left;
      line-height: 0.85;
      padding-right: 12px;
      font-weight: 700;
      color: var(--accent);
    }
    .article-content h2 {
      font-family: var(--font-sans);
      font-size: 32px;
      font-weight: 700;
      margin: 72px 0 24px;
      padding-bottom: 16px;
      border-bottom: 1px solid var(--border);
    }
    .article-content h3 {
      font-family: var(--font-sans);
      font-size: 24px;
      font-weight: 600;
      margin: 48px 0 16px;
    }
    .article-content pre {
      background: var(--code-bg);
      border-radius: 8px;
      padding: 18px 20px;
      overflow-x: auto;
      margin: 24px 0;
      font-size: 14px;
      line-height: 1.6;
      position: relative;
    }
    .article-content code {
      font-family: var(--font-mono);
      font-size: 15px;
      background: var(--bg-tertiary);
      padding: 3px 8px;
      border-radius: 4px;
    }
    .article-content pre code {
      background: transparent;
      padding: 0;
      color: #e0e0e0;
    }
    .diagram-container {
      background: var(--bg-secondary);
      border: 1px solid var(--border-light);
      border-radius: 12px;
      padding: 24px;
      margin: 32px 0;
      overflow-x: auto;
    }
    .concept-box {
      background: var(--quote-bg);
      border-left: 4px solid var(--quote-border);
      border-radius: 0 8px 8px 0;
      padding: 16px 20px;
      margin: 24px 0;
    }
    .comparison-table {
      width: 100%;
      border-collapse: collapse;
      margin: 28px 0;
      font-size: 15px;
      font-family: var(--font-sans);
    }
    .comparison-table th,
    .comparison-table td {
      border: 1px solid var(--border);
      padding: 14px 16px;
      text-align: left;
    }
    .comparison-table th {
      background: var(--bg-tertiary);
      font-weight: 600;
    }

    /* SUBSCRIBE BOX */
    .subscribe-section {
      background: var(--bg-secondary);
      border: 1px solid var(--border);
      border-radius: 12px;
      padding: 40px;
      margin: 60px 0;
      text-align: center;
    }
    .subscribe-form {
      display: flex;
      gap: 12px;
      max-width: 400px;
      margin: 24px auto 0;
    }
    .subscribe-input {
      flex: 1;
      padding: 14px 20px;
      border: 1px solid var(--border);
      border-radius: 30px;
      font-size: 15px;
    }
    .subscribe-button {
      padding: 14px 28px;
      background: var(--text-primary);
      color: white;
      border: none;
      border-radius: 30px;
      font-weight: 600;
      cursor: pointer;
      transition: background 0.2s;
    }
    .subscribe-button:hover {
      background: var(--accent);
    }

    /* BACK TO TOP */
    .back-to-top {
      position: fixed;
      bottom: 32px;
      right: 32px;
      width: 48px;
      height: 48px;
      background: var(--text-primary);
      color: white;
      border: none;
      border-radius: 50%;
      cursor: pointer;
      opacity: 0;
      visibility: hidden;
      transform: translateY(10px);
      transition: all 0.3s ease;
      z-index: 100;
      display: flex;
      align-items: center;
      justify-content: center;
    }
    .back-to-top.visible {
      opacity: 1;
      visibility: visible;
      transform: translateY(0);
    }

    /* RESPONSIVE */
    @media (max-width: 1024px) {
      .guide-container {
        grid-template-columns: 1fr;
      }
      .toc-sidebar {
        position: static;
        margin-bottom: 32px;
      }
    }
    @media (max-width: 768px) {
      .article-hero h1 {
        font-size: 32px;
      }
      .article-content {
        font-size: 17px;
      }
      .article-content h2 {
        font-size: 24px;
      }
    }
    @media (max-width: 480px) {
      .article-hero {
        padding: 60px 16px 40px;
      }
      .article-hero h1 {
        font-size: 28px;
      }
      .subscribe-form {
        flex-direction: column;
      }
      .subscribe-button {
        width: 100%;
      }
    }
  </style>
</head>
<body>
  <div class="progress-bar" id="progressBar"></div>

  <header class="page-header">
    <div class="header-content">
      <a href="/" class="site-title">Tien's Blog</a>
      <nav class="header-nav">
        <a href="/">Home</a>
        <a href="/archive">Archive</a>
        <a href="/feed.xml">RSS</a>
      </nav>
    </div>
  </header>

  <article>
    <div class="article-hero">
      <h1>{{ page.title }}</h1>
      <div class="article-hero-meta">
        <div class="article-hero-author">
          <div class="author-avatar">TN</div>
          <span>{{ page.author }}</span>
        </div>
        <span>{{ page.date | date: "%B %e, %Y" }}</span>
        <span>{{ page.reading_time }} min read</span>
      </div>
      {% if page.excerpt %}
      <p class="hero-excerpt">{{ page.excerpt }}</p>
      {% endif %}
    </div>

    <div class="guide-container">
      <!-- TABLE OF CONTENTS -->
      <aside class="toc-sidebar">
        <div class="toc-title">Contents</div>
        <ol class="toc-list">
          <li><a href="#overview" class="toc-link"><span class="toc-number">01</span> Kubernetes Overview</a></li>
          <li><a href="#pod" class="toc-link"><span class="toc-number">02</span> Pod: The Smallest Unit</a></li>
          <li><a href="#deployment" class="toc-link"><span class="toc-number">03</span> Deployment: Managing Pods</a></li>
          <li><a href="#service" class="toc-link"><span class="toc-number">04</span> Service: Networking</a></li>
          <li><a href="#config" class="toc-link"><span class="toc-number">05</span> ConfigMap & Secret</a></li>
          <li><a href="#ingress" class="toc-link"><span class="toc-number">06</span> Ingress: HTTP Routing</a></li>
          <li><a href="#statefulset" class="toc-link"><span class="toc-number">07</span> StatefulSet & DaemonSet</a></li>
          <li><a href="#job" class="toc-link"><span class="toc-number">08</span> Job & CronJob</a></li>
          <li><a href="#network-policy" class="toc-link"><span class="toc-number">09</span> Network Policy</a></li>
          <li><a href="#rbac" class="toc-link"><span class="toc-number">10</span> RBAC: Access Control</a></li>
          <li><a href="#hpa" class="toc-link"><span class="toc-number">11</span> HPA: Auto-scaling</a></li>
          <li><a href="#storage" class="toc-link"><span class="toc-number">12</span> Storage: PV & PVC</a></li>
          <li><a href="#taints" class="toc-link"><span class="toc-number">13</span> Taints & Tolerations</a></li>
        </ol>
      </aside>

      <!-- MAIN CONTENT -->
      <main class="article-content">
        {{ content }}
      </main>
    </div>

    <!-- SUBSCRIBE BOX -->
    <div class="subscribe-section">
      <h3>Enjoyed this guide?</h3>
      <p>Get weekly technical tutorials and insights delivered to your inbox.</p>
      <form class="subscribe-form" onsubmit="event.preventDefault(); alert('Thanks for subscribing!');">
        <input type="email" placeholder="your@email.com" class="subscribe-input" required>
        <button type="submit" class="subscribe-button">Subscribe</button>
      </form>
      <p style="font-size: 12px; color: var(--text-tertiary); margin-top: 16px;">
        By subscribing, you agree to receive emails. Unsubscribe anytime.
      </p>
    </div>
  </article>

  <!-- FOOTER -->
  <footer class="site-footer">
    <div class="footer-content">
      <p>&copy; {{ site.time | date: "%Y" }} Tien Nguyen. Built with Jekyll.</p>
      <div class="social-links">
        <a href="https://github.com/xxntti3n" target="_blank">GitHub</a>
        <a href="https://linkedin.com/in/yourprofile" target="_blank">LinkedIn</a>
        <a href="https://twitter.com/yourhandle" target="_blank">Twitter</a>
      </div>
    </div>
  </footer>

  <button class="back-to-top" id="backToTop" aria-label="Back to top">
    <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
      <path d="M12 19V5M5 12l7-7 7 7"/>
    </svg>
  </button>

  <script>
    // Progress bar
    const progressBar = document.getElementById('progressBar');
    window.addEventListener('scroll', () => {
      const scrollTop = document.documentElement.scrollTop || document.body.scrollTop;
      const scrollHeight = document.documentElement.scrollHeight - document.documentElement.clientHeight;
      const progress = Math.min((scrollTop / scrollHeight) * 100, 100);
      progressBar.style.width = progress + '%';
    });

    // TOC active state
    const tocLinks = document.querySelectorAll('.toc-link');
    const sections = document.querySelectorAll('.article-content h2[id]');

    window.addEventListener('scroll', () => {
      let current = '';
      sections.forEach(section => {
        const sectionTop = section.offsetTop;
        if (window.pageYOffset >= sectionTop - 100) {
          current = section.getAttribute('id');
        }
      });
      tocLinks.forEach(link => {
        link.classList.remove('active');
        if (link.getAttribute('href') === '#' + current) {
          link.classList.add('active');
        }
      });
    });

    // Back to top
    const backToTop = document.getElementById('backToTop');
    window.addEventListener('scroll', () => {
      if (window.pageYOffset > 500) {
        backToTop.classList.add('visible');
      } else {
        backToTop.classList.remove('visible');
      }
    });
    backToTop.addEventListener('click', () => {
      window.scrollTo({ top: 0, behavior: 'smooth' });
    });
  </script>
</body>
</html>
```

---

## Additional Tasks:

### 2. Update: `_layouts/default.html`

**Changes:**
- Update site title from "DevOps Notes" to "Tien's Blog"
- Make header/footer more general-purpose
- Remove DevOps-specific language

### 3. Update: `_config.yml`

**Changes:**
```yaml
title: Tien's Blog
description: Technical guides, tutorials, and explorations
author:
  name: Tien Nguyen
  url: https://github.com/xxntti3n
```

---

## Testing Checklist:

After creating the layout:

- [ ] HTML validates without errors
- [ ] All CSS variables are defined
- [ ] Sticky header works correctly
- [ ] Progress bar updates on scroll
- [ ] TOC highlights current section
- [ ] TOC smooth scrolls to sections
- [ ] Mobile layout collapses to single column
- [ ] All diagrams render correctly
- [ ] Code blocks have syntax highlighting
- [ ] Subscribe form displays
- [ ] Back-to-top button appears after scrolling
- [ ] Footer displays correctly
- [ ] All fonts load properly

---

## Expected Output:

A beautiful Substack-style single article layout with:
- Sticky header with progress bar
- Hero section with author info
- Sidebar TOC (desktop) / Inline TOC (mobile)
- Main content area with beautiful typography
- Subscribe section
- Social footer
- Back-to-top button
