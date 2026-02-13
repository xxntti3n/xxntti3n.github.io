---
layout: default
title: About
---

<style>
  .about-page {
    max-width: 700px;
    margin: 0 auto;
    padding: 48px 24px;
  }

  .about-header {
    text-align: center;
    margin-bottom: 48px;
  }

  .about-avatar {
    width: 120px;
    height: 120px;
    background: var(--gradient-hero);
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    color: white;
    font-family: var(--font-sans);
    font-weight: 700;
    font-size: 48px;
    margin: 0 auto 24px;
    box-shadow: 0 8px 32px rgba(102, 126, 234, 0.3);
  }

  .about-name {
    font-family: var(--font-serif);
    font-size: 32px;
    font-weight: 700;
    margin: 0 0 8px;
    color: var(--text-primary);
  }

  .dark-theme .about-name {
    color: #f0f6fc;
  }

  .about-tagline {
    font-family: var(--font-sans);
    font-size: 16px;
    color: var(--text-secondary);
    margin: 0;
  }

  .dark-theme .about-tagline {
    color: #8b949e;
  }

  .about-section {
    margin-bottom: 40px;
  }

  .about-section h2 {
    font-family: var(--font-sans);
    font-size: 18px;
    font-weight: 600;
    margin: 0 0 16px;
    color: var(--text-primary);
    text-transform: uppercase;
    letter-spacing: 1px;
  }

  .dark-theme .about-section h2 {
    color: #f0f6fc;
  }

  .about-section p {
    font-family: var(--font-serif);
    font-size: 17px;
    line-height: 1.75;
    color: var(--text-secondary);
    margin: 0 0 12px;
  }

  .dark-theme .about-section p {
    color: #8b949e;
  }

  .about-links {
    display: flex;
    gap: 12px;
    flex-wrap: wrap;
    margin-top: 24px;
  }

  .about-link {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 12px 20px;
    background: var(--bg-tertiary);
    color: var(--text-primary);
    text-decoration: none;
    border-radius: var(--radius-sm);
    font-family: var(--font-sans);
    font-size: 14px;
    font-weight: 500;
    transition: all 0.2s ease;
  }

  .dark-theme .about-link {
    background: rgba(255, 255, 255, 0.1);
    color: #f0f6fc;
  }

  .about-link:hover {
    background: var(--accent);
    color: white;
    transform: translateY(-2px);
  }

  .about-link svg {
    width: 18px;
    height: 18px;
  }

  .tech-stack {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    margin-top: 16px;
  }

  .tech-tag {
    padding: 6px 12px;
    background: var(--bg-tertiary);
    color: var(--text-secondary);
    font-family: var(--font-sans);
    font-size: 13px;
    font-weight: 500;
    border-radius: 20px;
  }

  .dark-theme .tech-tag {
    background: rgba(255, 255, 255, 0.1);
    color: #8b949e;
  }

  @media (max-width: 600px) {
    .about-avatar {
      width: 100px;
      height: 100px;
      font-size: 40px;
    }

    .about-name {
      font-size: 24px;
    }

    .about-page {
      padding: 32px 20px;
    }
  }
</style>

