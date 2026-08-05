---
layout: page
title: "Search"
permalink: /search/
sitemap: false
---

{%- comment -%}
  Hand-authored page. It lives at the site root, not in _pages/, because the
  migration script regenerates that whole folder from the WordPress export.
{%- endcomment -%}

<div id="search-page">
  <form class="searchpage-form" role="search" onsubmit="return false;">
    <label class="sr-only" for="searchpage-input">Search the site</label>
    <input type="search" id="searchpage-input" class="searchpage-input"
           placeholder="Search posts, pages and workshops…" autocomplete="off" autofocus>
  </form>
  <p id="searchpage-status" class="searchpage-status"></p>
  <div id="searchpage-results"></div>
</div>

<script>
(function () {
  var input   = document.getElementById('searchpage-input');
  var status  = document.getElementById('searchpage-status');
  var box     = document.getElementById('searchpage-results');
  var INDEX_URL = '{{ "/search.json" | relative_url }}';
  var index = null;

  // Shown in this order; anything unrecognised falls into "Other".
  var GROUPS = [
    { key: 'hyke-seminar',  label: 'HYKE seminar' },
    { key: 'invited-talks', label: 'Invited talks' },
    { key: 'news',          label: 'News' },
    { key: '__pages',       label: 'Pages' },
    { key: '__sites',       label: 'Workshops & events' },
    { key: '__other',       label: 'Other' }
  ];

  function escapeHtml(s) {
    return String(s).replace(/[&<>"']/g, function (c) {
      return { '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c];
    });
  }

  function snippet(body, terms) {
    if (!body) { return ''; }
    var low = body.toLowerCase(), at = -1;
    terms.forEach(function (t) {
      var p = low.indexOf(t);
      if (p !== -1 && (at === -1 || p < at)) { at = p; }
    });
    if (at === -1) { return escapeHtml(body.slice(0, 160)) + '…'; }
    var start = Math.max(0, at - 70);
    var text = (start > 0 ? '…' : '') + body.slice(start, start + 220) + '…';
    var out = escapeHtml(text);
    terms.forEach(function (t) {
      if (!t) { return; }
      var re = new RegExp('(' + t.replace(/[.*+?^${}()|[\]\\]/g, '\\$&') + ')', 'gi');
      out = out.replace(re, '<mark>$1</mark>');
    });
    return out;
  }

  function score(item, terms) {
    var title = (item.t || '').toLowerCase();
    var body  = (item.b || '').toLowerCase();
    var cats  = (item.c || '').toLowerCase();
    var total = 0;
    for (var i = 0; i < terms.length; i++) {
      var t = terms[i], hit = 0;
      if (!t) { continue; }
      if (title.indexOf(t) !== -1) { hit += 12; }
      if (cats.indexOf(t) !== -1)  { hit += 4; }
      var n = body.split(t).length - 1;
      if (n) { hit += Math.min(n, 6); }
      if (!hit) { return 0; }          // every term must appear somewhere
      total += hit;
    }
    return total;
  }

  function groupOf(item) {
    if (item.k === 'site')  { return '__sites'; }
    if (item.k === 'pages') { return '__pages'; }
    var cats = (item.c || '').split(',').map(function (s) { return s.trim(); });
    for (var i = 0; i < GROUPS.length; i++) {
      if (GROUPS[i].key.indexOf('__') !== 0 && cats.indexOf(GROUPS[i].key) !== -1) {
        return GROUPS[i].key;
      }
    }
    return '__other';
  }

  function render(query) {
    if (!query) {
      box.innerHTML = ''; status.textContent = 'Type something to search.';
      return;
    }
    var terms = query.toLowerCase().split(/\s+/).filter(Boolean);
    var hits = [];
    index.forEach(function (it) {
      var s = score(it, terms);
      if (s > 0) { hits.push({ it: it, s: s }); }
    });
    hits.sort(function (a, b) {
      if (b.s !== a.s) { return b.s - a.s; }
      return (b.it.d || '').localeCompare(a.it.d || '');
    });

    status.textContent = hits.length
      ? hits.length + ' result' + (hits.length === 1 ? '' : 's') + ' for “' + query + '”'
      : 'No results for “' + query + '”.';

    var buckets = {};
    hits.forEach(function (h) {
      var g = groupOf(h.it);
      (buckets[g] = buckets[g] || []).push(h);
    });

    var html = '';
    GROUPS.forEach(function (g) {
      var list = buckets[g.key];
      if (!list || !list.length) { return; }
      html += '<section class="search-group">' +
              '<h2 class="search-group-title">' + g.label +
              ' <span class="search-group-count">' + list.length + '</span></h2>' +
              '<ul class="search-results-list">';
      list.forEach(function (h) {
        var it = h.it;
        var meta = [it.d, (it.k === 'site' ? '' : it.c)].filter(Boolean).join(' · ');
        html += '<li>' +
          '<a href="' + it.u + '">' + escapeHtml(it.t) + '</a>' +
          (meta ? '<span class="search-meta">' + escapeHtml(meta) + '</span>' : '') +
          '<p class="search-snippet">' + snippet(it.b, terms) + '</p>' +
          '</li>';
      });
      html += '</ul></section>';
    });
    box.innerHTML = html;
  }

  function run(q) {
    if (index) { render(q); return; }
    status.textContent = 'Loading search index…';
    fetch(INDEX_URL)
      .then(function (r) { if (!r.ok) { throw new Error(r.status); } return r.json(); })
      .then(function (data) { index = data; render(q); })
      .catch(function () { status.textContent = 'Could not load the search index.'; });
  }

  var timer = null;
  input.addEventListener('input', function () {
    clearTimeout(timer);
    var q = input.value.trim();
    timer = setTimeout(function () { run(q); }, 150);
    history.replaceState(null, '', q ? '?q=' + encodeURIComponent(q) : location.pathname);
  });

  var q0 = new URLSearchParams(location.search).get('q') || '';
  if (q0) { input.value = q0; }
  run(q0);
})();
</script>
