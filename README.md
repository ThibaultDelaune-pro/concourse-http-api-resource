# HTTP API Resource

Concourse resource to allow interaction with (simple) HTTP (REST/JSON) API's. This resource is useful for API's which have simple one request interactions and will not be a one-size-fits-all solution. If your API is more complex [writing your own resource](http://concourse.ci/implementing-resources.html).

https://hub.docker.com/r/aequitas/http-api-resource/

## Recent changes

- 09-2018 Fix error on check (reporter: @OvermindDL1)
- 02-2018 Fix build detail load (@scottasmith, reporter: @pranaysharmadelhi)
- 02-2018 add support for UTF-8 in JSON (@masaki-takano)
- 01-2018 fix concourse 3.1.1 compatibility (@hfinucane)
- 01-2018 added license
- 01-2017 recursive interpolate over lists as well

## Source Configuration

Most of the `source` options can also be used in `params` for the specifc actions. This allows to use a different URL path. For example when a POST and GET use different endpoints.

Options set in `params` take precedence over options in `source`.

* `uri`: *Required.* The URI to use for the requests.
    Example: `https://www.hipchat.com/v2/room/1234321/notification`

* `method`: *Optional* Method to use, eg: GET, POST, PATCH (default `GET`).

* `headers`: *Optional* Object containing headers to pass to the request.
    Example:

        headers:
            X-Some-Header: some header content

* `json`: *Optional* JSON to send along with the request, set `application/json` header.

* `debug`: *Optional* Set debug logging of scripts, takes boolean (default `false`).

* `ssl_verify`: *Optional* Boolean or SSL CA content (default `true`).

* `form_data`: *Optional* Dictionary with form field/value pairs to send as data. Values are converted to JSON and URL-encoded.

* `parse_form_data`: *Optional* Boolean to specify if form_data should be converted to JSON or not (default `true`). If false, set `"Content-Type":"application/x-www-form-urlencoded"` header.

## Behavior

Currently the only useful action the resource supports is `out`. The actions `in` and `check` will be added later.

### Interpolation

All options support interpolation of variables by using [Python string formatting](https://docs.python.org/3.5/library/stdtypes.html#str.format).

In short it means variables can be used by using single curly brackets (instead of double for Concourse interpolation). Eg: `Build nr. {BUILD_NAME} passed.`

Build metadata (BUILD_NAME, BUILD_JOB_NAME, BUILD_PIPELINE_NAME and BUILD_ID) are available as well as the merged `source`/`params` objects. Interpolation will happen after merging the two objects.

Be aware that options containing interpolation variables need to be enclosed in double quotes `"`.

See Hipchat below for usage example.

## Examples

### Post notification on HipChat

This example show use of variable interpolation with build metadata and the params dict.

Also shows how the usage of a authentication header using Concourse variables.


```yaml
resource_types:
  - name: http-api
    type: docker-image
    source:
      repository: aequitas/http-api-resource
      tag: latest

resources:
    - name: hipchat
      type: http-api
      source:
          uri: https://www.hipchat.com/v2/room/team_room/notification
          method: POST
          headers:
              Authorization: "Bearer {hipchat_token}"
          json:
              color: "{color}"
              message: "Build {BUILD_PIPELINE_NAME}{BUILD_JOB_NAME}, nr: {BUILD_NAME} {message}!"
          hipchat_token: {{HIPCHAT_TOKEN}}

jobs:
    - name: Test and notify
      plan:
          - task: build
            file: ci/build.yaml
            on_success:
                put: hipchat
                params:
                    color: green
                    message: was a success
            on_failure:
                put: hipchat
                params:
                    color: red
                    message: failed horribly

```

### Trigger build in Jenkins

Trigger the job `job_name` with parameter `package` set to `test`.

More info: https://wiki.jenkins-ci.org/display/JENKINS/Remote+access+API#RemoteaccessAPI-Submittingjobs

```yaml
resources:
    - name: jenkins-trigger-job
      type: http-api
      source:
          uri: http://user:token@jenkins.example.com/job/job_name/build
          method: POST
          form_data:
              json:
                  parameter:
                    - name: package
                      value: test

jobs:
    - name: Test and notify
      plan:
          - task: build
            file: ci/build.yaml

          - put: jenkins-trigger-job

```


### Call a Rest API with data formated in a standard form

Register apps on a Spring Cloud Data Flow instance via a `POST` formated with `application/x-www-form-urlencoded` encoding type.

More info: https://docs.spring.io/spring-cloud-dataflow/docs/current/reference/htmlsingle/#resources-app-registry-bulk

```yaml
resources:
- name: dataflow-rest-api
  type: http-api
  source:
    uri: http://my-dataflow-platform.org

jobs:
- name: load-apps
  public: true
  plan:
  - put: dataflow-rest-api
    params:
      uri: http://my-dataflow-platform.org/apps
      method: POST
      debug: true
      parse_form_data: false
      form_data:
        uri: http://artefact.my-repo.com/artifactory/Maven/appsList/0.0.1/appsList-0.0.1.properties
```