<div class="about-page">
  <div class="about-header">
    <div class="about-avatar">TN</div>
    <h1 class="about-name">Tien Nguyen</h1>
    <p class="about-tagline">Software Engineer exploring distributed systems, cloud infrastructure, and developer tools</p>
  </div>

  <div class="about-section">
    <h2>About Me</h2>
    <p>Hi, I'm Tien — a software engineer passionate about building scalable systems and developer tools. I write about my learnings working with cloud-native technologies, container orchestration, and modern software development practices.</p>
    <p>This blog is where I share technical deep dives, tutorials, and insights from my journey in software engineering.</p>
  </div>

  <div class="about-section">
    <h2>What I Write About</h2>
    <p>• <strong>Kubernetes & Containers</strong> — Architecture, best practices, and production patterns</p>
    <p>• <strong>Cloud Infrastructure</strong> — AWS, GCP, and modern infrastructure patterns</p>
    <p>• <strong>Developer Tools</strong> — CLI tools, automation, and productivity workflows</p>
    <p>• <strong>Distributed Systems</strong> — Consensus algorithms, coordination, and scale</p>
  </div>

  <div class="about-section">
    <h2>Tech Stack</h2>
    <div class="tech-stack">
      <span class="tech-tag">Kubernetes</span>
      <span class="tech-tag">Docker</span>
      <span class="tech-tag">Go</span>
      <span class="tech-tag">Python</span>
      <span class="tech-tag">TypeScript</span>
      <span class="tech-tag">AWS</span>
      <span class="tech-tag">GCP</span>
      <span class="tech-tag">Terraform</span>
      <span class="tech-tag">GitHub Actions</span>
    </div>
  </div>

  <div class="about-section">
    <h2>Get In Touch</h2>
    <p>Feel free to reach out if you'd like to collaborate, have questions, or just want to say hello!</p>
    <div class="about-links">
      <a href="https://github.com/xxntti3n" target="_blank" rel="noopener" class="about-link">
        <svg viewBox="0 0 24 24" fill="currentColor">
          <path d="M12 0c-6.626 0-12 5.373-12 12 0 5.302 3.438 9.8 8.207 11.387.599.111.793-.261.793-.577v-2.234c-3.338.726-4.033-1.416-4.033-1.416-.546-1.387-1.333-1.756-1.333-1.756-1.089-.745.083-.729.083-.729 1.205.084 1.839 1.237 1.839 1.237 1.07 1.834 2.807 1.304 3.492.997.107-.775.418-1.305.762-1.604-2.665-.305-5.467-1.334-5.467-5.931 0-1.311.469-2.381 1.236-3.221-.124-.303-.535-1.524.117-3.176 0 0 1.008-.322 3.301 1.23.957-.266 1.983-.399 3.003-.404 1.02.005 2.047.138 3.006.404 2.291-1.552 3.297-1.23 3.297-1.23.653 1.653.242 2.874.118 3.176.77.84 1.235 1.911 1.236 3.221-1.236 4.608 0 2.807 1.803 5.626 5.469 5.921-.43.372-.823 1.102-.823 2.222v3.293c0 .319.192.694.801.576 4.765-1.589 8.199-6.086 8.199-11.386 0-6.627-5.373-12-12-12z"/>
        </svg>
        GitHub
      </a>
      <a href="https://linkedin.com/in/xxntti3n" target="_blank" rel="noopener" class="about-link">
        <svg viewBox="0 0 24 24" fill="currentColor">
          <path d="M20.447 20.452h-3.554v-5.569c0-1.328-.027-3.037-1.852-3.037-1.853 0-2.136 1.445-2.136 2.939v5.667H9.351V9h3.414v1.561h.046c.477-.9 1.637-1.85 3.37-1.85 3.601 0 4.267 2.37 4.267 5.455v6.286zM5.337 7.433c-1.144 0-2.063-.926-2.063-2.065 0-1.138.92-2.063 2.063-2.063 1.14 0 2.064.925 2.064 2.063 0 1.139-.925 2.065-2.064 2.065zm1.782 13.019H3.555V9h3.564v11.452zM22.225 0H1.771C.792 0 0 .774 0 1.729v20.542C0 23.227.792 24 1.771 24h20.451C23.2 24 24 23.227 24 22.271V1.729C24 .774 23.2 0 22.222 0h.003z"/>
        </svg>
        LinkedIn
      </a>
      <a href="mailto:thaitien0563621@gmail.com" class="about-link">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path d="M4 4h16c1.1 0 2 .9 2 2v12c0 1.1-.9 2-2 2H4c-1.1 0-2-.9-2-2V6c0-1.1.9-2 2-2z"/>
          <polyline points="22,6 12,13 2,6"/>
        </svg>
        Email
      </a>
    </div>
  </div>
</div>
