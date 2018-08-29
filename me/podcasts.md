---
title: Podcasts I Listen to
layout: podcasts
---

I listen to a lot of podcasts. But I also subscribe to a ton of them. I can only do this because I don't listen to every episode. Below is all the podcasts I listen to regularly, subscribe to or used to subscribe to.

{% assign sorted_podcasts = site.data.podcasts | sort: 'name' %}
---

# Favorites, listen to every episode

{% for podcast in site.data.podcasts %}
{% if podcast.tier == 1 %}
- [{{podcast.name}}]({{podcast.url}})
{% endif%}
{% endfor %}

---

# Favorite mini-series or non-regular release

{% for podcast in sorted_podcasts %}
{% if podcast.tier == 0 %}
- [{{podcast.name}}]({{podcast.url}})
{% endif%}
{% endfor %}

# Listen to most episodes, but will skip occasionally

{% for podcast in sorted_podcasts %}
{% if podcast.tier == 2 %}
- [{{podcast.name}}]({{podcast.url}})
{% endif%}
{% endfor %}

---

# Occasionally listen to an episode

{% for podcast in sorted_podcasts %}
{% if podcast.tier == 3 %}
- [{{podcast.name}}]({{podcast.url}})
{% endif%}
{% endfor %}

---

# I used to listen to these podcasts, but skip almost all episodes now.

{% for podcast in sorted_podcasts %}
{% if podcast.tier == 4 %}
- [{{podcast.name}}]({{podcast.url}})
{% endif%}
{% endfor %}

---

# Full List

{% for podcast in sorted_podcasts %}
- [{{podcast.name}}]({{podcast.url}})
{% endfor %}

