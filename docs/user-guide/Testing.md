# RESTler Test Mode

In *Test* mode, RESTler will attempt to execute successfully all request types of the Swagger specification at least once in order to identify request types that are unreachable/unexercizable with the current test setup. In other words, the purpose of a such a test, also called smoketest, is to quickly (minutes) debug the current test setup.

Inputs:

1. a grammar.py file generated by the RESTler compiler

    a RESTler fuzzing dictionary in JSON format

    a configuratio​n file and/or script that may be used to obtain a fresh authentication token, if required by your API.


Authentication options: configuring authentication is described in [Authentication​](Authentication.md).

4. a set of command-line options describing how to reach the service.  See the example below, or run Restler.exe without arguments to get the list of supported options.

How to invoke RESTler in test mode:

`C:\RESTler\restler\Restler.exe test --grammar_file <RESTLer grammar.py file> --dictionary_file <RESTler fuzzing-dictionary.json file> --token_refresh_interval <time in seconds> --token_refresh_command <command>`

Outputs: see the sub-directory Test

RESTLer will generate a sub-directory `Test\RestlerResults\experiment<GUID>\logs` including the following files:

- `main.txt` is the main log documenting how each request is attempted to be executed - an INVALID status means that RESTler could not execute that request successfully

- `request_rendering.txt` reports overall progress

Example: see [Tutorial](TutorialDemoServer.md)

This file ends with

>Rendered requests with "valid" status codes: 13 / 13

which means that all 13 requests were VALID and thus executed successfully during the test - this is the best possible outcome since RESTler was about to achieve 13/13, that is, 100% Swagger specification coverage.

- `network.<threadID>.txt` logs all HTTP(S) traffic generated with RESTler, including all REST API requests executd and their responses. This file is useful for detailed debugging. For instance, if some requests are never executed successfully by RESTler during a smkotest (INVALID status), the corresponding detailed requests generated by RESTler and their responses should be examined in order to troubleshoot and possibly fix the Swagger spec with annotations (see above) or fix the grammar.py, which can be manually edited, as discussed further below.

- `garbage_collector.<threadID>.txt` and the corresponding `network.<threadID>.txt` are, repespectively, the RESTler garbage-collector logs and the garbage-collector detailed traffic logs. Those logs can be safely ignored except for troubleshooting the garbage-collector, for instance in case of resource leaks.


RESTler will also generate a sub-directory `Test\ResponseBuckets` including the following files:

- runSummary.json is a report on all the HTTP response codes that were received

- errorBuckets.json includes a sample of up to 10 pairs of <request, response> for each HTTP error codes in the 4xx or 5xx ranges that were received

**Warning**: after running RESTler in test mode, you should monitor the service under test and delete any remaining resources created by RESTler, if any.  These left-over resources may either be the result of leaks (i.e. bugs found by RESTler) or limitations in how RESTler is able to garbage-collect resources after testing (e.g. if resources are left after fuzzing in a state where they cannot be deleted).

## Speccov.json File
At the end of each Test run a `speccov.json` file will be created in the logs directory.
This file contains test results for each request in the grammar.
Each request is represented by a hash of its definition.

#### Example of a single request from the json file:
```
    "5915766984a7c5deaaae43cae4cfb810c138d0f2": {
        "verb": "PUT",
        "endpoint": "/blog/posts/{postId}",
        "verb_endpoint": "PUT /blog/posts/{postId}",
        "valid": 0,
        "matching_prefix": {
            "id": "1d7752f6d5ca3e03e423967a57335038a3d1bb70",
            "valid": 1
        },
        "invalid_due_to_sequence_failure": 0,
        "invalid_due_to_resource_failure": 0,
        "invalid_due_to_parser_failure": 0,
        "invalid_due_to_500": 0,
        "status_code": "400",
        "status_text": "BAD REQUEST",
        "error_message": "{\n    \"errors\": {\n        \"id\": \"'5882' is not of type 'integer'\"\n    },\n    \"message\": \"Input payload validation failed\"\n}\n",
        "sample_request": {
            "request_sent_timestamp": null,
            "response_received_timestamp": "2021-03-31 18:20:14",
            "request_uri": "/api/blog/posts/5882",
            "request_headers": [
                "Accept: application/json",
                "Host: localhost:8888",
                "Content-Type: application/json"
            ],
            "request_body": "{\n    \"id\":\"5882\",\n    \"checksum\":\"fuzzstring\",\n    \"body\":\"fuzzstring\"}\r\n",
            "response_headers": [
                "Content-Type: application/json",
                "Content-Length: 124",
                "Server: Werkzeug/0.16.0 Python/3.7.8",
                "Date: Wed, 31 Mar 2021 18:20:14 GMT"
            ],
            "response_body": "{\n    \"errors\": {\n        \"id\": \"'5882' is not of type 'integer'\"\n    },\n    \"message\": \"Input payload validation failed\"\n}\n"
        },
        "request_order": 4
    },
```

In any of the boolean values above, 0 represents False and 1 represents True.

* The __"verb"__ and __"endpoint"__ values are as you would expect from the request.
* The __"valid"__ value specifies whether or not the request was considered valid by RESTler standards.
  * For a request to be "valid" it must have received a 2xx response from the server.
* The __"matching_prefix"__ dict contains the hash ID for the request that contained the matching prefix
and whether or not _that_ request was valid.
  * If there was no matching prefix, it would sayd "None".
* If a request was invalid,
the appropriate __"invalid_due_to_..."__ value will be set to 1.
  * "sequence_failure" will be set if a failure occurs while rendering a previously valid prefix sequence.
  * "resource_failure" will be set if the server responded with a 2xx,
  but the async resource creation polling indicated that there was a failure when creating the resource.
  * "parser_failure" will be set if the server responded with a 2xx,
  but there was a failure while parsing the response data.
  * "500" will be set if a 5xx bug was detected.
* The __"status_code"__ and __"status_text"__ values are the response values received from the server.
* The __"sample_request"__ contains the concrete values of the sent request and received response for which
the coverage data is being reported.
* The __"error_message"__ value will be set to the response body if the request was not "valid".
* The __"request_order"__ value is the 0 indexed order that the request was sent.
  * Requests sent during "preprocessing" or "postprocessing" will explicitely say so.

#### Postprocessing Scripts:
The `utilities` directory contains a sub-directory called `speccovparsing` that contains scripts for postprocessing speccov files.

* `diff_speccov.py` can be run to diff speccov files
and output the diff as a new json file.
  * A "left" file is chosen as the baseline file
  and a list of multiple "right" files can be specified to be compared to the left file.
* `sum_speccov.py` simply adds up the final coverage and failure types
and creates a new json file with the output.