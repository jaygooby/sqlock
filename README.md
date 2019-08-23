# `sqlock`

A simple mutex for ensuring only one instance of a command runs at a time. Ideal for long-running commands that might overlap or for starting cronjobs on clusters of servers where only one instance of the command should be run.

Uses mysql's [GET_LOCK()](https://dev.mysql.com/doc/refman/5.6/en/locking-functions.html#function_get-lock) function to orchestrate the locks.

## Dependencies
Clients need the `mysql` binary, and a mysql server must be availabe on the network. No new tables or schema changes are required to your existing mysql setup.

## Usage

```
sqlock command [args]
```

If a lock is obtained – the lock's key is generated from the command and arguments that are passed to sqlock, and thus the key will be the same for each instance of the command and arguments – the command and its arguments are run by the first instance to get a lock.

If a lock isn't obtained, sqlock quits and no command is run.

A simple test case to show that the same command can't be run is:

```
sqlock echo hello world & sqlock echo hello world
```

You should see `hello world` echoed plus a `Didn't get the lock` because the first command runs in the background, claims the unique lock for "echo hello world" and the second sqlock runs and then fails to claim the same lock.

There's a chance that both commands might run, if the first one runs quickly enough. If you've got [gnu parallel](https://www.gnu.org/software/parallel/) installed, this is a better test:

```
parallel ::: 'sqlock echo foo' 'sqlock echo foo'
```

The two calls run in parallel and one will definitely claim the lock before the other.

The lock is released once the job is done, so another simple test is to start a long running job and then duplicate it:

```
sqlock bash -c "echo sleeping; sleep 5; echo done" & \
sqlock bash -c "echo sleeping; sleep 5; echo done"
```

Because of the `sleep 5`, the lock won't be released immediately by that first backgrounded job and the second job will report `Already locked`.

## Usage with crontab
To use `sqlock` with crontab, put it in front of the actual cron task you want to run.

For example, say you've got a nightly data import task `/usr/local/bin/data-import.sh` that needs to run daily at 11pm, but you only want it to run once on your cluster of 5 servers. Add this to each of their crontabs:

```
0 23 * * * /bin/sqlock /usr/local/bin/data-import.sh
```

At 11pm each night, they'll all try and call `sqlock` with the task, but only one of them will win the lock. Your `data-import.sh` task will run on that machine and the other 4 will silently fail. Once the `data-import.sh` task has run, the machine that has the lock will drop it, ready for the next run.
