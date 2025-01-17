Artifactory Artifact Cleanup User Plugin
========================================

This plugin deletes all artifacts that have not been downloaded for the past *n time units*,
which is by default 1 month. It can be run manually from the REST API, or automatically as a scheduled job.

Many delete operations can affect performance due to disk I/O occurring. A new parameter now allows a delay per delete operation. See below.

To ensure logging for this plugin, edit ${ARTIFACTORY_HOME}/etc/logback.xml to add:

```xml
    <logger name="artifactCleanup">
        <level value="info"/>
    </logger>
```

**Note:**

If you're trying to clean Docker images, this plugin may lead to unexpectedly partial or broken cleans. It is recommended to instead use the [cleanDockerImages](https://github.com/jfrog/artifactory-user-plugins/tree/master/cleanup/cleanDockerImages) plugin for this purpose.

Parameters
----------

- `months`: **Deprecated**. Instead of `timeUnit` and `timeInterval` the `month` parameter is supported for backwards compatibility reasons. It defined the months to look back before deleting an artifact. Default *1*.
- `timeUnit`: The unit of the time interval. *year*, *month*, *day*, *hour* or *minute* are allowed values. Default *month*.
- `timeInterval`: The time interval to look back before deleting an artifact. Default *1*.
- `repos`: A list of repositories to clean. This parameter is required.
- `dryRun`: If this parameter is passed, artifacts will not actually be deleted. Default *false*.
- `disablePropertiesSupport`: Disable the support of Artifactory Properties (see below *Artifactory Properties support* section). Default
- `paceTimeMS`: The number of milliseconds to delay between delete operations. Default *0*.
- `keepArtifacts`: Enable protection of artifacts that should not be deleted. Default *false*.
- `keepArtifactsRegex`: Regular expression for protecting artifacts to not be deleted. Default `/.*-[\.\d+]*[-+][\.\d+]*\..*/`.

Artifactory Properties support
----------

