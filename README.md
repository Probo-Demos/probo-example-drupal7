# Probo Example: Drupal 7

A stock **Drupal 7** site (core 7.103) built by Probo from a database dump. It
is the legacy counterpart to [probo-example-drupal](../probo-example-drupal):
where that example runs Drupal 11 from a Composer build, this one demonstrates
the same **Drupal** plugin driving a non-Composer, Drush 8 workflow — the way
most surviving Drupal 7 codebases are actually laid out.

Unlike the standalone Node/Python/.NET examples (which run a single app
process), this is a full Drupal codebase served by Probo's **LAMP + Drupal**
plugin. There is no custom demo module here: the point of the example is the
build itself — importing a gzipped dump, running database updates, fixing the
files directory, clearing caches, and handing back a one-time login link.

## Example Database

The example database for this example can be downloaded
[here](https://probosupportfiles.blob.core.windows.net/utils/drupal7-example.sql.gz) —
be sure to place the `drupal7-example.sql.gz` file in your assets if you are
trying to build it on your own Probo.CI account.

## Logging in

The build's final step runs `drush uli`, which prints a one-time login URL to
the step output. Copy the path from the build log and append it to your build's
`*.probo.build` domain to land in the site as user 1 — no password required.

## How it runs in a Probo Drupal 7 container

`.probo.yaml` uses `type: lamp` with `php: 8.1` and the built-in **Drupal**
plugin:

```yaml
type: lamp
php: 8.1
database: mariadb:11.4

assets:
  - drupal7-example.sql.gz

steps:
  - name: Install Drupal 7
    plugin: Drupal
    drupalVersion: 7
    database: drupal7-example.sql.gz
    databaseGzipped: true
    clearCaches: true
    databaseUpdates: true
    subDirectory: web

  - name: Fix file permissions
    command: 'mkdir $SRC_DIR/web/sites/default/files/private && chown -R www-data:www-data $SRC_DIR/web/sites/default/files'

  - name: Cache clear
    command: "drush --root=/var/www/html cc all"

  - name: Login command
    command: "drush --root=/var/www/html uli"
```

`drupalVersion: 7` is the switch that matters. It tells Probo to install
**Drush 8** globally in the container (the last Drush release that supports
Drupal 7 — nothing needs to be committed to the repository), writes a Drupal 7
style `settings.php`, and runs the version-appropriate commands: `drush cc all`
for `clearCaches` rather than the `drush cr` used by Drupal 8+, and `drush updb`
for `databaseUpdates`. The plugin points Apache at the `web/` docroot given by
`subDirectory` and imports the gzipped dump; nginx reverse-proxies
`*.probo.build` to it.

The three `command` steps run after provisioning, in the container, with Drush
already on `$PATH`:

- **Fix file permissions** — creates `sites/default/files/private` and hands the
  whole files tree to `www-data` so Drupal can write to it. `$SRC_DIR` is the
  checked-out repository; the live docroot is `/var/www/html`.
- **Cache clear** — a second `drush cc all` after the permission change, so the
  rebuilt caches reflect the writable files directory.
- **Login command** — emits the one-time login link described above.

Standard Probo build variables and any organization/project secrets are injected
into the container automatically and are available to every step.

## Project layout

```
.probo.yaml   Probo build config (lamp type + Drupal plugin step, drupalVersion 7)
web/          Drupal 7 core docroot (sites/default/settings.php is gitignored)
```

Site files and private files (`web/sites/*/files`, `web/sites/*/private`) are
gitignored — the build creates them, and content comes from the database dump.
