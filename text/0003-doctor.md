- Feature Name: checkup_doctor
- Start Date: 2020-11-09
- RFC PR: (leave this empty)
- Checkup Issue: (leave this empty)

# Summary

[summary]: #summary

Checkup's use cases can be expanded to include functionality that allows users to build health and quality checks, which will explicitly signal whether an application is adhering to certain configuration or structural standards.

# Motivation

[motivation]: #motivation

Checkup's current task infrastructure is focused on creating metrics aggregators - tasks that inspect the quality and structure of code, and report on that quality and structure by aggregating metrics from disparate checks. Tasks run to completion, and return a result. There is no error condition expected, and as such there's no expectation for checkup's main CLI to return a non-zero exit code on process exit.

Having the capability to create validation-style tasks that explicitly fail, or return a non-zero exit code on process exit, allow for users to produce a series of checks to ensure their project is correctly configured. This is common in other tools, such as [Homebrew's doctor](https://docs.brew.sh/Manpage#doctor-options).

This RFC proposes the addition of a new subcommand to checkup's CLI: `checkup doctor`. The `doctor` subcommand will allow for validation-style tasks to be run, which, when they fail, can return a non-zero exit code on process exit, signaling to users that validation steps have failed. This addition allows for a clean separation between checkup's standard, metrics gathering tasks vs. these specific validation tasks.

# Details

[details]: #details

Checkup's `doctor` checks will be structured very similarly to standard tasks:

- checks will be configured in the `.checkuprc` in a `checks` configuration property
- checks will be loaded via a `registerDoctorTask` hook
- checks will use a similar structure to tasks, using a class with an async `run` method

checks will differ from standard tasks in the following ways:

- They will run a series of validation steps per check
- If any single validation step fails for any single check, the entire process will exit with a non-zero exit code

As previously described, the `doctor` command will be a subcommand off the main `checkup` CLI:

```shell
# runs all checks configured in the `checks` configuration property
checkup doctor

# runs only the `volta-configuration` validation, by name
checkup doctor --check volta-configuration
```

## Output

Output will be provided in multiple formats:

1. JSON - using the SARIF format
1. stdout - using lists indicating the status of validation steps within a check

# Unresolved questions

[unresolved]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
