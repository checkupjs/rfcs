- Feature Name: Task Result JSON Schema
- Start Date: 2020-07-02
- RFC PR: (leave this empty)
- Checkup Issue: (leave this empty)

# Summary

[summary]: #summary

Up to now, Checkup's development has been largely focused on the basic, core feature set that enables experimentation and iteration, end-to-end. At this point, we have a reasonable set of experimental tasks that it's become clear there's a need to standardize the output.

# Motivation

[motivation]: #motivation

Similar to other projects that concern themselves with code quality (eslint, ember-template-lint, etc), we should seek to provide standardized, predictable, and ultimately parsable output from tasks. Adding this additional structure to output data will allow us to adapt Checkup for future use cases.

## Consumption of Generated Data

Checkup's primary use case is to summarize insights through static analysis. As such, its primary consumption modalities are

1. A one-time snapshot report summarizing the data
1. Historical data that can visualize changes over time

Summarization, and the associated insights and actions that are generated from that data, is largely tied to the task's implementation; the task defines what's important information to derive.

Secondary consumption modalities are

1. Richer reporting in the one-time snapshot, where issues can be tied back directly to lines of code in the associated repository
1. The ability to run Checkup in Github pull requests, and annotate files with specific code quality issues
1. Integrate more tightly with IDEs like vscode, where Checkup can be run on the currently open file(s), allowing for inline annotations on code quality issues

These secondary forms of data usage are tied more directly to specific files, and also can potentially duplicate the role of other tools in annotating files' specific issues. We should seek to provide an enhancement in insights over what is provided by established tools such as eslint/ember-template-lint, but not supplant them.

Implementing a more structured approach to data will allow us to provide version guarantees around functionality, and will give users confidence when authoring tasks that we're not going to break their functionality.

Specifically, we want to design a structure for task results that is as generic as possible, allowing for the representation and display of that data to accommodate the use cases referenced above.

# Details

[details]: #details

