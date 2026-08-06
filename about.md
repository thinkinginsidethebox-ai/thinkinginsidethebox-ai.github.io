---
layout: default
title: About
description: About thinkinginsidethebox.ai — agentic AI articles by Vincent Caldeira and collaborating authors.
permalink: /about/
---

<section class="space-y-10">
  <header class="space-y-4">
    <p class="text-xs font-mono uppercase tracking-wider text-brand-teal">{{ site.brand.aaif_cohort }}</p>
    <h1 class="text-3xl sm:text-4xl font-extrabold text-slate-900 tracking-tight">About This Publication</h1>
    <p class="text-lg text-slate-600 leading-relaxed">
      <strong class="text-slate-800">{{ site.brand.domain }}</strong> is a technical publication on agentic AI,
      hosted by <strong class="text-slate-800">{{ site.author.name }}</strong> as part of the Agentic AI Foundation
      Ambassador program and open to <strong class="text-slate-800">collaborating authors</strong>.
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

  <div class="space-y-4">
    <h2 class="text-sm font-mono uppercase tracking-wider text-brand-teal font-bold">Authors</h2>
    <p class="text-sm text-slate-600 leading-relaxed max-w-3xl">
      Articles are written by the host author and invited collaborators. Each byline links
      the piece to the person who wrote it; guest posts appear with the contributing author's profile.
    </p>
    <div class="not-prose grid gap-6 sm:grid-cols-2">
      {% for author_entry in site.data.authors %}
        {% assign author = author_entry[1] %}
        <div class="p-6 rounded-xl border border-slate-200 bg-slate-50 flex flex-col gap-4">
          <div class="flex items-start gap-4">
            {% include author-avatar.html author=author class="h-10 w-10 rounded-full border-2 border-brand-crimson object-cover object-top flex-shrink-0" size=40 %}
            <div class="space-y-1">
              <p class="font-sans font-bold text-slate-900">{{ author.name }}</p>
              <p class="text-xs text-slate-500 font-medium">{{ author.role }}</p>
              {% if author.aaif %}
                <p class="text-[0.65rem] font-mono uppercase tracking-wider text-brand-teal">Host · {{ site.brand.aaif_cohort }}</p>
              {% else %}
                <p class="text-[0.65rem] font-mono uppercase tracking-wider text-brand-teal">Contributing Author</p>
              {% endif %}
            </div>
          </div>
          <p class="text-sm text-slate-600 leading-relaxed whitespace-pre-line">{{ author.bio }}</p>
          {% if author.aaif %}
            <p class="pt-1">
              <a href="{{ site.brand.aaif_url }}" target="_blank" rel="noopener noreferrer" class="text-brand-teal hover:text-brand-crimson transition text-sm font-sans font-medium">
                AAIF Ambassador Program →
              </a>
            </p>
          {% endif %}
        </div>
      {% endfor %}

      <div class="p-6 rounded-xl border border-slate-200 bg-white flex flex-col gap-4 sm:col-span-2">
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
          publication's brand identity and is distinct from the authors behind the articles.
        </p>
      </div>
    </div>
  </div>
</section>
