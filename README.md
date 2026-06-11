# Charmed Airflow documentation

This is the code repository for Charmed Airflow documentation.
See [Official documentation](https://canonical-charmed-airflow.readthedocs-hosted.com/en/latest/) for the rendered documentation.

## Get started

This documentation is based on [Canonical's Sphinx starter pack](https://github.com/canonical/sphinx-docs-starter-pack).
See [its documentation](https://canonical-starter-pack.readthedocs-hosted.com/latest/) for more details on how to build and customize it, run automatic checks, and explore other available configurations.

### Build it

Run the following command within the `docs` folder to build the documentation locally:

```
   make run
```

### Check it

Run various checks locally with the following commands.

Check links:

```
   make linkcheck
```

Check spelling:

```
   make spelling
```

Check inclusive language:

```
   make woke
```

Check accessibility:

```
   make pa11y
```