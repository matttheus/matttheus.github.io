# Site pessoal

Feito com [Jekyll](https://jekyllrb.com) e publicado no GitHub Pages.

## Rodar localmente

```bash
bundle install
bundle exec jekyll serve
```

Abre em <http://localhost:4000>. Mudanças em posts e páginas recarregam
sozinhas; mudanças no `_config.yml` exigem reiniciar o comando.

## Onde editar cada coisa

| O quê                                   | Arquivo               |
|-----------------------------------------|-----------------------|
| Nome, email, GitHub, YouTube, LinkedIn  | `_config.yml`         |
| Empresa e YouTube                       | `index.md`            |
| Texto da página About                   | `about.md`            |
| Lista de artigos publicados fora daqui  | `_data/articles.yml`  |
| Menu do topo                            | `header_pages` no `_config.yml` |
| Foto de perfil                          | `assets/img/matheus.jpg` |
| Favicon / avatar                        | `favicon.ico`, `assets/img/favicon-*.png`, `apple-touch-icon.png`, `avatar.png` |
| Estilo (CSS)                            | `assets/main.scss`    |

## Escrever um post

Crie um arquivo em `_posts/` com o nome `AAAA-MM-DD-titulo.md`:

```markdown
---
layout: post
title: "Título do post"
description: "Uma frase para o Google e as redes sociais."
tags: [tag1, tag2]
---

Conteúdo em Markdown.
```
