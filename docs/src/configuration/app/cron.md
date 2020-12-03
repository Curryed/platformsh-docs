---
title: "Cron jobs"
weight: 12
sidebarTitle: "Cron and scheduled tasks"
---

Cron jobs allow you to run scheduled tasks at specified times or intervals. The `crons` section of `.platform.app.yaml` describes these tasks and the schedule when they are triggered.  Each item in the list is a unique name identifying a separate cron job. Crons are started right after build phase.

It has a few subkeys listed below:

* **spec**: The [cron specification](https://en.wikipedia.org/wiki/Cron#CRON_expression). For example: `*/19 * * * *` to run every 19 minutes.
* **cmd**: The command to execute.
* **commands**: A set of lifecycle commands to execute.

The minimum interval between cron runs is 5 minutes, even if specified as less.  Additionally, a variable delay is added to each cron job in each project in order to prevent host overloading should every project try to run their nightly tasks at the same time.  Your crons will *not* run exactly at the time that you specify, but will be delayed by 0-300 seconds.

A single application container may have any number of cron tasks configured, but only one may be running at a time.  That is, if a cron task fires while another cron task is still running it will pause and then continue normally once the first has completed.

Cron runs are executed using the dash shell, not the bash shell used by normal SSH logins. In most cases that makes no difference but may impact some more involved cron scripts.

If an application defines both a `web` instance and a `worker` instance, cron tasks will be run only on the `web` instance.

{{< note >}}
Cron log output is captured in the at `/var/log/cron.log`.  See the [Log page](/development/logs.md) for more information on logging.
{{< /note >}}

### commands

There are two mutually-exclusive ways to define the cron commands.

The full version is the following:

```yaml
crons:
    mycommand:
        spec: '*/17 * * * *'
        commands:
            start: <command here>
            stop: <command here>
            shutdown_timeout: 10
```

The `start` key is the cron command that should execute according to the specified schedule.  If a cron task is interrupted by a user through the CLI or Web Admin Console, then the `stop` command will be issued to give the cron command a chance to shutdown gracefully, such as finish an active item in a list of tasks.  If no `stop` command is specified then a `SIGTERM` signal will be sent to the process.

If the cron process is still running after `shutdown_timeout` seconds, a `SIGKILL` signal will be sent to the process to force terminate it.  If not specified, the default `shutdown_timer` is 10 seconds.

The second cron variant uses `cmd` and specifies only a start command.  That is a shorthand for the default `shutdown_timeout` and no custom `stop` command.  That is, the following two declarations are equivalent.

```yaml
crons:
    sendemails:
        spec: '*/7 * * * *'
        commands:
            start: cd public && send-pending-emails.sh
            shutdown_timeout: 10

# or

crons:
    sendemails:
        spec: '*/7 * * * *'
        cmd: cd public && send-pending-emails.sh
```

## How do I setup Cron for a typical Drupal site?

The following example runs Drupal's normal cron hook every 19 minutes, using Drush.  It also sets up a second cron task to run Drupal's queue runner on the aggregator_feeds queue every 7 minutes.

```yaml
crons:
    # Run Drupal's cron tasks every 19 minutes.
    drupal:
        spec: '*/19 * * * *'
        cmd: 'cd web ; drush core-cron'
    # But also run pending queue tasks every 7 minutes.
    # Use an odd number to avoid running at the same time as the `drupal` cron.
    drush-queue:
        spec: '*/7 * * * *'
        cmd: 'cd web ; drush queue-run aggregator_feeds'
```
