---
layout: default
title: About
description: About thinkinginsidethebox.ai — agentic AI articles by Vincent Caldeira, AAIF Ambassador.
permalink: /about/
---

<section class="space-y-10">
  <header class="space-y-4">
    <p class="text-xs font-mono uppercase tracking-wider text-brand-teal">{{ site.brand.aaif_cohort }}</p>
    <h1 class="text-3xl sm:text-4xl font-extrabold text-slate-900 tracking-tight">About This Publication</h1>
    <p class="text-lg text-slate-600 leading-relaxed">
      <strong class="text-slate-800">{{ site.brand.domain }}</strong> is where
      <strong class="text-slate-800">{{ site.author.name }}</strong> publishes agentic AI articles —
      solo or co-authored — as part of the Agentic AI Foundation Ambassador program.
      The site covers enterprise agent architecture, open protocols, and the engineering
      practices needed to run autonomous systems safely in production.
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

    <h2>Topics</h2>
    <p>This publication focuses on the following editorial themes:</p>
  </div>

  <ul class="not-prose space-y-3">
    {% for item in site.data.keywords.topics %}
      <li class="flex items-start gap-3 p-4 rounded-xl border border-slate-200 bg-slate-50">
        <span class="inline-flex flex-shrink-0 items-center px-3 py-1 rounded-full text-xs font-medium bg-white border border-slate-200 text-brand-teal mt-0.5">
          {{ item.label }}
        </span>
        <p class="text-sm text-slate-600 leading-relaxed">{{ item.description }}</p>
      </li>
    {% endfor %}
  </ul>

  <div class="prose prose-slate max-w-none font-serif prose-headings:font-sans prose-headings:text-slate-900">
    <h2>AAIF &amp; Open-Source Projects</h2>
    <p>Articles frequently reference and contribute to these open projects:</p>
  </div>

  <ul class="not-prose space-y-3">
    {% for item in site.data.keywords.projects %}
      <li class="flex items-start gap-3 p-4 rounded-xl border border-slate-200 bg-white">
        <span class="inline-flex flex-shrink-0 items-center px-3 py-1 rounded-full text-xs font-medium bg-brand-crimson/5 border border-brand-crimson/25 text-brand-crimson mt-0.5">
          {{ item.label }}
        </span>
        <div class="text-sm text-slate-600 leading-relaxed">
          <p>{{ item.description }}</p>
          {% if item.url %}
            <p class="mt-1">
              <a href="{{ item.url }}" target="_blank" rel="noopener noreferrer" class="text-brand-teal hover:text-brand-crimson transition font-sans text-xs font-medium">
                {{ item.url | replace: 'https://', '' }} →
              </a>
            </p>
          {% endif %}
        </div>
      </li>
    {% endfor %}
  </ul>

  <div class="not-prose grid gap-6 sm:grid-cols-2">
    <div class="p-6 rounded-xl border border-slate-200 bg-slate-50 flex flex-col gap-4">
      <h2 class="text-sm font-mono uppercase tracking-wider text-brand-teal font-bold">About the Author</h2>
      <div class="flex items-start gap-4">
        {% include author-avatar.html class="h-16 w-16 rounded-full border-2 border-brand-crimson bg-slate-100 flex-shrink-0 flex items-center justify-center" size=64 %}
        <div class="space-y-1">
          <p class="font-sans font-bold text-slate-900">{{ site.author.name }}</p>
          <p class="text-xs text-slate-500 font-medium">{{ site.author.role }}</p>
        </div>
      </div>
      <p class="text-sm text-slate-600 leading-relaxed">{{ site.author.bio }}</p>
      <p class="pt-1">
        <a href="{{ site.brand.aaif_url }}" target="_blank" rel="noopener noreferrer" class="text-brand-teal hover:text-brand-crimson transition text-sm font-sans font-medium">
          AAIF Ambassador Program →
        </a>
      </p>
    </div>

    <div class="p-6 rounded-xl border border-slate-200 bg-white flex flex-col gap-4">
      <h2 class="text-sm font-mono uppercase tracking-wider text-brand-crimson font-bold">Brand Mascot</h2>
      <div class="flex items-start gap-4">
        <img
          src="{{ site.brand.mascot_logo | relative_url }}"
          alt="{{ site.brand.mascot }}"
          class="h-16 w-16 rounded-xl object-contain bg-white border border-slate-200 flex-shrink-0"
          width="64"
          height="64"
        >
        <div class="space-y-1">
          <p class="font-sans font-bold text-slate-900">{{ site.brand.mascot }}</p>
          <p class="text-xs text-slate-500">Publication brand emblem</p>
        </div>
      </div>
      <p class="text-sm text-slate-600 leading-relaxed">
        {{ site.brand.mascot }} is the visual emblem of {{ site.brand.domain }} — a minimalist
        robot who guides agent reasoning safely inside the Box. The mascot represents the
        publication's brand identity and is distinct from {{ site.author.name }}, the author
        and AAIF Ambassador behind the articles.
      </p>
    </div>
  </div>
</section>
