# JUnit Reporter for Cypress 

[![Build Status][travis-badge]][travis-build]
[![npm][npm-badge]][npm-listing]

Produces JUnit-style XML test results.

The main reason for this fork is to enable options like 'attachments', 'consoleOutputs' and other in cypress, which does
not work in cypress with base 'mocha-junit-reporter';

## Installation
Is compatible with cypress version >= 10.

```shell
$ npm install @vpavlik62/cypress-junit-reporter --save-dev
```

or as a global module
```shell
$ npm install -g @vpavlik62/cypress-junit-reporter
```

## Usage
Cypress configuration:
```javascript
const { defineConfig } = require('cypress')

module.exports = defineConfig({
  reporter: '@vpavlik62/cypress-junit-reporter',
  reporterOptions: {
    mochaFile: 'results/my-test-output.[hash].xml',
    toConsole: true
  }
})
```
or with `cypress-multi-reporters`
```javascript
const { defineConfig } = require('cypress')

module.exports = defineConfig({
  reporter: 'cypress-multi-reporters',
  reporterOptions: {
    configFile: 'reporter-config.json'
  }
})
```

having `reporter-config.json`
```json
{
    "reporterEnabled": "spec, @vpavlik62/cypress-junit-reporter",
    "vpavlik62CypressJunitReporterReporterOptions": {
        "mochaFile": "results/test-output.[hash].xml",
        "toConsole": true
    }
}
```

### Typescript

To avoid typescript errors, put this inside your `globa.d.ts`:

```javascript
declare namespace Cypress {

    // specify additional properties in the TestConfig object
    interface TestConfigOverrides {
        attachments?: string[];
        consoleOutputs?: string[];
        consoleErrors?: string[];
    }
}
```

### Debug

To see debug log set env variable:
```shell
DEBUG='cypress-junit-reporter'
```

### Append properties to testsuite

You can also add properties to the report under `testsuite`. This is useful if you want your CI environment to add extra build props to the report for analytics purposes

```xml
<testsuites>
  <testsuite>
    <properties>
      <property name="BUILD_ID" value="4291"/>
    </properties>
    <testcase/>
    <testcase/>
    <testcase/>
  </testsuite>
</testsuites>
```

To do so pass them in via env variable:
```shell
PROPERTIES=BUILD_ID:4291
```
or
```javascript
const { defineConfig } = require('cypress')

module.exports = defineConfig({{
    reporter: '@vpavlik62/cypress-junit-reporter',
    reporterOptions: {
        properties: {
            BUILD_ID: 4291
        }
    }
})
```

### Results Report

Results XML filename can contain `[hash]`, e.g. `./path_to_your/test-results.[hash].xml`. `[hash]` is replaced by MD5 hash of test results XML. This enables support of parallel execution of multiple `mocha-junit-reporter`'s writing test results in separate files.

In order to display full suite title (including parents) just specify `testsuitesTitle` option
```javascript
const { defineConfig } = require('cypress')

module.exports = defineConfig({{
    reporter: '@vpavlik62/cypress-junit-reporter',
    reporterOptions: {
        testsuitesTitle: true,
        suiteTitleSeparatedBy: '.' // suites separator, default is space (' '), or period ('.') in jenkins mode
    }
});
```

If you want to **switch classname and name** of the generated testCase XML entries, you can use the `testCaseSwitchClassnameAndName` reporter option.

```javascript
const { defineConfig } = require('cypress')

module.exports = defineConfig({{
    reporter: '@vpavlik62/cypress-junit-reporter',
    reporterOptions: {
        testCaseSwitchClassnameAndName: true
    }
});
```

Here is an example of the XML output when using the `testCaseSwitchClassnameAndName` option:

| value             | XML output                                                                              |
| ----------------- | --------------------------------------------------------------------------------------- |
| `true`            | `<testcase name="should behave like so" classname="Super Suite should behave like so">` |
| `false` (default) | `<testcase name="Super Suite should behave like so" classname="should behave like so">` |

You can also configure the `testsuites.name` attribute by setting `reporterOptions.testsuitesTitle` and the root suite's `name` attribute by setting `reporterOptions.rootSuiteTitle`.

### System out and system err
The JUnit format defines a pair of tags - `<system-out/>` and `<system-err/>` - for describing a test's generated output
and error streams, respectively. It is possible to pass the test outputs/errors as an array of text lines:
```js
it ('should report output', function () {
  this.test._testConfig.consoleOutputs = [ 'line 1 of output', 'line 2 of output' ];
});
it ('should report error', function () {
  this.test._testConfig.consoleErrors = [ 'line 1 of errors', 'line 2 of errors' ];
});
```

