---
theme: apple-basic
fonts:
  sans: Inter
  mono: Iosevka
  local: Fira Code, Roboto, Roboto Slab
info: |
  Presentation about maintenance of dbt for Oslo DBT MeetUp
title: Maintaining dbt
layout: intro-image
highlighter: shiki
lineNumbers: true
image: https://images.unsplash.com/photo-1580158118644-91fcae686c6b
date: 05/31/2023
---

# Delivering dbt with Docker and Poetry

Nikita Barskov, Senior Data Engineer @ Völur

<div class="abs-br m-6 flex gap-2">
  <a href="https://github.com/slidevjs/slidev" target="_blank" alt="GitHub"
    class="text-xl slidev-icon-btn opacity-50 !border-none !hover:text-white">
    <simple-icons-github />
  </a>
</div>

---

# Target group

We use dbt every day in our work and it is incredibly important to deliver
it reliably and consistently.

Engineers who...

- do not use Docker to deliver dbt models,
- are looking for a different way to manage dependencies for dbt,
- are looking for more reliable options to deliver dbt models in production.

---

# We want to...

- automate management of dbt version and ecosystem configuration around it,
- make sure that our we CI results are reproducible, 
- make sure that code we deliver is consistent and following our style guide.

---

# Managing dbt version

To manage dependency versions we use <a href="https://github.com/python-poetry/poetry">`poetry`</a>.

We decide to go with Poetry because

- we can declare **all** our dbt dependencies in the centralised
  `pyproject.toml`[^1][^2] file,
- Poetry generates `poetry.lock` file, which we can use to reliably create
  environments as in CI as locally,
- many tools around dbt supports `pyproject.toml` as configuration format,
- supported natively by @dependabot.

You optionally can use [pip-tools][pip-tools] or [rye][rye].

[pip-tools]: https://github.com/jazzband/pip-tools
[rye]: https://github.com/mitsuhiko/rye

[^1]: [`pyproject.toml`(PyPA documentation)](https://pip.pypa.io/en/stable/reference/build-system/pyproject-toml/)
[^2]: [`pyproject.toml`(Poetry documentation)](https://python-poetry.org/docs/pyproject/)

---

```toml {all|2-7|10-12,15-17|19-20,23|all}
[tool.poetry]
name = "change-me-dbt-project-name"
version = "0.0.0"
description = "change-me-dbt-project-description"
authors = []
readme = "README.md"
packages = []

[tool.poetry.dependencies]
python = ">=3.9,<3.12"
dbt-core = "^1.5.0"
dbt-snowflake = "^1.5.0"

[tool.poetry.group.dev.dependencies]
sqlfluff = ">=2.1.0"
sqlfluff-templater-dbt = ">=2.1.0"

[tool.sqlfluff.core]
templater = "dbt"
dialect = "snowflake"

[tool.sqlfluff.templater.jinja]
apply_dbt_builtins = true

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

---

# Packaging

We build image with our dbt code and deliver it via Container Registry to our
orchestration system.

- We control Python version,
- We encapsulate necessary dependencies and configurations in isolated manner.
- We achieve reproducible builds.
- We allow developers to run CI commands in the same environment as in actual
  CI system (GitHub Actions).
 
---

```dockerfile {all|1|2-11|all}
FROM docker.io/library/python:3.11.3@sha256:3a619e3c96fd4c5fc5e1998fd4dcb1f1403eb90c4c6409c70d7e80b9468df7df as python
FROM python as devtools

ENV VENV_PATH="/opt/poetry"
ENV PATH="${VENV_PATH}/bin:$PATH"

COPY poetry/requirements.txt requirements.txt

RUN python -m venv $VENV_PATH && \
    $VENV_PATH/bin/pip install --no-cache-dir --require-hashes --requirement requirements.txt && \
    rm -rf requirements.txt
```

---

```dockerfile {all|1-4|6-19|all}
FROM devtools as requirements

COPY pyproject.toml pyproject.toml
RUN poetry export --output=requirements.txt

FROM python as deployment

COPY --from=requirements requirements.txt .

RUN python -m pip install --no-cache-dir --require-hashes --requirement requirements.txt && \
    rm -rf requirements.txt

COPY dbt_project.yml packages.yml .ci/profiles.yml .

RUN dbt deps

COPY macros models seeds tests .

ENTRYPOINT []
```

---

# Conclusions

By using Poetry and Docker we achieved...

- consistency between environments (production, CI and local),
- reproducible builds (easy to deploy and run models across the environments),
- a huge potential for scaling in te future,
- an automation of update with @dependabot.

# Next steps

We are looking for these options in the future...

- getting CI/CD work with a modern CI/CD frameworks (such as [Earthly][earthly]
  or [Dagger][dagger]),
- getting [slim CI][slim-ci] works for us,
- integrate [model versioning][model-versioning] with versioning of our dbt
  model and image.

[earthly]: https://github.com/earthly/earthly
[dagger]: https://github.com/dagger/dagger
[slim-ci]: https://docs.getdbt.com/guides/legacy/best-practices#run-only-modified-models-to-test-changes-slim-ci
[model-versioning]: https://docs.getdbt.com/docs/collaborate/govern/model-versions

---
layout: end
class: text-center
---

# Made with 🌝 @ Völur

[Völur](https://volur.no)