Some Artifactory [Properties](https://www.jfrog.com/confluence/display/RTF/Properties) are supported if defined on *artifacts* or *folders*:

- `cleanup.skip`: Skip the artifact deletion if property defined on artifact's path ; artifact itself or in a parent folder(s).

`artifactCleanup.json`
----------

The json contains the policies for scheduling cleanup jobs for different repositories.

The following properties are supported by each policy descriptor object:

- `cron`: The cron time when the job is executed. Default `0 0 5 ? * 1` *should* be redefined.
- `repos`: The mandatory list of repositories the policies will be applied to.
- `timeUnit`: The unit of the time interval. *year*, *month*, *day*, *hour* or *minute* are allowed values. Default *month*.
- `timeInterval`: The time interval to look back before deleting an artifact. Default *1*.
- `dryRun`: If this parameter is passed, artifacts will not actually be deleted. Default *false*.
- `paceTimeMS`: The number of milliseconds to delay between delete operations. Default *0*.
- `disablePropertiesSupport`: Disable the support of Artifactory Properties (see above *Artifactory Properties support* section).
- `keepArtifacts`: Enable protection of artifacts that should not be deleted. Default *false*.
- `keepArtifactsRegex`: Regular expression for protecting artifacts to not be deleted. Default `~/.*-[\.\d+]*[-+][\.\d+]*\..*/`.

An example file could contain the following json:

```json
{
    "policies": [
        {
            "cron": "0 0 1 ? * 1",
            "repos": [
                "libs-releases-local"
            ],
            "timeUnit": "day",
            "timeInterval": 3,
            "dryRun": true,
            "paceTimeMS": 500,
            "disablePropertiesSupport": true,
            "keepArtifacts": true,
            "keepArtifactsRegex": "~/.*-[.\\d+]+-\\d{2}\\.[1-4][\\.\\w+]*/"
        }
    ]
}
```

The regular expression used in the above example prevents the plugin from deleting artifacts that contain a version X.Y.Z and release number based on year and quarter (eg. 21.3).

  ```bash
  my-artifact-1.2.0-21.3
  my-artifact-1.2.0-21.3.zip
  my-artifact-1.2.0-21.3+4.5.6
  my-artifact-1.2.0-21.3+4.5.6.zip
  ```

**Note**: If a deprecated `artifactCleanup.properties` is defined it will only be applied if no `artifactCleanup.json` is present.

`artifactCleanup.properties` (DEPRECATED)
----------

The properties file contains the policies for scheduling cleanup jobs for different repositories.

The following properties are supported by each policy descriptor object:

- `cron`: The cron time when the job is executed. Default `0 0 5 ? * 1` *should* be redefined.
- `repos`: The mandatory list of repositories the policies will be applied to.
- `months`: The number of months back to look before deleting an artifact. Default: *6*.
- `paceTimeMS`: The number of milliseconds to delay between delete operations. Default *0*.
- `dryRun`: If this parameter is passed, artifacts will not actually be deleted. Default *false*.
- `disablePropertiesSupport`: Disable the support of Artifactory Properties (see above *Artifactory Properties support* section).
- `keepArtifacts`: Enable protection of artifacts that should not be deleted. Default *false*.
- `keepArtifactsRegex`: Regular expression for protecting artifacts to not be deleted. Default `~/.*-[\.\d+]*[-+][\.\d+]*\..*/`.

An example file could contain the following policy:

```groovy
policies = [
               [ "0 0 5 ? * 1", [ "libs-release-local" ], 3, 500, true, true, true, ~/.*-\d+\.\d+\.\d+\.*/ ],
           ]
```

Executing
---------

To execute the plugin:

For Artifactory 4.x:
`curl -X POST -v -u admin:password "http://localhost:8080/artifactory/api/plugins/execute/cleanup?params=timeUnit=month|timeInterval=1|repos=libs-release-local|dryRun=true|paceTimeMS=2000|disablePropertiesSupport=true"`

For Artifactory 5.x or higher:
`curl -X POST -v -u admin:password "http://localhost:8080/artifactory/api/plugins/execute/cleanup?params=timeUnit=month;timeInterval=1;repos=libs-release-local;dryRun=true;paceTimeMS=2000;disablePropertiesSupport=true"`

For Artifactory 5.x or higher, using the depracted `months` parameter:
`curl -X POST -v -u admin:password "http://localhost:8080/artifactory/api/plugins/execute/cleanup?params=months=1;repos=libs-release-local;dryRun=true;paceTimeMS=2000;disablePropertiesSupport=true"`

For Artifactory 5.x or higher, using the `keepArtifactsRegex` parameter:

In order to pass a regular expression to a POST request we need to encode the expression to an URL-encoded format. You can either do this by using this
Python3 one-liner:

```bash
python3 -c "import urllib.parse, sys; print(urllib.parse.quote_plus(sys.argv[1]))" "~/.*-[\.\d+]*[-+][\.\d+]*\..*/"
```

or use [www.urlencoder.org](https://www.urlencoder.org).

Once you have encoded the regular expression you can call the cleanup script:

```bash
curl -X POST -uadmin:password "http://localhost:8080/artifactory/api/plugins/execute/cleanup?params=timeUnit=month;timeInterval=1;repos=libs-release-local;dryRun=true;paceTimeMS=2000;disablePropertiesSupport=true;keepArtifactsRegex=~%2F.%2A-%5B%5C.%5Cd%2B%5D%2A%5B-%2B%5D%5B%5C.%5Cd%2B%5D%2A%5C..%2A%2F"
```

Admin users and users inside the `cleaners` group can execute the plugin.

There is also ability to control the running script. The following operations can occur

Operation
---------

The plugin has 4 control options:

- `stop`: When detected, the loop deleting artifacts is exited and the script ends. Example:

`curl -X POST -v -u admin:password "http://localhost:8080/artifactory/api/plugins/execute/cleanupCtl?params=command=stop"`

- `pause`: Suspend operation. The thread continues to run with a 1 minute sleep for retest. Example:

`curl -X POST -v -u admin:password "http://localhost:8080/artifactory/api/plugins/execute/cleanupCtl?params=command=pause"`

- `resume`: Resume normal execution. Example:

`curl -X POST -v -u admin:password "http://localhost:8080/artifactory/api/plugins/execute/cleanupCtl?params=command=resume"`

- `adjustPaceTimeMS`: Modify the running delay factor by increasing/decreasing the delay value. Example:

For Artifactory 4.x
`curl -X POST -v -u admin:password "http://localhost:8080/artifactory/api/plugins/execute/cleanupCtl?params=command=adjustPaceTimeMS|value=-1000"`

For Artifactory 5.x or higher:
`curl -X POST -v -u admin:password "http://localhost:8080/artifactory/api/plugins/execute/cleanupCtl?params=command=adjustPaceTimeMS;value=-1000"`
