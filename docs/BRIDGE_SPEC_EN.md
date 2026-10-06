# AI Professor Lite Bridge API Specification v0.1

**Status:** Draft  
**Baseline date:** 2026-10-07

## 1. Scope

This specification defines the minimum compatibility contract that allows AI Professor Lite to use the same publishing flow with Moodle and non-Moodle platforms.

A bridge does not implement Moodle as a whole. It implements only the endpoints and Web Service functions required by the current AI Professor Lite publishing workflow.

## 2. Base URL and Endpoints

AI Professor Lite is configured with a Bridge Base URL and API token.

The following relative paths MUST be available below that base URL:

~~~text
/webservice/rest/server.php
/webservice/upload.php
~~~

The REST endpoint MUST accept POST. A bridge MAY return an error JSON object for GET requests, but SHOULD return a non-404 response so operators can verify that the endpoint exists.

## 3. Authentication

REST calls use:

~~~text
wstoken=<token>
wsfunction=<function>
moodlewsrestformat=json
~~~

Upload calls use:

~~~text
token=<token>
file_1=<multipart file>
~~~

Token comparison SHOULD be timing-safe. Tokens MUST NOT be exposed in logs, rendered HTML, or error messages.

## 4. Required Functions

A conforming bridge MUST provide at least:

- core_webservice_get_site_info
- core_enrol_get_users_courses
- core_course_get_contents
- mod_simplevideotracker_create_activity
- mod_simplevideotracker_set_video

For legacy compatibility a bridge MAY also provide:

- mod_videotracker_create_activity
- mod_videotracker_set_video

AI Professor Lite may inspect the function list returned by site info and choose either the videotracker or simplevideotracker component when both required functions for that component are exposed.

## 5. create_activity

Input:

| Field | Required | Meaning | Default |
|---|---:|---|---:|
| courseid | Y | target mapped as a course | - |
| sectionnum | Y | section or logical publish location | - |
| name | Y | lecture title | - |
| intro | N | lecture introduction HTML/text | empty |
| completionpercent | N | completion threshold | 80 |
| preventseeking | N | block forward seek beyond verified progress | true |
| seektolerance | N | allowed seek tolerance in seconds | 3 |
| heartbeat | N | heartbeat interval in seconds | 5 |

The current AI Professor Lite publisher sends courseid, sectionnum, name, and intro. A bridge may apply defaults for the remaining fields.

A successful response MUST contain at least:

~~~json
{
  "success": true,
  "cmid": 123
}
~~~

Recommended additional fields are courseid, sectionnum, instanceid, name, and url.

cmid MUST be a stable activity identifier that allows the bridge to resolve the lecture later.

## 6. File Upload

/webservice/upload.php accepts a multipart upload and returns a JSON array.

AI Professor Lite reads itemid or draftitemid from the first item.

Minimum compatible example:

~~~json
[
  {
    "itemid": 456,
    "filename": "lecture.mp4"
  }
]
~~~

The draft identifier MUST remain associated with the authenticated token until set_video consumes it. Temporary uploads SHOULD have an expiry and cleanup policy.

## 7. set_video

Input:

| Field | Required | Meaning |
|---|---:|---|
| cmid | Y | target activity identifier |
| draftitemid | Y | temporary file ID returned by upload |
| duration | N | authoritative video duration in seconds |

If duration is supplied, it MUST be validated as finite and positive.

A successful response SHOULD contain at least:

~~~json
{
  "success": true,
  "cmid": 123
}
~~~

Platform implementations may add stored filename, URL, duration, change flags, or other metadata.

## 8. Platform Mapping

| Common concept | Moodle | Rhymix | WordPress | GnuBoard5 |
|---|---|---|---|---|
| Course | course | configured board/module | logical publish target | bo_table |
| Activity/cmid | course module id | document_srl or bridge id | post_id or bridge id | wr_id or bridge id |
| User | user.id | member_srl | WP user ID | mb_id |
| Video | Moodle file area | Rhymix attachment | Media Library | board file |
| Progress | plugin table | bridge progress table | bridge progress table | bridge progress table |

Internal identifier formats may differ, but the external contract MUST remain stable.

## 9. Errors

Moodle-style JSON errors are RECOMMENDED:

~~~json
{
  "exception": "moodle_exception",
  "errorcode": "invalidparameter",
  "message": "..."
}
~~~

A bridge may preserve Moodle-compatible behavior where an application error is represented in a JSON body, but operational logs should clearly distinguish authentication, validation, and internal failures.

## 10. Idempotency

create_activity has an inherent duplicate-creation risk. Bridges SHOULD support an external_id or equivalent idempotency mechanism when practical.

When a create result is ambiguous, AI Professor Lite favors operator verification over blind automatic recreation.

set_video SHOULD be designed so repeating the same attachment operation does not corrupt state.

## 11. Security

- HTTPS SHOULD be used.
- API tokens SHOULD be long random values.
- Upload extension and MIME validation SHOULD be enforced.
- Path traversal MUST be prevented.
- Temporary uploads SHOULD expire and be cleaned up.
- User-supplied HTML SHOULD be sanitized.
- Learning progress MUST be validated server-side.
- Secrets MUST NOT be logged.
