- Feature Name: Migrating Result Schema to SARIF 
- Start Date: 2020-09-01
- RFC PR:
- Checkup Issue: 

# Summary
[summary]: #summary

In [a previous RFC](./0001-task-result-schema.md), the need to standardize the output of Checkup's data was discussed. In this RFC, I propose leveraging the [SARIF](https://github.com/microsoft/sarif-tutorials) schema to shape the Checkup output. 

# Motivation
[motivation]: #motivation

Since the [Task Result Schema RFC](./0001-task-result-schema.md), changes have been implemented to the output that align with the schema described in that RFC, cleaning up the output significantly and adding shared helpers that help task authors format their data properly to fit the schema. Though defining a refined schema was a useful step in structuring the output, it didn't leverage any standardized schema for static analysis, such as the SARIF format from Microsoft.
SARIF is a well-established schema that "defines a standard format for the output of static analysis tools." There are a multitude of tools and plugins available in the open-source community that leverage SARIF - aligning our data structure with this shared schema will allow the data produced by checkup to be consumed by data pipelines that are already consuming SARIF data. For example, there is a [VSCode plugin](https://github.com/microsoft/sarif-vscode-extension) that prettyprints SARIF data, a [react based visualization tool](https://github.com/Microsoft/sarif-web-component), and a package that [converts eslint data to SARIF](https://www.npmjs.com/package/%40microsoft%2Feslint-formatter-sarif

# Details
[details]: #details

This migration will be pretty straightforward for the following reasons:
- SARIF schema isn't too far off from the schema we have already implemented to return result data. 
- The creation utilties used by Tasks to create CheckupResult data are abstracted cleanly from the Tasks themselves, so we can make this change without having to really touch the Tasks much at all. 

Note: some fields we are using in SARIF are nested deeper than the existing schema we have, yielding a 7-8% increase in number of lines of JSON required to describe the same results.

<details>
  <summary>Current types for structuring result data</summary>

  ```
  export type IndexableObject = { [key: string]: any };

  export type DataSummary = {
    values: Record<string, number>;
    dataKey: string;
    total: number;
    units?: string;
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

  export type Result = SummaryResult | MultiValueResult | LookupValueResult;

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
        args: {
          paths: string[];
        };
        flags: {
          config?: string;
          task?: string[];
          format: string;
          outputFile?: string;
          excludePaths?: string[];
        };
        timings: Record<string, number>;
      };
      analyzedFiles: string[];
      analyzedFilesCount: number;
    };
    results: TaskResult[];
    errors: TaskError[];
    actions: Action[];
  }
  ```
</details>

With SARIF, the results would look like this (there are a ton of optional properties here, so I am keeping only the ones we will need): 
*Note* I left in the comments from SARIF because I think they are useful to understand the shape of the data here, so when I wanted to add 
checkup related commentary I added a newline after the existing SARIF comments and prefaced with `CHECKUP:` before adding checkup commentary to differentiate.

<details>
  <summary>Relevant SARIF data fields</summary>

  ```
  export interface Log {
      /**
      * The URI of the JSON schema corresponding to the version.
      */
      $schema?: string;

      /**
      * The SARIF format version of this log file.
      */
      version: Log.version;

      /**
      * The set of runs contained in this log file. 
      * 
      * CHECKUP: this will generally be just one Run
      */
      runs: Run[];

      /**
      * Key/value pairs that provide additional information about the physical location.
      *
      * CHECKUP: we can store the project info, analyzedFiles, actions, and analyzedFilesCount here
      *
      */
      properties?: PropertyBag;

  }

  export interface Run {
      /**
      * Describes the invocation of the analysis tool. 
      *
      * CHECKUP: we will put cli info here
      */
      invocations?: Invocation[];

      
      /**
      * The set of results contained in an SARIF log. The results array can be omitted when a run is solely exporting
      * rules metadata. It must be present (but may be empty) if a log file represents an actual scan.
      */
      results?: Result[];

      /**
      * Information about the tool or tool pipeline that generated the results in this run. A run can only contain
      * results produced by a single tool or tool pipeline. A run can aggregate results from multiple log files, as long
      * as context around the tool run (tool command-line arguments and the like) is identical for all aggregated files.
      *
      * CHECKUP: We may not _need_ to populate this, but I don't see why not :) There is a use case for products using this schema for other 
      * data being aggregated with checkup data that is generated from a different tool, so it seems useful to fill this out to make it easy to differentiate 
      */
      tool: Tool;
  }

  /**
  * The analysis tool that was run.
  */
  export interface Tool {
      /**
      * The analysis tool that was run.
      */
      driver: ToolComponent;

      /**
      * Tool extensions that contributed to or reconfigured the analysis tool that was run.
      */
      extensions?: ToolComponent[];

      /**
      * Key/value pairs that provide additional information about the tool.
      */
      properties?: PropertyBag;
  }

  /**
  * A component, such as a plug-in or the driver, of the analysis tool that was run.
  */
  export interface ToolComponent {

      /**
      * The kinds of data contained in this object.
      */
      contents?: ToolComponent.contents[];

      /**
      * A comprehensive description of the tool component.
      */
      fullDescription?: MultiformatMessageString;

      /**
      * The name of the tool component along with its version and any other useful identifying information, such as its
      * locale.
      */
      fullName?: string;

      /**
      * A unique identifer for the tool component in the form of a GUID.
      */
      guid?: string;

      /**
      * The minimum value of localizedDataSemanticVersion required in translations consumed by this component; used by
      * components that consume translations.
      */
      minimumRequiredLocalizedDataSemanticVersion?: string;

      /**
      * The name of the tool component.
      */
      name: string;

      /**
      * An array of reportingDescriptor objects relevant to the notifications related to the configuration and runtime
      * execution of the tool component.
      */
      notifications?: ReportingDescriptor[];

      /**
      * The organization or company that produced the tool component.
      */
      organization?: string;

      /**
      * A product suite to which the tool component belongs.
      */
      product?: string;

      /**
      * A localizable string containing the name of the suite of products to which the tool component belongs.
      */
      productSuite?: string;

      /**
      * A string specifying the UTC date (and optionally, the time) of the component's release.
      */
      releaseDateUtc?: string;

      /**
      * An array of reportingDescriptor objects relevant to the analysis performed by the tool component.
      */
      rules?: ReportingDescriptor[];

      /**
      * The tool component version in the format specified by Semantic Versioning 2.0.
      */
      semanticVersion?: string;

      /**
      * A brief description of the tool component.
      */
      shortDescription?: MultiformatMessageString;

      /**
      * The tool component version, in whatever format the component natively provides.
      */
      version?: string;

      /**
      * Key/value pairs that provide additional information about the tool component.
      */
      properties?: PropertyBag;
  }

  /**
  * The runtime environment of the analysis tool run.
  */
  export interface Invocation {
      /**
      * An array of strings, containing in order the command line arguments passed to the tool from the operating
      * system.
      *
      * CHECKUP: args go heree 
      */
      arguments?: string[];

      /**
      * The command line used to invoke the tool.
      */
      commandLine?: string;

      /**
      * The Coordinated Universal Time (UTC) date and time at which the run ended. See "Date/time properties" in the
      * SARIF spec for the required format.
      * 
      * CHECKUP: this property along with startTimeUtc will contain timing info
      */
      endTimeUtc?: string;

      /**
      * The environment variables associated with the analysis tool process, expressed as key/value pairs.
      * 
      * CHECKUP: this property will contain the flags, config, configHash 
      */
      environmentVariables?: { [key: string]: string };

      /**
      * Specifies whether the tool's execution completed successfully.
      */
      executionSuccessful: boolean;

      /**
      * The process exit code.
      */
      exitCode?: number;

      /**
      * The machine that hosted the analysis tool run.
      */
      machine?: string;

      /**
      * The Coordinated Universal Time (UTC) date and time at which the run started. See "Date/time properties" in the
      * SARIF spec for the required format.
      */
      startTimeUtc?: string;

      /**
      * A list of runtime conditions detected by the tool during the analysis.
      * 
      * CHECKUP: this is where we will store nonterminal errors collected from Tasks in checkup
      */
      toolExecutionNotifications?: Notification[];


  }

  export namespace Result {
      type kind =
          "notApplicable" |
          "pass" |
          "fail" |
          "review" |
          "open" |
          "informational";

      type level =
          "none" |
          "note" |
          "warning" |
          "error";
  }

  /**
  * A result produced by an analysis tool.
  */
  export interface Result {
      /**
      * An array of 'fix' objects, each of which represents a proposed fix to the problem indicated by the result.
      * 
      */
      fixes?: Fix[];

      /**
      * A stable, unique identifer for the result in the form of a GUID.
      */
      guid?: string;
      
      /**
      * A value that categorizes results by evaluation state.
      */
      kind?: Result.kind;

      /**
      * A value specifying the severity level of the result.
      */
      level?: Result.level;

      /**
      * A message that describes the result. The first sentence of the message only will be displayed when visible space
      * is limited.
      *
      * CHECKUP: if we wanted to keep concepts of Result type `SummaryResult | MultiValueResult | LookupValueResult;`, we can store those here, as well as the dataSummary and data for each Result
      */
      message: Message;

      /**
      * A positive integer specifying the number of times this logically unique result was observed in this run.
      */
      occurrenceCount?: number;

      /**
      * A number representing the priority or importance of the result.
      */
      rank?: number;

      /**
      * The stable, unique identifier of the rule, if any, to which this notification is relevant. This member can be
      * used to retrieve rule metadata from the rules dictionary, if it exists.
      */
      ruleId?: string;

      /**
      * The index within the tool component rules array of the rule object associated with this result.
      */
      ruleIndex?: number;

      /**
      * Key/value pairs that provide additional information about the result.
      * 
      */
      properties?: PropertyBag;
  }

  /**
  * Encapsulates a message intended to be read by the end user.
  */
  export interface Message {
      /**
      * An array of strings to substitute into the message string.
      */
      arguments?: string[];

      /**
      * The identifier for this message.
      */
      id?: string;

      /**
      * A Markdown message string.
      */
      markdown?: string;

      /**
      * A plain text message string.
      */
      text?: string;

      /**
      * Key/value pairs that provide additional information about the message.
      */
      properties?: PropertyBag;
  }

  /**
  * A proposed fix for the problem represented by a result object. A fix specifies a set of artifacts to modify. For
  * each artifact, it specifies a set of bytes to remove, and provides a set of new bytes to replace them.
  */
  export interface Fix {
      /**
      * One or more artifact changes that comprise a fix for a result.
      */
      artifactChanges: ArtifactChange[];

      /**
      * A message that describes the proposed fix, enabling viewers to present the proposed change to an end user.
      */
      description?: Message;

      /**
      * Key/value pairs that provide additional information about the fix.
      */
      properties?: PropertyBag;
  }
  ```
</details>

Note: there are type definitions for SARIF [here](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/289ba098a611b80ee03ed72a2feee41a593fe765/types/sarif/index.d.ts). These types are being used by a number of the high profile tools I linked above, so I view them as very reliable, even though they are third party.

Here is a full JSON output from running checkup _using existing schema_. I have removed most of the lists of file names to keep it clean, but it was 24550 lines of JSON.
<details>

  <summary>Existing checkup output</summary>

```
{
  "info": {
    "project": {
      "name": "travis",
      "version": "0.0.1",
      "repository": {
        "totalCommits": 5870,
        "totalFiles": 1538,
        "age": "8 years",
        "activeDays": "1380"
      }
    },
    "cli": {
      "configHash": "1913a68bff82a711d2a1395e25d6992e",
      "config": {
        "plugins": [
          "checkup-plugin-ember",
          "checkup-plugin-ember-octane",
          "checkup-plugin-javascript"
        ],
        "tasks": {
          "ember/ember-test-types": [
            "on",
            {
              "actions": {
                "ratioApplicationTests": {
                  "threshold": 3
                }
              }
            }
          ]
        }
      },
      "version": "0.7.4",
      "schema": 1,
      "args": {
        "paths": []
      },
      "flags": {
        "format": "json",
        "outputFile": ""
      },
      "timings": {
        "ember/ember-types": 16.03910597,
        "ember/ember-in-repo-addons-engines": 16.021184852,
        "ember/ember-dependencies": 16.01988663,
        "javascript/eslint-summary": 14.159414049,
        "ember/ember-test-types": 16.034998766,
        "ember-octane/ember-octane-migration-status": 6.50991697,
        "ember/ember-template-lint-disables": 18.496345184,
        "javascript/eslint-disables": 11.405058044,
        "meta/lines-of-code": 20.990534104,
        "javascript/outdated-dependencies": 28.323344796
      }
    },
    "analyzedFiles": [],
    "analyzedFilesCount": 1539
  },
  "results": [
    {
      "info": {
        "taskName": "ember-types",
        "taskDisplayName": "Ember Types",
        "category": "metrics",
        "group": "ember"
      },
      "result": [
        {
          "key": "components",
          "type": "summary",
          "data": [],
          "count": 219
        },
        {
          "key": "controllers",
          "type": "summary",
          "data": [],
          "count": 41
        },
        {
          "key": "helpers",
          "type": "summary",
          "data": [],
          "count": 35
        },
        {
          "key": "initializers",
          "type": "summary",
          "data": [],
          "count": 4
        },
        {
          "key": "instance-initializers",
          "type": "summary",
          "data": [],
          "count": 6
        },
        {
          "key": "mixins",
          "type": "summary",
          "data": [],
          "count": 16
        },
        {
          "key": "models",
          "type": "summary",
          "data": [],
          "count": 63
        },
        {
          "key": "routes",
          "type": "summary",
          "data": [],
          "count": 52
        },
        {
          "key": "services",
          "type": "summary",
          "data": [],
          "count": 45
        },
        {
          "key": "templates",
          "type": "summary",
          "data": [],
          "count": 238
        }
      ]
    },
    {
      "info": {
        "taskName": "ember-in-repo-addons-engines",
        "taskDisplayName": "Ember In-Repo Addons / Engines",
        "category": "metrics",
        "group": "ember"
      },
      "result": [
        {
          "key": "in-repo engines",
          "type": "summary",
          "data": [],
          "count": 0
        },
        {
          "key": "in-repo addons",
          "type": "summary",
          "data": [],
          "count": 0
        }
      ]
    },
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
          "data": [],
        }
      ]
    },
    {
      "info": {
        "taskName": "ember-dependencies",
        "taskDisplayName": "Ember Dependencies",
        "category": "dependencies",
        "group": "ember"
      },
      "result": [
        {
          "key": "ember core libraries",
          "type": "summary",
          "data": [
            {
              "packageName": "ember-source",
              "version": "~3.12.0"
            },
            {
              "packageName": "ember-cli",
              "version": "~3.12.0"
            },
            {
              "packageName": "ember-data",
              "version": "~3.12.2"
            }
          ],
          "count": 3
        },
        {
          "key": "ember addon dependencies",
          "type": "summary",
          "data": [],
          "count": 0
        },
        {
          "key": "ember addon devDependencies",
          "type": "summary",
          "data": [],
          "count": 36
        },
        {
          "key": "ember-cli addon dependencies",
          "type": "summary",
          "data": [],
          "count": 0
        },
        {
          "key": "ember-cli addon devDependencies",
          "type": "summary",
          "data": [],
          "count": 24
        }
      ]
    },
    {
      "info": {
        "taskName": "outdated-dependencies",
        "taskDisplayName": "Outdated Dependencies",
        "category": "dependencies"
      },
      "result": [
        {
          "key": "dependencies",
          "type": "multi-value",
          "dataSummary": {
            "values": {
              "current": 70,
              "major": 28,
              "minor": 9,
              "nonSemver": 6,
              "patch": 2
            },
            "dataKey": "semverBump",
            "total": 115
          },
          "data": []
        }
      ]
    },
    {
      "info": {
        "taskName": "ember-template-lint-disables",
        "taskDisplayName": "Number of template-lint-disable Usages",
        "category": "linting",
        "group": "disabled-lint-rules"
      },
      "result": [
        {
          "key": "template-lint-disable usages",
          "type": "summary",
          "data": [],
          "count": 0
        }
      ]
    },
    {
      "info": {
        "taskName": "eslint-disables",
        "taskDisplayName": "Number of eslint-disable Usages",
        "category": "linting",
        "group": "disabled-lint-rules"
      },
      "result": [
        {
          "key": "eslint-disable usages",
          "type": "summary",
          "data": [],
          "count": 32
        }
      ]
    },
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
              "no-redeclare": 3,
              "no-setter-return": 1
            },
            "dataKey": "ruleId",
            "total": 24
          },
          "data": [],
        },
        {
          "key": "eslint-warnings",
          "type": "multi-value",
          "dataSummary": {
            "values": {
              "": 3
            },
            "dataKey": "ruleId",
            "total": 3
          },
          "data": []
        }
      ]
    },
    {
      "info": {
        "taskName": "ember-test-types",
        "taskDisplayName": "Test Types",
        "category": "testing",
        "group": "ember"
      },
      "result": [
        {
          "key": "unit",
          "type": "multi-value",
          "dataSummary": {
            "values": {
              "test": 163,
              "skip": 0,
              "only": 0,
              "todo": 0
            },
            "dataKey": "method",
            "total": 163
          },
          "data": []
        },
        {
          "key": "rendering",
          "type": "multi-value",
          "dataSummary": {
            "values": {
              "test": 153,
              "skip": 0,
              "only": 0,
              "todo": 0
            },
            "dataKey": "method",
            "total": 153
          },
          "data": []
        },
        {
          "key": "application",
          "type": "multi-value",
          "dataSummary": {
            "values": {
              "test": 265,
              "skip": 5,
              "only": 0,
              "todo": 0
            },
            "dataKey": "method",
            "total": 270
          },
          "data": []
        }
      ]
    },
    {
      "info": {
        "taskName": "ember-octane-migration-status",
        "taskDisplayName": "Ember Octane Migration Status",
        "category": "migrations",
        "group": "ember"
      },
      "result": [
        {
          "key": "Native Classes",
          "type": "multi-value",
          "dataSummary": {
            "values": {
              "ember/no-classic-classes": 290,
              "ember/classic-decorator-no-classic-methods": 0,
              "ember/no-actions-hash": 63,
              "ember/no-get": 11,
              "ember/no-mixins": 32
            },
            "dataKey": "ruleId",
            "total": 396
          },
          "data": []
        },
        {
          "key": "Tagless Components",
          "type": "multi-value",
          "dataSummary": {
            "values": {
              "ember/require-tagless-components": 142
            },
            "dataKey": "ruleId",
            "total": 142
          },
          "data": []
        },
        {
          "key": "Glimmer Components",
          "type": "multi-value",
          "dataSummary": {
            "values": {
              "ember/no-classic-components": 155
            },
            "dataKey": "ruleId",
            "total": 155
          },
          "data": []
        },
        {
          "key": "Tracked Propeties",
          "type": "multi-value",
          "dataSummary": {
            "values": {
              "ember/no-computed-properties-in-native-classes": 0
            },
            "dataKey": "ruleId",
            "total": 0
          },
          "data": []
        },
        {
          "key": "Angle Brackets Syntax",
          "type": "multi-value",
          "dataSummary": {
            "values": {
              "no-curly-component-invocation": 2
            },
            "dataKey": "ruleId",
            "total": 2
          },
          "data": [
            {
              "filePath": "/app/templates/components/billing/select-plan.hbs",
              "ruleId": "no-curly-component-invocation",
              "message": "You are using the component {{selectedPlan.builds}} with curly component syntax. You should use <SelectedPlan.Builds> instead. If it is actually a helper you must manually add it to the 'no-curly-component-invocation' rule configuration, e.g. `'no-curly-component-invocation': { allow: ['selectedPlan.builds'] }`.",
              "severity": 2,
              "line": 68,
              "column": 49
            },
            {
              "filePath": "/app/templates/components/forms/form-switch.hbs",
              "ruleId": "no-curly-component-invocation",
              "message": "You are using the component {{offText}} with curly component syntax. You should use <OffText> instead. If it is actually a helper you must manually add it to the 'no-curly-component-invocation' rule configuration, e.g. `'no-curly-component-invocation': { allow: ['offText'] }`.",
              "severity": 2,
              "line": 11,
              "column": 12
            }
          ]
        },
        {
          "key": "Named Arguments",
          "type": "multi-value",
          "dataSummary": {
            "values": {
              "no-args-paths": 0
            },
            "dataKey": "ruleId",
            "total": 0
          },
          "data": []
        },
        {
          "key": "Own Properties",
          "type": "multi-value",
          "dataSummary": {
            "values": {
              "no-implicit-this": 55
            },
            "dataKey": "ruleId",
            "total": 55
          },
          "data": []
        },
        {
          "key": "Modifiers",
          "type": "multi-value",
          "dataSummary": {
            "values": {
              "no-action": 237
            },
            "dataKey": "ruleId",
            "total": 237
          },
          "data": []
        }
      ]
    }
  ],
  "errors": [],
  "actions": [
    {
      "name": "reduce-eslint-disable-usages",
      "summary": "Reduce number of eslint-disable usages",
      "details": "32 usages of eslint-disable",
      "defaultThreshold": 2,
      "items": [
        "Total eslint-disable usages: 32"
      ],
      "input": 32
    },
    {
      "name": "reduce-eslint-errors",
      "summary": "Reduce number of eslint errors",
      "details": "24 total errors",
      "defaultThreshold": 20,
      "items": [
        "Total eslint errors: 24"
      ],
      "input": 24
    },
    {
      "name": "reduce-outdated-major-dependencies",
      "summary": "Update outdated major versions",
      "details": "28 major versions outdated",
      "defaultThreshold": 0.05,
      "items": [],
      "input": 0.24347826086956523
    },
    {
      "name": "reduce-outdated-minor-dependencies",
      "summary": "Update outdated minor versions",
      "details": "9 minor versions outdated",
      "defaultThreshold": 0.05,
      "items": [],
      "input": 0.0782608695652174
    },
    {
      "name": "reduce-outdated-dependencies",
      "summary": "Update outdated versions",
      "details": "34% of versions outdated",
      "defaultThreshold": 0.2,
      "items": [],
      "input": 0.3391304347826087
    }
  ]
}
```
</details>

Here is a full JSON output from running checkup _using SARIF_. I have removed most of the lists of file names to keep it clean, but it was 26487 lines of JSON.
<details>

  <summary>SARIF output</summary>

  ```
{
  "version": "2.1.0",
  "properties": {
    "project": {
      "name": "travis",
      "version": "0.0.1",
      "repository": {
        "totalCommits": 5870,
        "totalFiles": 1538,
        "age": "8 years",
        "activeDays": "1380",
        "linesOfCode": {
          "types": [
            {
              "total": 44939,
              "extension": "js"
            },
            {
              "total": 201,
              "extension": "html"
            },
            {
              "total": 14355,
              "extension": "scss"
            },
            {
              "total": 10803,
              "extension": "hbs"
            },
            {
              "total": 22836,
              "extension": "svg"
            },
            {
              "total": 608,
              "extension": "rb"
            }
          ],
          "total": 93742
        }
      }
    },
    "cli": {
      "configHash": "1913a68bff82a711d2a1395e25d6992e",
      "config": {
        "plugins": [
          "checkup-plugin-ember",
          "checkup-plugin-ember-octane",
          "checkup-plugin-javascript"
        ],
        "tasks": {
          "ember/ember-test-types": [
            "on",
            {
              "actions": {
                "ratioApplicationTests": {
                  "threshold": 3
                }
              }
            }
          ]
        }
      },
      "version": "0.7.4",
      "schema": 1,
      "args": {
        "paths": []
      },
      "flags": {
        "format": "json",
        "outputFile": ""
      },
      "timings": {
        "ember/ember-types": 23.371086841,
        "ember/ember-in-repo-addons-engines": 23.329550989,
        "ember/ember-dependencies": 23.327852732,
        "javascript/eslint-summary": 20.294677398,
        "ember/ember-test-types": 23.346301169,
        "ember-octane/ember-octane-migration-status": 8.863200936,
        "ember/ember-template-lint-disables": 27.640971204,
        "javascript/eslint-disables": 17.036571088,
        "javascript/outdated-dependencies": 43.921809849
      }
    },
    "analyzedFiles": [],
    "analyzedFilesCount": 1539,
    "actions": [
      {
        "name": "reduce-eslint-disable-usages",
        "summary": "Reduce number of eslint-disable usages",
        "details": "32 usages of eslint-disable",
        "defaultThreshold": 2,
        "items": [
          "Total eslint-disable usages: 32"
        ],
        "input": 32
      },
      {
        "name": "reduce-outdated-major-dependencies",
        "summary": "Update outdated major versions",
        "details": "28 major versions outdated",
        "defaultThreshold": 0.05,
        "items": [],
        "input": 0.24347826086956523
      },
      {
        "name": "reduce-outdated-minor-dependencies",
        "summary": "Update outdated minor versions",
        "details": "9 minor versions outdated",
        "defaultThreshold": 0.05,
        "items": [],
        "input": 0.0782608695652174
      },
      {
        "name": "reduce-outdated-dependencies",
        "summary": "Update outdated versions",
        "details": "32% of versions outdated",
        "defaultThreshold": 0.2,
        "items": [],
        "input": 0.3217391304347826
      }
    ]
  },
  "runs": [
    {
      "results": [
        {
          "message": {
            "text": "components"
          },
          "locations": [],
          "occurrenceCount": 219,
          "properties": {
            "taskName": "ember-types",
            "taskDisplayName": "Ember Types",
            "category": "metrics",
            "group": "ember"
          }
        },
        {
          "message": {
            "text": "controllers"
          },
          "locations": [],
          "occurrenceCount": 41,
          "properties": {
            "taskName": "ember-types",
            "taskDisplayName": "Ember Types",
            "category": "metrics",
            "group": "ember"
          }
        },
        {
          "message": {
            "text": "helpers"
          },
          "locations": [],
          "occurrenceCount": 35,
          "properties": {
            "taskName": "ember-types",
            "taskDisplayName": "Ember Types",
            "category": "metrics",
            "group": "ember"
          }
        },
        {
          "message": {
            "text": "initializers"
          },
          "locations": [
            {
              "physicalLocation": {
                "artifactLocation": {
                  "uri": "/app/initializers/app.js"
                }
              }
            },
            {
              "physicalLocation": {
                "artifactLocation": {
                  "uri": "/app/initializers/configure-inflector.js"
                }
              }
            },
            {
              "physicalLocation": {
                "artifactLocation": {
                  "uri": "/app/initializers/services.js"
                }
              }
            },
            {
              "physicalLocation": {
                "artifactLocation": {
                  "uri": "/app/initializers/tracer.js"
                }
              }
            }
          ],
          "occurrenceCount": 4,
          "properties": {
            "taskName": "ember-types",
            "taskDisplayName": "Ember Types",
            "category": "metrics",
            "group": "ember"
          }
        },
        {
          "message": {
            "text": "instance-initializers"
          },
          "locations": [],
          "occurrenceCount": 6,
          "properties": {
            "taskName": "ember-types",
            "taskDisplayName": "Ember Types",
            "category": "metrics",
            "group": "ember"
          }
        },
        {
          "message": {
            "text": "mixins"
          },
          "locations": [],
          "occurrenceCount": 16,
          "properties": {
            "taskName": "ember-types",
            "taskDisplayName": "Ember Types",
            "category": "metrics",
            "group": "ember"
          }
        },
        {
          "message": {
            "text": "models"
          },
          "locations": [],
          "occurrenceCount": 63,
          "properties": {
            "taskName": "ember-types",
            "taskDisplayName": "Ember Types",
            "category": "metrics",
            "group": "ember"
          }
        },
        {
          "message": {
            "text": "routes"
          },
          "locations": [],
          "occurrenceCount": 52,
          "properties": {
            "taskName": "ember-types",
            "taskDisplayName": "Ember Types",
            "category": "metrics",
            "group": "ember"
          }
        },
        {
          "message": {
            "text": "services"
          },
          "locations": [],
          "occurrenceCount": 45,
          "properties": {
            "taskName": "ember-types",
            "taskDisplayName": "Ember Types",
            "category": "metrics",
            "group": "ember"
          }
        },
        {
          "message": {
            "text": "templates"
          },
          "locations": [],
          "occurrenceCount": 238,
          "properties": {
            "taskName": "ember-types",
            "taskDisplayName": "Ember Types",
            "category": "metrics",
            "group": "ember"
          }
        },
        {
          "message": {
            "text": "in-repo addons"
          },
          "locations": [],
          "occurrenceCount": 0,
          "properties": {
            "taskName": "ember-in-repo-addons-engines",
            "taskDisplayName": "Ember In-Repo Addons / Engines",
            "category": "metrics",
            "group": "ember"
          }
        },
        {
          "message": {
            "text": "in-repo engines"
          },
          "locations": [],
          "occurrenceCount": 0,
          "properties": {
            "taskName": "ember-in-repo-addons-engines",
            "taskDisplayName": "Ember In-Repo Addons / Engines",
            "category": "metrics",
            "group": "ember"
          }
        },
        {
          "occurrenceCount": 3,
          "message": {
            "text": "ember core libraries"
          },
          "properties": {
            "dependencies": [
              {
                "packageName": "ember-source",
                "version": "~3.12.0"
              },
              {
                "packageName": "ember-cli",
                "version": "~3.12.0"
              },
              {
                "packageName": "ember-data",
                "version": "~3.12.2"
              }
            ],
            "taskName": "ember-dependencies",
            "taskDisplayName": "Ember Dependencies",
            "category": "dependencies",
            "group": "ember"
          }
        },
        {
          "occurrenceCount": 0,
          "message": {
            "text": "ember addon dependencies"
          },
          "properties": {
            "dependencies": [],
            "taskName": "ember-dependencies",
            "taskDisplayName": "Ember Dependencies",
            "category": "dependencies",
            "group": "ember"
          }
        },
        {
          "occurrenceCount": 36,
          "message": {
            "text": "ember addon devDependencies"
          },
          "properties": {
            "dependencies": [],
            "taskName": "ember-dependencies",
            "taskDisplayName": "Ember Dependencies",
            "category": "dependencies",
            "group": "ember"
          }
        },
        {
          "occurrenceCount": 0,
          "message": {
            "text": "ember-cli addon dependencies"
          },
          "properties": {
            "dependencies": [],
            "taskName": "ember-dependencies",
            "taskDisplayName": "Ember Dependencies",
            "category": "dependencies",
            "group": "ember"
          }
        },
        {
          "occurrenceCount": 24,
          "message": {
            "text": "ember-cli addon devDependencies"
          },
          "properties": {
            "dependencies": [
              {
                "packageName": "ember-cli",
                "version": "~3.12.0"
              },
              {
                "packageName": "ember-cli-app-version",
                "version": "^3.2.0"
              },
              {
                "packageName": "ember-cli-autoprefixer",
                "version": "^0.8.1"
              },
              {
                "packageName": "ember-cli-babel",
                "version": "^7.7.3"
              },
              {
                "packageName": "ember-cli-cjs-transform",
                "version": "^1.3.0"
              },
              {
                "packageName": "ember-cli-clipboard",
                "version": "^0.13.0"
              },
              {
                "packageName": "ember-cli-dependency-checker",
                "version": "^3.1.0"
              },
              {
                "packageName": "ember-cli-deploy",
                "version": "^1.0.2"
              },
              {
                "packageName": "ember-cli-deploy-lightning-pack",
                "version": "1.2.2"
              },
              {
                "packageName": "ember-cli-deprecation-workflow",
                "version": "^1.0.0"
              },
              {
                "packageName": "ember-cli-document-title-northm",
                "version": "^1.0.3"
              },
              {
                "packageName": "ember-cli-dotenv",
                "version": "^2.2.3"
              },
              {
                "packageName": "ember-cli-eslint",
                "version": "^5.1.0"
              },
              {
                "packageName": "ember-cli-head",
                "version": "^0.4.1"
              },
              {
                "packageName": "ember-cli-htmlbars",
                "version": "^4.0.1"
              },
              {
                "packageName": "ember-cli-inject-live-reload",
                "version": "^2.0.1"
              },
              {
                "packageName": "ember-cli-mirage",
                "version": "^1.1.0"
              },
              {
                "packageName": "ember-cli-moment-shim",
                "version": "^3.5.0"
              },
              {
                "packageName": "ember-cli-page-object",
                "version": "^1.15.0-beta.3"
              },
              {
                "packageName": "ember-cli-postcss",
                "version": "^4.2.1"
              },
              {
                "packageName": "ember-cli-sentry",
                "version": "^4.0.0"
              },
              {
                "packageName": "ember-cli-sri",
                "version": "^2.1.1"
              },
              {
                "packageName": "ember-cli-test-loader",
                "version": "^2.1.0"
              },
              {
                "packageName": "ember-cli-uglify",
                "version": "^3.0.0"
              }
            ],
            "taskName": "ember-dependencies",
            "taskDisplayName": "Ember Dependencies",
            "category": "dependencies",
            "group": "ember"
          }
        },
        {
          "occurrenceCount": 70,
          "message": {
            "text": "current"
          },
          "properties": {
            "dependencies": [],
            "taskName": "outdated-dependencies",
            "taskDisplayName": "Outdated Dependencies",
            "category": "dependencies"
          }
        },
        {
          "occurrenceCount": 28,
          "message": {
            "text": "major"
          },
          "properties": {
            "dependencies": [],
            "taskName": "outdated-dependencies",
            "taskDisplayName": "Outdated Dependencies",
            "category": "dependencies"
          }
        },
        {
          "occurrenceCount": 9,
          "message": {
            "text": "minor"
          },
          "properties": {
            "dependencies": [],
            "taskName": "outdated-dependencies",
            "taskDisplayName": "Outdated Dependencies",
            "category": "dependencies"
          }
        },
        {
          "occurrenceCount": 6,
          "message": {
            "text": "nonSemver"
          },
          "properties": {
            "dependencies": [],
            "taskName": "outdated-dependencies",
            "taskDisplayName": "Outdated Dependencies",
            "category": "dependencies"
          }
        },
        {
          "occurrenceCount": 2,
          "message": {
            "text": "patch"
          },
          "properties": {
            "dependencies": [
              {
                "packageName": "ember-tooltips",
                "packageJsonVersion": "^3.1.1",
                "homepage": "https://github.com/sir-dunxalot/ember-tooltips#readme",
                "latest": "3.4.5",
                "installed": "3.4.4",
                "wanted": "3.4.5",
                "semverBump": "patch"
              },
              {
                "packageName": "qunit",
                "homepage": "https://qunitjs.com",
                "latest": "2.11.3",
                "installed": "2.11.0",
                "wanted": null,
                "semverBump": "patch"
              }
            ],
            "taskName": "outdated-dependencies",
            "taskDisplayName": "Outdated Dependencies",
            "category": "dependencies"
          }
        },
        {
          "ruleId": 0,
          "message": {
            "text": 0
          },
          "locations": [],
          "occurrenceCount": 0,
          "properties": {
            "taskName": "ember-template-lint-disables",
            "taskDisplayName": "Number of template-lint-disable Usages",
            "category": "linting",
            "group": "disabled-lint-rules"
          }
        },
        {
          "ruleId": "no-eslint-disable",
          "message": {
            "text": "eslint-disable usages"
          },
          "locations": [],
          "occurrenceCount": 32,
          "properties": {
            "taskName": "eslint-disables",
            "taskDisplayName": "Number of eslint-disable Usages",
            "category": "linting",
            "group": "disabled-lint-rules"
          }
        },
        {
          "ruleId": "no-prototype-builtins",
          "message": {
            "text": "Do not access Object.prototype method 'hasOwnProperty' from target object."
          },
          "locations": [],
          "occurrenceCount": 20,
          "properties": {
            "type": "error",
            "taskName": "eslint-summary",
            "taskDisplayName": "Eslint Summary",
            "category": "linting"
          }
        },
        {
          "ruleId": "no-redeclare",
          "message": {
            "text": "'window' is already defined as a built-in global variable."
          },
          "locations": [],
          "occurrenceCount": 3,
          "properties": {
            "type": "error",
            "taskName": "eslint-summary",
            "taskDisplayName": "Eslint Summary",
            "category": "linting"
          }
        },
        {
          "ruleId": "no-setter-return",
          "message": {
            "text": "Setter cannot return a value."
          },
          "locations": [
            {
              "physicalLocation": {
                "artifactLocation": {
                  "uri": "/app/utils/log.js"
                },
                "region": {
                  "startLine": 430,
                  "startColumn": 7
                }
              }
            }
          ],
          "occurrenceCount": 1,
          "properties": {
            "type": "error",
            "taskName": "eslint-summary",
            "taskDisplayName": "Eslint Summary",
            "category": "linting"
          }
        },
        {
          "ruleId": "",
          "message": {
            "text": "File ignored because of a matching ignore pattern. Use \"--no-ignore\" to override."
          },
          "locations": [],
          "occurrenceCount": 3,
          "properties": {
            "type": "warning",
            "taskName": "eslint-summary",
            "taskDisplayName": "Eslint Summary",
            "category": "linting"
          }
        },
        {
          "ruleId": "test-types",
          "message": {
            "text": "application"
          },
          "locations": [],
          "occurrenceCount": 265,
          "properties": {
            "method": "test",
            "taskName": "ember-test-types",
            "taskDisplayName": "Test Types",
            "category": "testing",
            "group": "ember"
          }
        },
        {
          "ruleId": "test-types",
          "message": {
            "text": "application"
          },
          "locations": [],
          "occurrenceCount": 5,
          "properties": {
            "method": "skip",
            "taskName": "ember-test-types",
            "taskDisplayName": "Test Types",
            "category": "testing",
            "group": "ember"
          }
        },
        {
          "ruleId": "test-types",
          "message": {
            "text": "rendering"
          },
          "locations": [],
          "occurrenceCount": 153,
          "properties": {
            "method": "test",
            "taskName": "ember-test-types",
            "taskDisplayName": "Test Types",
            "category": "testing",
            "group": "ember"
          }
        },
        {
          "ruleId": "test-types",
          "message": {
            "text": "unit"
          },
          "locations": [],
          "occurrenceCount": 163,
          "properties": {
            "method": "test",
            "taskName": "ember-test-types",
            "taskDisplayName": "Test Types",
            "category": "testing",
            "group": "ember"
          }
        },
        {
          "ruleId": "ember/no-classic-classes",
          "message": {
            "text": "Native JS classes should be used instead of classic classes"
          },
          "locations": [],
          "occurrenceCount": 290,
          "properties": {
            "resultGroup": "Native Classes",
            "taskName": "ember-octane-migration-status",
            "taskDisplayName": "Ember Octane Migration Status",
            "category": "migrations",
            "group": "ember"
          }
        },
        {
          "ruleId": "ember/no-actions-hash",
          "message": {
            "text": "Use the @action decorator instead of declaring an actions hash"
          },
          "locations": [],
          "occurrenceCount": 63,
          "properties": {
            "resultGroup": "Native Classes",
            "taskName": "ember-octane-migration-status",
            "taskDisplayName": "Ember Octane Migration Status",
            "category": "migrations",
            "group": "ember"
          }
        },
        {
          "ruleId": "ember/no-mixins",
          "message": {
            "text": "Don't use a mixin"
          },
          "locations": [],
          "occurrenceCount": 32,
          "properties": {
            "resultGroup": "Native Classes",
            "taskName": "ember-octane-migration-status",
            "taskDisplayName": "Ember Octane Migration Status",
            "category": "migrations",
            "group": "ember"
          }
        },
        {
          "ruleId": "ember/no-get",
          "message": {
            "text": "Use `this.isExpanded` instead of `this.get('isExpanded')`"
          },
          "locations": [],
          "occurrenceCount": 11,
          "properties": {
            "resultGroup": "Native Classes",
            "taskName": "ember-octane-migration-status",
            "taskDisplayName": "Ember Octane Migration Status",
            "category": "migrations",
            "group": "ember"
          }
        },
        {
          "ruleId": "ember/require-tagless-components",
          "message": {
            "text": "Please switch to a tagless component by setting `tagName: ''` or converting to a Glimmer component"
          },
          "locations": [],
          "occurrenceCount": 142,
          "properties": {
            "resultGroup": "Tagless Components",
            "taskName": "ember-octane-migration-status",
            "taskDisplayName": "Ember Octane Migration Status",
            "category": "migrations",
            "group": "ember"
          }
        },
        {
          "ruleId": "ember/no-classic-components",
          "message": {
            "text": "Use Glimmer components(@glimmer/component) instead of classic components(@ember/component)"
          },
          "locations": [],
          "occurrenceCount": 155,
          "properties": {
            "resultGroup": "Glimmer Components",
            "taskName": "ember-octane-migration-status",
            "taskDisplayName": "Ember Octane Migration Status",
            "category": "migrations",
            "group": "ember"
          }
        },
        {
          "ruleId": "no-curly-component-invocation",
          "message": {
            "text": "You are using the component {{selectedPlan.builds}} with curly component syntax. You should use <SelectedPlan.Builds> instead. If it is actually a helper you must manually add it to the 'no-curly-component-invocation' rule configuration, e.g. `'no-curly-component-invocation': { allow: ['selectedPlan.builds'] }`."
          },
          "locations": [],
          "occurrenceCount": 2,
          "properties": {
            "resultGroup": "Angle Brackets Syntax",
            "taskName": "ember-octane-migration-status",
            "taskDisplayName": "Ember Octane Migration Status",
            "category": "migrations",
            "group": "ember"
          }
        },
        {
          "ruleId": "no-implicit-this",
          "message": {
            "text": "Ambiguous path 'accounts.fetchSubscriptions.isRunning' is not allowed. Use '@accounts.fetchSubscriptions.isRunning' if it is a named argument or 'this.accounts.fetchSubscriptions.isRunning' if it is a property on 'this'. If it is a helper or component that has no arguments, you must either convert it to an angle bracket invocation or manually add it to the 'no-implicit-this' rule configuration, e.g. 'no-implicit-this': { allow: ['accounts.fetchSubscriptions.isRunning'] }."
          },
          "locations": [],
          "occurrenceCount": 55,
          "properties": {
            "resultGroup": "Own Properties",
            "taskName": "ember-octane-migration-status",
            "taskDisplayName": "Ember Octane Migration Status",
            "category": "migrations",
            "group": "ember"
          }
        },
        {
          "ruleId": "no-action",
          "message": {
            "text": "Do not use `action` as {{action ...}}. Instead, use the `on` modifier and `fn` helper."
          },
          "locations": [],
          "occurrenceCount": 237,
          "properties": {
            "resultGroup": "Modifiers",
            "taskName": "ember-octane-migration-status",
            "taskDisplayName": "Ember Octane Migration Status",
            "category": "migrations",
            "group": "ember"
          }
        }
      ],
      "invocations": [
        {
          "arguments": [],
          "executionSuccessful": true,
          "endTimeUtc": "Thu, 08 Oct 2020 00:35:47 GMT",
          "environmentVariables": {
            "cwd": "/Users/ckessler/travis-web",
            "outputFile": "",
            "format": "json"
          },
          "toolExecutionNotifications": [],
          "startTimeUtc": "Thu, 08 Oct 2020 00:35:01 GMT"
        }
      ],
      "tool": {
        "driver": {
          "name": "Checkup"
        }
      }
    }
  ]
}
```
</details>

[proposal]: #proposal

Based on the details listed above, I propose the following:

1. Changing JSON results to adhere to SARIF schema
2. Migrating all builders/utilities to create types that adhere to the SARIF schema (so using SARIF schema under the hood) 
3. [STRETCH] add SARIF visualization tool to checkup to use during local development
   