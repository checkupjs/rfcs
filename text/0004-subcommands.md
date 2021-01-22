- Feature Name: Checkup sub-commands
- Start Date: 2021-01-07
- RFC PR: (leave this empty)
- Checkup Issue: (leave this empty)

# Summary

[summary]: #summary

Checkup was originally conceived as a single CLI command, which would run a collection of configured tasks and output the result. In simple cases, where the nature of the tasks are fairly homogeneous, this works well - the output of those tasks can follow a fairly predictable output format. Where tasks begin to diverge in intent, this is where a once clear output format can become cluttered and unclear. This is where Checkup is today. This PR proposes a path to address this problem.

# Motivation

[motivation]: #motivation

This PR proposes sub-dividing tasks into sub-commands.

Checkup's task types have diverged enough as to warrant sub-division. Some of those divergent types are:

- Informational based tasks, which are intended to present 'pure facts' about a project. Examples would include tasks that count types, test types, lines of code, plugins, etc.
- Migration based tasks, which track the absolute count of migration steps to better estimate on their completion. The output of these tasks are more interactive via their SARIF output, when viewed in a SARIF viewer.
- [Validation/Doctor](0003-doctor.md) based tasks, which are intended to validate the structure of the project to ensure it's setup and configured correctly.

This divergence warrants subdivision of tasks, such that they're runnable as separate groups of output. This will allow for more targeted, cleaner output, more sensible groupings of data in the SARIF files, and less confusion for task authors in determining where their task should live. We will provide clear guidelines around what tasks are suitable for which sub-command, taking the decision making burden out of the developer's hands.

# Details

[details]: #details

We will leverage oclif's multi-command infrastructure to provide the CLI with the ability to run sub-commands.

## Top-level command

We will re-purpose the top-level command to reflect the changes in overall CLI structure. Invoking the top level command will result in the following output:

```shell
A health checkup for your project

VERSION
  @checkup/cli/0.13.2 darwin-x64 node-v14.15.1

USAGE
  $ checkup [COMMAND]

COMMANDS
  generate  Runs a generator to scaffold Checkup code
  info      Runs information-based tasks
  migration Runs migration-based tasks
  validate  Runs validation tasks
```

## Sub-commands

The initially proposed sub-commands are as follows:

- `info` - runs all informational-based tasks
- `migration` - runs all migration-based tasks
- `validate` (formerly proposed as [doctor](0003-doctor.md)) - runs all validation-based tasks

The same options available in the current top-level Checkup command will be available in each sub-command, including:

```shell
USAGE
  $ checkup <command> PATHS [OPTIONS]

ARGUMENTS
  PATHS  The paths that checkup will operate on. If no paths are provided, checkup will run on the entire directory beginning at --cwd.

OPTIONS
  -c, --config=config                Use this configuration, overriding .checkuprc.* if present.

  -d, --cwd=cwd                      [default: /Users/scalvert/workspace/personal/travis-web] The path referring to the root
                                     directory that Checkup will run in

  -e, --exclude-paths=exclude-paths  Paths to exclude from checkup. If paths are provided via command line and via checkup
                                     config, command line paths will be used.

  -f, --format=stdout|json           [default: stdout] The output format, one of stdout, json

  -h, --help                         show CLI help

  -l, --list-tasks                   List all available tasks to run.

  -o, --output-file=output-file      Specify file to write JSON output to. Requires the `--format` flag to be set to `json`

  -t, --task=task                    Runs specific tasks specified by the fully qualified task name in the format
                                     pluginName/taskName. Can be used multiple times.

  -v, --version                      show CLI version

  --category=category                Runs specific tasks specified by category. Can be used multiple times.

  --group=group                      Runs specific tasks specified by group. Can be used multiple times.

  --verbose
```

## Task Registration

Since tasks will be tied to the command they're intended to be run under, we'll expand the available list of hooks for task registration in order to support registering specific tasks for specific commands.

The following hooks will be available:

- `register-info-task`
- `register-migration-task`
- `register-validate-task`

This allows us to use the existing task registration infrastructure with almost no changes; tasks will explicitly be registered on the command instance they're associated with. Tasks not related to a particular command will not be loaded.
