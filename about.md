---
layout: default
title: About
description: About Thinking Inside the Box — an AAIF Ambassador technical publication on enterprise agentic AI governance.
permalink: /about/
---

<section class="space-y-8">
  <header class="space-y-4">
    <p class="text-xs font-mono uppercase tracking-wider text-brand-teal">{{ site.brand.aaif_cohort }}</p>
    <h1 class="text-3xl sm:text-4xl font-extrabold text-slate-900 tracking-tight">About This Publication</h1>
    <p class="text-lg text-slate-600 leading-relaxed">
      {{ site.brand.name }} is a dedicated technical blog — not a personal site — focused on
      enterprise-grade agentic AI architecture, governance, and open-source interoperability.
    </p>
  </header>

  <div class="prose prose-slate max-w-none font-serif
    prose-headings:font-sans prose-headings:text-slate-900
    prose-p:text-slate-700
    prose-a:text-brand-teal hover:prose-a:text-brand-crimson">
    <h2>Editorial Philosophy</h2>
    <p>
      In consumer design, "thinking outside the box" is praised. In regulated enterprise
      environments — especially financial services — letting autonomous agents run
      unconstrained is an operational and security risk. For agentic AI to be trusted in
      production, we must design systems where agents reason, plan, and execute inside
      strict, deterministic, and inspectable <strong>boxes</strong>.
    </p>

    <h2>Editorial Focus</h2>
    <ul>
      <li><strong>Bounded Autonomy</strong> — Engineering self-correcting cognitive loops that operate within system-defined permissions.</li>
      <li><strong>System-Level Control Planes</strong> — Runtime governance via agentgateway, the Model Context Protocol (MCP), and AGENTS.md.</li>
      <li><strong>Sovereign Infrastructure</strong> — Local, open-source model pipelines for data and operational sovereignty.</li>
    </ul>

    <h2>Brand & Mascot</h2>
    <p>
      <strong>{{ site.brand.mascot }}</strong> represents containment, governance, and structural
      safety — a peer concept to the AAIF mascot, always illustrated working inside the
      transparent Box. Visual identity uses Sovereign Crimson (<code>#C91D1D</code>),
      Validator Teal (<code>#0D9488</code>), and clean white surfaces with slate typography.
    </p>

    <div class="not-prose flex flex-col sm:flex-row items-start gap-6 p-6 rounded-xl border border-slate-200 bg-slate-50">
      <img
        src="{{ site.logo | relative_url }}"
        alt="{{ site.brand.mascot }}"
        class="h-32 w-32 rounded-xl object-contain bg-white border border-slate-200"
        width="128"
        height="128"
      >
      <div class="space-y-2 text-sm text-slate-600">
        <p class="font-sans font-bold text-slate-900">{{ site.author.name }}</p>
        <p>{{ site.author.role }}</p>
        <p>{{ site.author.bio }}</p>
        <p class="pt-2">
          <a href="{{ site.brand.aaif_url }}" target="_blank" rel="noopener noreferrer" class="text-brand-teal hover:text-brand-crimson transition">
            Learn about the AAIF Ambassador Program →
          </a>
        </p>
      </div>
    </div>
  </div>
</section>
