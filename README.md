# ansistrano-php

## Project

### Add ansistrano-php to your project

*needs to be done once*

```bash
git submodule add https://github.com/artack/ansistrano-php.git deployment
```

### Copy dist-files to your project

*needs to be done once*

```bash
cp deployment/ansible.cfg.dist ansible.cfg
cp deployment/hosts.yaml.dist hosts.yaml
cp deployment/deploy.yaml.dist deploy.yaml
cp deployment/rollback.yaml.dist rollback.yaml
```

Multistage ist also possible when copy the deploy.yaml.dist and rollback.yaml.dist multiple times, e.g.:

```bash
cp deployment/deploy.yaml.dist deploy_prod.yaml
cp deployment/deploy.yaml.dist deploy_stag.yaml
cp deployment/rollback.yaml.dist rollback_prod.yaml
cp deployment/rollback.yaml.dist rollback_stag.yaml
```

### Modify your copied files - these are YOURS
Adjust, comment, uncomment any variables like:
- git repo & branch
- paths
- project type (role)

## Database step

`hooks/symfony_after_cache.yaml` (wired via `ansistrano_symfony_after_cache_tasks_file` in `deploy.yaml.dist`) runs the database step in the new release, after composer and cache and before the release goes live. It sources `~/env.sh` when it exists, since a non-interactive shell does not read the server environment. Keep the Symfony role's own `symfony_run_doctrine_migrations` off: it runs without that environment.

The old release keeps serving traffic on the database while the step runs, so every migration must stay backward compatible with the previous release.

Pick at most one mode per playbook:

- `artack_run_doctrine_migrations: true` runs `doctrine:migrations:migrate`. Use it for prod and every stage whose data stays.
- `artack_run_stage_database: true` runs the project's `bin/stage-database`, for a stage whose database may be rebuilt (never prod). The step records the deployed branch in a `BRANCH` file of the release and compares it with the live release:
  - same branch → `bin/stage-database` (migrate only, testers keep their data)
  - another branch, no `BRANCH` file yet, or `-e artack_database_reset=true` → `bin/stage-database --reset` (rebuild: drop all tables, migrate, load fixtures)

  A CI deploy passes the branch as `artack_deploy_branch`, because it deploys the exact commit; by hand the branch entered at the prompt counts. `bin/stage-database` lives in the project and refuses to run unless the server's `APP_STAGE` (set in `~/env.sh`) is one of the stages it may rebuild.

  The step runs it as `sh bin/stage-database [--reset]` in the release root, with `~/env.sh` sourced, `APP_ENV` exported and the playbook's PHP binary in `PHP_BIN`. So the script must work under POSIX `sh` whatever its shebang, and call the console through `${PHP_BIN:-php}` to use the playbook's PHP version. A non-zero exit fails the deployment before the release goes live.

`artack_database_reset=true` fails the deployment unless the playbook sets `artack_run_stage_database: true`.