Since this module is only a reporter and not a self-contained test runner, it does not perform
output capture itself. Thus, the author of the tests is responsible for providing a mechanism
via which the outputs/errors array will be populated.

If capturing only console.log/console.error is an option, a simple (if a bit hack-ish) solution is to replace
the implementations of these functions globally, like so:
```js
var util = require('util');

describe('my console tests', function () {
  var originalLogFunction = console.log;
  var originalErrorFunction = console.error;
  beforeEach(function _mockConsoleFunctions() {
    var currentTest = this.currentTest;
    console.log = function captureLog() {
      var formattedMessage = util.format.apply(util, arguments);
      currentTest._testConfig.consoleOutputs = (currentTest._testConfig.consoleOutputs || []).concat(formattedMessage);
    };
    console.error = function captureError() {
      var formattedMessage = util.format.apply(util, arguments);
      currentTest._testConfig.consoleErrors = (currentTest._testConfig.consoleErrors || []).concat(formattedMessage);
    };
  });
  afterEach(function _restoreConsoleFunctions() {
    console.log = originalLogFunction;
    console.error = originalErrorFunction;
  });
  it('should output something to the console', function() {
    // This should end up in <system-out>:
    console.log('hello, %s', 'world');
  });
});
```

Remember to run with `--reporter-options outputs=true` if you want test outputs in XML.

### Attachments
enabling the `attachments` configuration option will allow for attaching files and screenshots in [JUnit Attachments Plugin](https://wiki.jenkins.io/display/JENKINS/JUnit+Attachments+Plugin) format.

Attachment path can be injected into the test object (testConfig is the only property that is not cleared of unverified
attributes by cypress)
```js
it ('should include attachment', function () {
  this.test._testConfig.attachments = ['/absolut/path/to/file.png'];
});
```

If both attachments and outputs are enabled, and a test injects both consoleOutputs and attachments, then
the XML output will look like the following:
```xml
<system-out>output line 1
output line 2
[[ATTACHMENT|path/to/file]]</system-out>
```

### Full configuration options

| Parameter                      | Default                | Effect                                                                                                                  |
| ------------------------------ | ---------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| mochaFile                      | `test-results.xml`     | configures the file to write reports to                                                                                 |
| includePending                 | `false`                | if set to a truthy value pending tests will be included in the report                                                   |
| properties                     | `null`                 | a hash of additional properties to add to each test suite                                                               |
| toConsole                      | `false`                | if set to a truthy value the produced XML will be logged to the console                                                 |
| useFullSuiteTitle              | `false`                | if set to a truthy value nested suites' titles will show the suite lineage                                              |
| suiteTitleSeparatedBy          | ` ` (space)            | the character to use to separate nested suite titles. (defaults to ' ', '.' if in jenkins mode)                         |
| testCaseSwitchClassnameAndName | `false`                | set to a truthy value to switch name and classname values                                                               |
| rootSuiteTitle                 | `Root Suite`           | the name for the root suite. (defaults to 'Root Suite')                                                                 |
| testsuitesTitle                | `Mocha Tests`          | the name for the `testsuites` tag (defaults to 'Mocha Tests')                                                           |
| outputs                        | `false`                | if set to truthy value will include console output and console error output                                             |
| attachments                    | `false`                | if set to truthy value will attach files to report in `JUnit Attachments Plugin` format (after console outputs, if any) |
| antMode                        | `false`                | set to truthy value to return xml compatible with [Ant JUnit schema][ant-schema]                                        |
| antHostname                    | `process.env.HOSTNAME` | hostname to use when running in `antMode`  will default to environment `HOSTNAME`                                       |
| jenkinsMode                    | `false`                | if set to truthy value will return xml that will display nice results in Jenkins                                        |
| jenkinsClassnamePrefix         | `undefined`            | adds a prefix to a classname when running  in `jenkinsMode`                                                             |
| includeSpecFile                | `false`                | set to truthy value to include spec file (with relative path) in test name                                              |

[travis-badge]: https://travis-ci.org/michaelleeallen/mocha-junit-reporter.svg?branch=master
[travis-build]: https://travis-ci.org/michaelleeallen/mocha-junit-reporter
[npm-badge]: https://img.shields.io/npm/v/mocha-junit-reporter.svg?maxAge=2592000
[npm-listing]: https://www.npmjs.com/package/mocha-junit-reporter
[ant-schema]: http://windyroad.org/dl/Open%20Source/JUnit.xsd
