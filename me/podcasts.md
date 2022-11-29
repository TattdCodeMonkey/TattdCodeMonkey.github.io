---
title: Podcasts I Listen to
layout: podcasts
---

{% assign sorted_podcasts = site.data.podcasts.list | sort: 'name' %}

{% for podcast_tier in site.data.podcasts.tiers %}

# {{ podcast_tier.header }}

{% for podcast in sorted_podcasts %}
{% if podcast.tier == podcast_tier.tier %}
{% if podcast.logo %}
<a href="{{podcast.url}}"><img src="{{podcast.logo}}" alt="{{podcast.name}}" width="250" /></a>
{% else %}

- [{{podcast.name}}]({{podcast.url}})
  {% endif %}
  {% endif%}
  {% endfor %}

---

{% endfor %}

# Full List

{% for podcast in sorted_podcasts %}
{% if podcast.logo %}
<a href="{{podcast.url}}"><img src="{{podcast.logo}}" alt="{{podcast.name}}" width="250" /></a>
{% else %}

- [{{podcast.name}}]({{podcast.url}})
  {% endif %}
  {% endfor %}
