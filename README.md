# Reading pgrust

Notes from reading the [pgrust](https://github.com/malisper/pgrust) source, built with [Zola](https://www.getzola.org/) 0.23.

## Writing

```sh
zola serve --drafts      # live preview at http://127.0.0.1:1111, drafts included
```

New post: copy `content/posts/reading-pgrust/` to `content/posts/<slug>/`, then set
`title`, `date`, `extra.part` and the tags. Images go next to `index.md`.
Remove `draft = true` to publish.

Link to pgrust code at the pinned commit with the `src` component (`templates/components.html`):

```
{{<src path="crates/backend/storage/smgr/smgr/src/lib.rs" />}}
{{<src path="crates/backend/storage/smgr/smgr/src/lib.rs" line={42} end={60} text="smgrread" />}}
```

When moving to a newer pgrust, bump the commit in both `zola.toml` (`extra.pgrust_commit`) and
`templates/components.html`: components can't read the site config.

Zola 0.23 templates content with Tera 2, so any literal `{{` or `{%` in a post (e.g. in a code
block showing template syntax) must be wrapped in `{% raw %}...{% endraw %}`.

Tag Postgres-related posts `postgresql`: `/tags/postgresql/atom.xml` is the feed to submit to
Planet PostgreSQL. `/atom.xml` is the feed for everything.

## Deploying

Push to `main` on GitHub, then under Settings → Pages set the source to "GitHub Actions".
The workflow in `.github/workflows/deploy.yml` builds and publishes the site, setting
`base_url` automatically. For a custom domain, add `static/CNAME` and set `base_url` in
`zola.toml`.