As previously mentioned, Checkup is similar to, and uses, other static analysis tools such as [eslint](https://eslint.org/) and [ember-template-lint](https://github.com/ember-template-lint/ember-template-lint) to generate its data. While there are similarities, there are also fundamental differences.

## Task based vs. File based

The aforementioned tools mainly operate at the file level, and provide insight into specific files to help guide the user to addressing issues. Checkup, on the other hand, is a task-based tool who's main goal is to aggregate information based on groupings of information defined in tasks. The primary function is the summarization of that data, while the secondary function is to annotate issues at the file level (when running Checkup on a Github PR, or when configured to run in an IDE such as VSCode).

This is mainly due to Checkup's difference in function: insights are gained through the grouping and aggregation of data, and actions are determined by the makeup of that data. While annotating information on files is important, there are already well established tools that do this (the previously mentioned linters), and Checkup should not seek to supplant those tools as this would be a confusing experience to users.

Ultimately, whether Check was to operate against tasks vs. files is relevant but not critical; either mechanism will allow the data to be subsequently formatted for specific purposes.

## Types of Data Gathered by Tasks

Tasks are broken up into a few distinct types

1. Informational (meta) tasks

   Gathers data related to the project, largely irrespective of where that data lives (not concerned with specific files)

   eg. Project info (name, version, etc.) / repository info (git stats)

1. Data gathering tasks

   **Count-based**

   Gathers project structure data that seeks to provide aggregate counts to illuminate the 'shape' of the project

   eg. Project types

   **File-based**

   Gathers info, largely from linters, that give line/column information on specific files

Providing more structure that addresses these differing types of data will allow for a structured approach to the output, allowing for external consumers of the data to have some guarantees when integrating and consuming it.

# Proposal

[proposal]: #proposal

Based on the details listed above, I propose the following:

1. Design of an output schema such that it provides structure, with flexibility, to achieve the above use cases
1. Implementation of utilities that can adapt data gathered from other code quality tools into this schema
1. Schema validation, which ensures tasks return the correctly structured data
1. Provide utilities to reformat the data into a file-based schema to accommodate those use cases.

## Proposed aggregate data schema

The below example illustrates the complete proposed data schema for JSON output. Following the example, each of the composite parts will be explained.

### TypeScript Interfaces

```ts
export type IndexableObject = { [key: string]: any };

export type DataSummary = {
  values: Record<string, number>;
  dataKey: string;
  total: number;
};

export interface TaskResult {
  info: {
    taskName: TaskName;
    taskDisplayName: string;
    category: string;
    group?: string;
  };
  result: Record<string, any>;
}

interface BaseResult {
  key: string;
  type: string;
  data: Array<string | IndexableObject>;
}

export interface SummaryResult extends BaseResult {
  type: 'summary';
  count: number;
}

export interface MultiValueResult extends BaseResult {
  type: 'multi-value';
  dataSummary: DataSummary;
}

export interface LookupValueResult extends BaseResult {
  type: 'lookup-value';
  dataSummary: {
    values: Record<string, number>;
    dataKey: string;
    valueKey: string;
    total: number;
  };
}

type Result = SummaryResult | MultiValueResult | LookupValueResult;

export interface CheckupResult {
  info: {
    project: {
      name: string;
      version: string;
      repository: {
        totalCommits: number;
        totalFiles: number;
        age: string;
        activeDays: string;
      };
    };
    cli: {
      schema: number;
      configHash: string;
      config: CheckupConfig;
      version: string;
      flags: {
        paths: [];
      };
    };
    analyzedFilesCount: number;
  };
  results: TaskResult[];
  errors: TaskError[];
  actions: Action[];
}
```

### Checkup Result Interface

#### `info` property

Contains information related to the project that was run.

#### `results` property

Contains a list of the task results from each task.

#### `errors` property

Contains a list of errors generated from tasks during invocation.

#### `actions` property

Contains any actions triggered as a result of running a task.

## `info` property schema

<details>
  <summary>Schema fragment</summary>

```ts
export interface CheckupResult {
  info: {
    project: {
      name: string;
      version: string;
      repository: {
        totalCommits: number;
        totalFiles: number;
        age: string;
        activeDays: string;
      };
    };
    cli: {
      schema: number;
      configHash: string;
      config: CheckupConfig;
      version: string;
      flags: {
        paths: [];
      };
    };
    analyzedFilesCount: number;
  };
  results: TaskResult[];
  errors: TaskError[];
  actions: Action[];
}
```

</details>

<details>
  <summary>Example</summary>

```json
{
  "info": {
    "project": {
      "name": "travis",
      "version": "0.0.1",
      "repository": {
        "totalCommits": 5869,
        "totalFiles": 1537,
        "age": "8 years",
        "activeDays": "1379"
      },
      "analyzedFilesCount": 1538
    },
    "cli": {
      "schema": 1,
      "configHash": "73249aa52d0783c15cd068a47a808543",
      "config": "",
      "version": "0.2.2"
      flags: {
        paths: ['*.js']
      }
    }
  }
}
```

</details>

## `results` property schema

<details>
  <summary>Schema fragment</summary>

```ts
export interface CheckupResult {
  //...

  results: TaskResult[];

  // ...
}
```

</details>

Results contain one or more result objects, which represent the data Checkup has gathered. At their core, they contain a simpler interface:

```typescript
interface BaseResult {
  key: string;
  type: string;
  data: Array<string | IndexableObject>;
}
```

- `key` - represents a unique identifier
- `type` - the result type ('summary', 'multi-value', 'lookup-value')
- `data` - the data Checkup has gathered and will ultimately operate on

All result types contain these core properties, but each add additional data to give different shapes to that data, depending on need.

### Summary results

Summary task results (tasks that have a `category` of `metrics`) are strictly informational in nature. They count a particular entity, such as a type, and display that value. Their main purpose is to help give a summary of the _shape_ and _size_ of a project, and to allow you to chart the changes to that _shape_ and _size_ over time.

The summary results have a simple integer value for the `count` property equivalent to the following:

```ts
export interface SummaryResult extends BaseResult {
  type: 'summary';
  count: number;
}
```

<details>
  <summary>Summary example</summary>

```json
"results": [
  {
    "info": {
      "taskName": "ember-types",
      "friendlyTaskName": "Ember Types",
      "taskClassification": {
        "category": "metrics",
        "group": "ember"
      }
    },
    "result": [
      {
        "key": "components",
        "count": 11,
        "data": [
          "app/components/account-token.js",
          "app/components/active-repo-count.js",
          "app/components/add-cron-job.js",
          "app/components/add-env-var.js",
          "app/components/add-ssh-key.js",
          "app/components/annotated-yaml.js",
          "app/components/beta-feature.js",
          "app/components/billing-summary-status.js",
          "app/components/billing/account.js",
          "app/components/billing/address.js",
          "app/components/billing/authorization.js"
        ]
      },
      {
        "key": "services",
        "count": 9,
        "data": [
          "app/services/accounts.js",
          "app/services/ajax.js",
          "app/services/animation.js",
          "app/services/api.js",
          "app/services/app-loading.js",
          "app/services/auth.js",
          "app/services/broadcasts.js",
          "app/services/external-links.js",
          "app/services/feature-flags.js"
        ]
      },
      {
        "key": "mixins",
        "count": 9,
        "data": [
          "app/mixins/branch-searching.js",
          "app/mixins/build-favicon.js",
          "app/mixins/builds/load-more.js",
          "app/mixins/components/form-select.js",
          "app/mixins/components/with-config-validation.js",
          "app/mixins/controller/account-repositories.js",
          "app/mixins/controller/billing.js",
          "app/mixins/duration-attributes.js",
          "app/mixins/duration-calculations.js",
          "app/mixins/polling.js"
        ]
      }
    ]
  }
]
```

</details>

### Multi value results

Multi value results break down the data in specific ways. These are used to determine ways to sort the data in order to derive insights.

Multi value results contain a bit more information than summary results.

```ts
{
  dataSummary: {
    values: Record<string, number>;
    dataKey: string;
    total: number;
  };
}
```

- `dataSummary.values` - a dictionary of values used to subdivide and count specific pieces of data
- `dataSummary.dataKey` - a string representing a property name that is the source of the aggregate data
- `dataSummary.total`- the sum of `data` of the parent result

The dataSummary can be used to calculate percentages:

```ts
let completed = Object.values(dataSummary.values).reduce(
  (total, value) => total + value,
  0
);

let dataSummary = Math.round((completed / dataSummary.total) * 100);
```

The `dataSummary` contains a `dataKey` configuration value, which allows you to specify a key corresponding to the
source of the values in the `values` dictionary. The `dataKey` is used to look up a property on the data, and the presence of unique values are used to count the total occurrences of that value.

<details>
  <summary>Multi value example</summary>

In the example below, the `ruleId` `dataKey` is used to lookup the values referenced by the `ruleId` property in the data, and the occurrences of those unique values are summed and used as the value.

```json
"results": [
  {
    "info": {
      "taskName": "eslint-summary",
      "taskDisplayName": "Eslint Summary",
      "category": "linting"
    },
    "result": [
      {
        "key": "eslint-errors",
        "type": "multi-value",
        "dataSummary": {
          "values": {
            "no-prototype-builtins": 20,
            "no-redeclare": 3
          },
          "dataKey": "ruleId",
          "total": 23
        },
        "data": [
          {
            "filePath": "/app/adapters/preference.js",
            "ruleId": "no-prototype-builtins",
            "message": "Do not access Object.prototype method 'hasOwnProperty' from target object.",
            "severity": 2,
            "line": 17,
            "column": 31
          },
          {
            "filePath": "/app/components/x-tracer.js",
            "ruleId": "no-redeclare",
            "message": "'window' is already defined as a built-in global variable.",
            "severity": 2,
            "line": 1,
            "column": 25
          },
          //...
        ]
      }
    ]
  }
]
```

</details>

### Lookup value results

Lookup value results are similar to multi value results, in that they summarize the data using a `dataSummary` property. Also similar to multi value results, they employ a `dataKey` to lookup the values of the properties in the data that the `dataKey` references.

In addition to the `dataKey`, they additionally have a `valueKey`. While the `dataKey` values ultimately form the keys in the `values` dictionary, the `valueKey`'s values are summed to form the numeric value of those data keys.

<details>
  <summary>Lookup value example</summary>

In the example below, the `extension` `dataKey` is used to lookup the values referenced by the `extension` property in the data. The `lines` `valueKey` is used to summarize the values.

```json
"results": [
  {
    "info": {
      "taskName": "lines-of-code",
      "taskDisplayName": "Lines of Code",
      "category": "metrics"
    },
    "result": [
      {
        "key": "lines of code",
        "type": "lookup-value",
        "dataSummary": {
          "values": {
            "js": 44939,
            "html": 201,
            "scss": 14355,
            "hbs": 10803,
            "svg": 22836,
            "rb": 608
          },
          "dataKey": "extension",
          "valueKey": "lines",
          "total": 93742
        },
        "data": [
          {
            "filePath": "/.eslintrc.js",
            "extension": "js",
            "lines": 359
          },
          {
            "filePath": "/.template-lintrc.js",
            "extension": "js",
            "lines": 5
          },
          //...
        ]
      }
    ]
  }
]
```

</details>

## `errors` property schema

<details>
  <summary>Schema fragment</summary>

```json
export interface CheckupResult {
  // ...

  errors: TaskError[];

  // ...
}
```

</details>

<details>
  <summary>Example</summary>

```json
"errors": [
  { "taskName": "lines-of-code", "error": "Uncaught TypeError: Cannot read property." }
],
```

</details>

## `actions` property schema

<details>
  <summary>Schema fragment</summary>

```ts
export interface CheckupResult {
  // ...

  actions: Action[];
}
```

</details>

<details>
  <summary>Example</summary>

```json
"actions": [
    {
      "name": "reduce-outdated-major-dependencies",
      "summary": "Update outdated major versions",
      "details": "25 major versions outdated",
      "defaultThreshold": 0.05,
      "items": [],
      "input": 0.21739130434782608
    },
    {
      "name": "reduce-outdated-minor-dependencies",
      "summary": "Update outdated minor versions",
      "details": "21 minor versions outdated",
      "defaultThreshold": 0.05,
      "items": [],
      "input": 0.1826086956521739
    }
  ]
```

</details>

## Associating data to UI

The following wireframe of the proposed HTML reporter demonstrates the usage of the above schema in a practical example.

<img width="800" alt="Checkup" src="../images/checkup-html-reporter.png">

It's almost certain that more detail will emerge that will require slight adjustments to the schema. This is expected, and anticipated.

## Adapting data for other uses

The above schema is primarily focused on addressing the [two major use cases](#motivation). While the primary goal of Checkup is to display aggregate information gathered from the configured tasks, the secondary use cases that require grouping and annotation at the file level can still be supported.

Transforming the data from the task-based form to the file-based form will require a utility, and that should be written and published by Checkup for use by consumers.

# Unresolved questions

[unresolved]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
