# callback_isolated_executor_doc

Documentation site for [`callback_isolated_executor`](https://github.com/autowarefoundation/callback_isolated_executor) —
a ROS 2 executor and component container that assigns a dedicated OS thread to each callback group,
enabling per-callback scheduling control (policy, priority, CPU affinity).

The rendered site is published at
<https://autowarefoundation.github.io/callback_isolated_executor_doc/>.

## Local preview

```bash
pip install -r requirements.txt
mkdocs serve          # or: bash run.bash
```

Then open <http://127.0.0.1:8000>.

## Build

```bash
mkdocs build --strict
```

`--strict` turns warnings (such as broken internal links) into errors. The same command runs in CI on
every pull request.

## Deployment

Pushing to `main` triggers [`.github/workflows/deploy-docs.yaml`](.github/workflows/deploy-docs.yaml),
which publishes the site to the `gh-pages` branch via [mike](https://github.com/jimporter/mike) under
the `latest` alias. Enable GitHub Pages (Settings → Pages → source: `gh-pages`) once, after the first
deploy.
