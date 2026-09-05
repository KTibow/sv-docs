## StudentVue App API Docs

Documentation and research of the API routes from the Official StudentVue Apps.

There are two app generations with different APIs, both currently on the app stores:

| Doc | App | IDs | Protocol | Auth |
|-----|-----|-----|----------|------|
| [SOAP-API.md](SOAP-API.md) | **StudentVUE** (legacy, 1M+ installs) | Android `com.FreeLance.StudentVUE` · iOS `studentvue/id412050327` | SOAP — `ProcessWebServiceRequest` on `PXPCommunication.asmx` | `userID` + `password` fields in the SOAP envelope |
| [JSON-API.md](JSON-API.md) | **StudentVUE (New)** (100K+ installs) | Android `com.edupoint.studentvue` · iOS `studentvue-new/id6618112702` | JSON — `POST /api/v1/mobile/PXPWebServices/<Method>`, plus reduced SOAP (`ProcessWebServiceRequestMultiWeb`) | HTTP Basic at login → Bearer token |

### Which one do I use?

Whichever app your school makes you use — these days that's usually **StudentVUE (New)**. If you're on the old app, use [SOAP-API.md](SOAP-API.md).

### Shared concepts

- District hosts look like `https://<district>.edupoint.com`; the district lookup by zip code against `support.edupoint.com` returns each district's `PvueURL` (see [SOAP-API.md](SOAP-API.md#getting-zip-codes) and [JSON-API.md](JSON-API.md#district-lookup)).
- `PXPWebServices` is the StudentVUE web-service handle; TeacherVUE/AdminVUE/etc. have their own handles (see [JSON-API.md](JSON-API.md#soap)).
- JSON-API errors arrive inside HTTP 200 bodies — see [JSON-API.md](JSON-API.md#errors) before writing a parser.
