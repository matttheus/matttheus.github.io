---
layout: page
title: Contact
permalink: /contact/
description: How to reach Matheus: email, GitHub, YouTube and LinkedIn.
---

Want to exchange ideas, propose a project, or just say hi? These are the channels:

- **Name:** {{ site.author }}
- **Email:** [{{ site.email }}](mailto:{{ site.email }})
- **GitHub:** [github.com/{{ site.github_username }}](https://github.com/{{ site.github_username }})
- **YouTube:** [youtube.com/@{{ site.youtube_handle }}](https://www.youtube.com/@{{ site.youtube_handle }})
{%- if site.linkedin_username %}
- **LinkedIn:** [linkedin.com/in/{{ site.linkedin_username }}](https://www.linkedin.com/in/{{ site.linkedin_username }})
{%- endif %}
