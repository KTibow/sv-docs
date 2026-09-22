## StudentVue JSON API Docs

Documentation of the API routes used by the current **StudentVUE (New)** app — Android `com.edupoint.studentvue`, iOS `studentvue-new/id6618112702` (researched against Android v1.9.16). The legacy **StudentVUE** app (`com.FreeLance.StudentVUE`, iOS `studentvue/id412050327`) is the one documented in [SOAP-API.md](SOAP-API.md).

The new app uses a JSON API for student data and keeps a subset of SOAP for configuration and account tasks. Authentication for the JSON API is HTTP Basic at login, then a Bearer token. GUIDs, tokens, and personal fields in the examples are placeholders; the district host is written as `https://<district>.edupoint.com`.

### TOC
[Transports](#transports)

[The JSON API](#the-json-api)

[Authentication](#authentication)

[Token refresh](#token-refresh)

[The edupointkeyversion header](#the-edupointkeyversion-header)

[Device attestation](#device-attestation)

[Errors](#errors)

[SOAP](#soap)

[Unauthenticated config methods](#unauthenticated-config-methods)

[Forgot password](#forgot-password)

[District lookup](#district-lookup)

[Method catalog](#method-catalog)

[Examples](#examples)

### Transports
[Top](#TOC)

| Endpoint | Used for |
|----------|----------|
| `POST https://<district>.edupoint.com/api/v1/mobile/PXPWebServices/<Method>` | all student data: schedule, gradebook, attendance, documents, mail, … |
| `POST https://<district>.edupoint.com/Service/PXPCommunication.asmx` | boot-time district config, activation, forgot password (SOAP) |
| `POST https://support.edupoint.com/Service/HDInfoCommunication.asmx` | district lookup by zip code (Edupoint-central) |

### The JSON API
[Top](#TOC)

**Example Request:**
```http
POST /api/v1/mobile/PXPWebServices/<Method> HTTP/1.1
Host: <district>.edupoint.com
Content-Type: application/json
User-Agent: ksoap
AppNameOSAndVersion: StudentVUE|Android|1.9.16
Authorization: Bearer <access_token>

{"arguments":{"request":"{\"childIntID\":0,\"languageCode\":\"en\"}"}}
```

**Example Response:**
```json
{"error": null, "data": {"studentDocuments": {}}}
```

**Notes:**

- The real parameters are a JSON object **serialized to a string** and placed at `arguments.request`. Not a nested object — a string. Some request fields (e.g. `UpdateNotificationPrefs`' `mobileUserSettings`) are JSON strings one level deeper still.
- Responses have exactly one method-specific key under `data` (see the [Method catalog](#method-catalog)). Field names ending in `XML` are usually JSON objects despite the name; a couple really are XML strings.
- `User-Agent: ksoap` is sent verbatim on JSON calls — a leftover from the SOAP days.
- `AppNameOSAndVersion` is `StudentVUE|Android|<appVersion>`.
- An optional `?DN=<districtDN>` query parameter is appended when the client knows a district number; omitting it works.
- ASP.NET session cookies are sent by the app but are not needed — bearer-only requests return full data. No certificate pinning was observed.

### Authentication
[Top](#TOC)

**Example Request:**
```http
POST /api/v1/mobile/PXPWebServices/AttemptLogin HTTP/1.1
Host: <district>.edupoint.com
Content-Type: application/json
User-Agent: ksoap
AppNameOSAndVersion: StudentVUE|Android|1.9.16
Authorization: Basic <base64(userID:password)>

{"arguments":{"request":"{\"userID\":null,\"password\":null,\"userType\":\"Student\"}"}}
```

**Example Response:**
```json
{
  "access_token": "<88 opaque characters>",
  "refresh_token": "<opaque>",
  "token_type": null,
  "expires_in": null,
  "scope": null
}
```

**Notes:**

- Credentials travel in the `Authorization: Basic` header only; the app nulls `userID`/`password` in the body. Both placements are accepted server-side, but header-only is what the app does.
- `userType` is `"Student"` for StudentVUE (`"parent"` on ParentVUE).
- `token`, `solu`, `decode` are only used for SAML/SSO re-entry; absent on a plain login.
- The login/refresh responses are a bare token object with **no `error` and no `data` key** — a different envelope than every other call. `expires_in`, `token_type`, `scope` were `null` on every observation; nothing about token expiry can be determined client-side.
- A failed login returns the ordinary error envelope, still with HTTP 200.

### Token refresh
[Top](#TOC)

**Example Request:**
```http
POST /api/v1/mobile/PXPWebServices/RefreshToken HTTP/1.1
Host: <district>.edupoint.com
Content-Type: application/json
User-Agent: ksoap
AppNameOSAndVersion: StudentVUE|Android|1.9.16
Authorization: Bearer <refresh_token>

{}
```

**Example Response:** the same bare token object as `AttemptLogin`, with a fresh pair.

**Notes:**

- Body is an empty JSON object — no `arguments` wrapper.
- The client does this automatically whenever a call returns HTTP **401** (the only non-200 status observed) and it holds a refresh token.

### The edupointkeyversion header
[Top](#TOC)

The app normally sends an `edupointkeyversion` header on every JSON call. Verified behaviour:

| request carries | result |
|---|---|
| header omitted | **accepted** (200) |
| header with a garbage value | **rejected** |
| header with a correctly computed value | accepted |

Clients can omit it today; the server-side check evidently exists behind a flag.

<details>
<summary>How the value is computed (reverse-engineered)</summary>

The app bundle contains CryptoJS-based helpers (`getGpaE`/`getGpaD`/`getGpaE2`/`getGpaES`/`getGpaDS`). The header uses `getGpaE2`, an AES-256-CBC encryption:

- **Plaintext:** `<MMDDYYYY>|<appVersion>|<MMDDYYYY>|android` — today's local date zero-padded, the app version, the date again, the literal platform. Verified in bytecode: exactly 7 concatenation parts.
- **Key:** a 32-byte ASCII constant embedded in the app's crypto helpers, trivially extractable from any APK. Its literal value is deliberately not published here — see the note below.
- **IV:** CryptoJS `Utf8.parse("AES")` — 3 bytes. CryptoJS XORs the IV word-per-32-bit-word and `x ^= undefined` coerces to `x ^= 0` in JS, so the effective 16-byte IV is `41 45 53 00 00 00 00 00 00 00 00 00 00 00 00 00`.
- **Padding/output:** PKCS#7, base64.

The same function encrypts the `<Parms>` XML of the forgot-password SOAP method (see [Forgot password](#forgot-password)).

Note: please don't paste live header values (or the extracted key) into issues, PRs, or pastes. If Edupoint rotates the key or starts enforcing the header, published values break things for everyone. Omitting the header works; prefer that.
</details>

### Device attestation
[Top](#TOC)

The client contains a device-key attestation flow that runs around login when a server-side `enableAttestation` flag is on: it fetches a signing challenge from `POST /api/v1/mobile/android-sign-challenge` (no auth; returns `challenge`, `challengeId`, `userGU`), signs it with a hardware-backed Keystore key, and submits `{keyAlias, publicKey, challenge, challengeId, attestationChain, packageName}` with the login. On a 403 "Key not registered" it wipes the local key and re-registers once.

As of v1.9.16 the flow is off in practice: the challenge endpoint returns 404 on districts observed, logins succeed with it skipped entirely, and there is a client kill switch. Nothing to implement today.

### Errors
[Top](#TOC)

Every call returns **HTTP 200, even on failure**. Check the body's `error` field:

**Example Response (gradebook at a school without gradebook):**
```json
{
  "error": {"code": "2100",
            "message": "Grade Book data not available for this school",
            "stackTrace": null},
  "data": null
}
```

**Example Response (module disabled district-wide):**
```json
{
  "error": {"code": "400",
            "message": "2100 - School District has not enabled access for the Fees Module.",
            "stackTrace": null},
  "data": null
}
```

**Example Response (server-side failure with a support ID):**
```json
{
  "error": {"code": "500",
            "message": "Error:  Please contact your school office for assistance. (ID: XXXXX)",
            "stackTrace": null},
  "data": null
}
```

**Notes:**

- `code "2100"` — the feature is not enabled at this school; treat as empty data, not as a failure.
- `code "400"` — e.g. past attendance not available for this school, or a module the district has disabled; the message often embeds a `2100 -` prefix even though the code is `400`.
- `code "500"` — generic server error. The `(ID: XXXXX)` suffix is a server-side event ID for the district's support desk; it varies per failure. Some `GetSynergyMail*` calls return this at districts even when `pxpMessagesData.supportingSynergyMail` is `true`.
- Don't treat unseen codes as benign; log them.
- This matters most for rarely-used areas (documents especially), where an empty result and an error look identical if you only check `data`.

### SOAP
[Top](#TOC)

The district SOAP endpoint is still `/Service/PXPCommunication.asmx`, but the wrapper element is now `ProcessWebServiceRequestMultiWeb` — the legacy `ProcessWebServiceRequest` from [SOAP-API.md](SOAP-API.md) plus a `webDBName` element. No `SOAPAction` header is sent.

**Example Request:**
```xml
POST /Service/PXPCommunication.asmx HTTP/1.1
Host: <district>.edupoint.com
Content-Type: text/xml; charset=utf-8

<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:web="http://edupoint.com/webservices/">
  <soapenv:Header/>
  <soapenv:Body>
    <web:ProcessWebServiceRequestMultiWeb>
      <web:userID></web:userID>
      <web:password></web:password>
      <web:skipLoginLog>1</web:skipLoginLog>
      <web:parent>0</web:parent>
      <web:webDBName></web:webDBName>
      <web:webServiceHandleName>PXPWebServices</web:webServiceHandleName>
      <web:methodName>GETSAMLSTATUS</web:methodName>
      <web:paramStr></web:paramStr>
    </web:ProcessWebServiceRequestMultiWeb>
  </soapenv:Body>
</soapenv:Envelope>
```

**Example Response:**
```xml
HTTP/1.1 200 OK
Content-Type: text/xml; charset=utf-8

<?xml version="1.0" encoding="utf-8"?><soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" ...><soap:Body><ProcessWebServiceRequestMultiWebResponse xmlns="http://edupoint.com/webservices/"><ProcessWebServiceRequestMultiWebResult>&lt;AllSAMLRecordData ... /&gt;</ProcessWebServiceRequestMultiWebResult></ProcessWebServiceRequestMultiWebResponse></soap:Body></soap:Envelope>
```

**Notes:**

- The inner result is HTML-escaped XML inside `ProcessWebServiceRequestMultiWebResult`, exactly like the legacy API — parse it twice.
- On failure the result contains `<RT_ERROR ERROR_MESSAGE="..." />` instead of a payload.
- Element order matters: `userID, password, skipLoginLog, parent, webDBName, webServiceHandleName, methodName, paramStr` (`parent` only in the `PXPWebServices` variant).
- Handle names by app: StudentVUE → `PXPWebServices`, TeacherVUE → `TXPWebServices`, AdminVUE → `AXPWebServices`, KioskVUE → `KXPWebServices`, HealthVUE → `HEALTHWEBSERVICES`, SEVUE → `SPECIALEDWEBSERVICES`, CounselVUE → `CounselorWebService`; help-desk lookups use `HDInfoServices` / `HDVueWebServices`.
- During the legacy API's deprecation the gate was per method: these config calls still work unauthenticated, while the student-data SOAP methods were the ones disabled.

### Unauthenticated config methods
[Top](#TOC)

Called by the app on cold start, before any login; all observed working with empty `userID`/`password`:

| `methodName` | `paramStr` | returns |
|---|---|---|
| `GetSupportedLanguages` | `&lt;Parms&gt;&lt;LanguageCode&gt;en&lt;/LanguageCode&gt;&lt;/Parms&gt;` | `<LanguageLists>` — `<Language Code="1" Name="English" EnglishDescription="English" />`, `Code="45"` Spanish, `Code="39"` Russian, … |
| `GETSAMLSTATUS` | *(empty)* | `<AllSAMLRecordData ShowLoginButtonForStudentVUE="false" ... />` — per-district SSO configuration |
| `GETACTIVATIONLINKSTATUS_PARENTVUE` | `&lt;Parms&gt;&lt;Parent&gt;0&lt;/Parent&gt;&lt;/Parms&gt;` | bare `true`/`false` — whether to show account activation |
| `SHOW_GET_FORGOT_PASSWORD_BUTTON_STATUS` | *(empty)* | bare `false`/`true` |
| `GETACKTEXT` | `&lt;Parms&gt;&lt;Parent&gt;0&lt;/Parent&gt;&lt;/Parms&gt;` (`1` for ParentVUE) | `<AckStatment PRIV_STMT="…">` — privacy acknowledgement text |

### Forgot password
[Top](#TOC)

`SHOW_GET_FORGOT_PASSWORD_UPDATE` (SOAP, unauthenticated, `skipLoginLog=1`) takes its `<Parms>` XML AES-encrypted through `getGpaE2` and base64-encoded into `paramStr`. Plaintext:

```xml
<Parms><UserName>…</UserName><Student>…</Student><Password>…</Password><Code>en</Code><LanguageCode>en</LanguageCode></Parms>
```

A bogus username returns `<RT_ERROR ERROR_MESSAGE="Error:  Please contact your school office for assistance." />` — no user-enumeration oracle. The JSON-side companion is `GetForgotPasswordEmailToken` (response key `twoFactorTokenData`).

### District lookup
[Top](#TOC)

Zip-code lookup against Edupoint central, as in [SOAP-API.md](SOAP-API.md) but wrapped in `ProcessWebServiceRequestMultiWeb` (handle `HDInfoServices`; `HDVueWebServices` for the help-desk variant):

```
paramStr = &lt;Parms&gt;&lt;Key&gt;5E4B7859-B805-474B-A833-FDB15D205D40&lt;/Key&gt;&lt;MatchToDistrictZipCode&gt;94127&lt;/MatchToDistrictZipCode&gt;&lt;/Parms&gt;
```

Returns `<DistrictLists><DistrictInfos><DistrictInfo DistrictID="" Name="…" Address="…" PvueURL="https://…/" /></DistrictInfos></DistrictLists>` (HTML-escaped in the result). `PvueURL` is the `<district>` host used everywhere else; `DistrictID` is the `DN` value the client can send as `?DN=`.

### Method catalog
[Top](#TOC)

All of these are JSON-API calls: `POST /api/v1/mobile/PXPWebServices/<Method>`. Request fields are the inner `request` object's fields, extracted from the app's generated typed client; defaults shown are the client's. **✅** = verified live against a district server; **◦** = from the client only.

| Method | Inner request fields | `data` key(s) | |
|--------|----------------------|---------------|--|
| `AttemptLogin` | `userID`, `password` (nulled in body), `userType`, `token`*, `solu`*, `decode`* | *(bare token object)* | ✅ |
| `RefreshToken` | *(none — empty body, Bearer refresh token)* | *(bare token object)* | ✅ |
| `GetChildListData` | `legacyAppRequest=false`, `secondaryLogin=false` | `children` | ✅ |
| `StudentClassList` | `childIntID`, `loadAllTerms`, `conSchOrgYearGU`, `conSchTermIndex`, `termIndex` (`"-1"` = all) | `studentClassSchedule`, `studentClassScheduleForAllTerms` | ✅ |
| `GetStudentClasesForGivenDay` | `childIntID`, `schDate` (`MM/DD/YYYY`), `dayType` | `todayScheduleInfo` | ✅ |
| `GetStudentClasesForGivenDayResponse` | same as above — the wire name really carries the `Response` suffix (and Edupoint's `Clases` typo) | `todayScheduleInfo` | ✅ |
| `GetStudentClassTime` | `childIntID` | `studentClassNow` | ✅ |
| `Gradebook` | `reportPeriod` (index from `reportingPeriods[].index`), `concurrentSchOrgYearGU` (from StudentClassList), `childIntID`, `languageCode` | `traditionalGradebook`, `standardsGradebook` | ✅ |
| `GetStudentAttendanceList` | `childIntID` | `dailyAttendance`, `periodAttendance` | ✅ |
| `GetStudentPastAttendanceData` | `childIntID` | `reportPastAtteendanceXML` *(sic)* | ✅ |
| `GetStudentInfoData` | `childIntID` | `studentInfoXML`, `studentInfoDetailXML` | ✅ |
| `GetCalendarData` | `childIntID`, optional date-window fields | `calendarListingData` | ✅ |
| `GetCalendarAssignmentDetails` | calendar event fields (`AGU`, `DGU`, `dguInternal`, `dgU2`, `viewType`, `addLinkData`, `dayType`) | `calendarAssignmentDetails` | ✅ |
| `GetStudentDocuments` | `childIntID`, `languageCode` | `studentDocuments` | ✅ |
| `GetStudentDocumentContent` | `childIntID`, `documentGU` | `studentAttachedDocumentData` (base64 PDF inline) | ◦ |
| `GetStudentHWNotes` | `childIntID`, `gu` | `gbhwNotesDatas` | ✅ |
| `UpdateStudentHWNotes` | `gbhwNotesUpdateData` | — | ◦ |
| `UpdateStudentGBLastCheckTime` | — | — | ◦ |
| `GetPXPContentMessage` | `childIntID` | `pxpMessagesData` | ✅ |
| `GetUserDefinedModule` | `childIntID`, `moduleIndex` | `allModuleRecordData` | ✅ |
| `GetFlexScheduleData` | `childIntID`, date fields | `studentFlexScheduleListingXML` | ◦ |
| `UpdateFlexSchedule` | `removeStudentFromSection=false`, section fields | — | ◦ |
| `GetSchoolInformationData` | `childIntID` | `studentSchoolInfoListing` | ✅ |
| `GetSchoolPayUrl` | — | payment-portal URL | ◦ |
| `GetSchoologoResponse` | *(none)* | `schoolAndDistrictLogo` (base64 images) | ✅ |
| `GetSoundFile` / `SaveSoundFile` | sound id / data | `soundFileData` | ◦ |
| `GenerateAuthToken` | *(session)* | `authToken` (SSO token for portal web-views) | ◦ |
| `GetAckData` | `processActivation=false` | `acctResult`, `ackStatmentData`, `parentOrStudentData` | ◦ |
| `GetAcknowledgementsData` | `childIntID` | `parentAcknowledgementMain` | ✅ |
| `GetAcknowledgementDetailsMobile` | acknowledgement id | `acknowledgement` | ◦ |
| `UpdateAckFromMyAccount` / `UpdateParentAcknowledgement` | `ackUpdateListing` | — | ◦ |
| `UpdateEmergencyResponse` | emergency contact answers | — | ◦ |
| `UpdateStudentAbsence` | `absenceReportListings`, `absenceReportPastList`, `reportingOption` | `requestResult` | ◦ |
| `UpdateAttachPhotoResponse` | `photoAttachDocumentData` | — | ◦ |
| `UpdateMyAccountData` | `pxpMobileUpdateMyAccountData` | — | ◦ |
| `GetContentMyAccountData` | `childIntID` | `pxpMyAccountData` | ✅ |
| `GetStudentDisciplineData` | `childIntID` | `studentDisciplineListing` | ◦ |
| `GetStudentFeeData` | `childIntID` | `studentFeeData` | ◦ |
| `GetStudentSpecialEdData` | `childIntID` | `specialEdData` | ◦ |
| `GetConferenceData` | `childIntID` | `studentConferenceData` | ◦ |
| `GetParentTeacherConferenceResponse` | conference fields | `conferenceDataList` | ◦ |
| `GetStudentsVideoMeeting` / `GetParentsVideoMeeting` | meeting fields | `meetingsForUserResponse`, `videoCallResponseModel` | ◦ |
| `GetHallPassData` | `getOnlyScheduedPases=false` *(sic)* | `studentHallPassXML` | ◦ |
| `GetHallPassHistory` | `onlyOverTimeLimit=false` | `passesData`, `statsData`, `summaryData` | ◦ |
| `GetHallPassSetup` | `childIntID` | `hallPassRoomSetupXML` | ✅ |
| `UpdateHallPass` | pass action fields | `studentHallPassXML` | ◦ |
| `GetHealthData` | `getDatahealthConditions=false`, `getDatahealthImmunizations=false`, `getDatahealthVisits=false` | `studentHealthData` | ◦ |
| `GetCounselorVisitBasicData` | `getPositionInLine=false` | `counselorVisiNthInLine` | ◦ |
| `GetLunchOrderInfoResponse` | `childIntID`, date | `lunchOrdersForStudent`, `lunchOrdersListForSchool`, `validSchoolDays` | ◦ |
| `UpdateLunchOrderInfoResponse` | `studentLunchOrder` | `result` | ◦ |
| `GetOLRInitialDataResponse` | `childIntID` | `olrInitialData` | ◦ |
| `GetOLRDocumentDownloadResponse` | `delete=false`, doc fields | `olrDocumentData` | ◦ |
| `UploadOLRDocumentUploadResponse` | `olrUploadDocumentData` | — | ◦ |
| `GetDownloadDocumentForSignatureResponse` | doc fields | `document` | ◦ |
| `UpdateDocumentSignatureResponse` | `docViewed=false`, `signPicDocumentData` | — | ◦ |
| `UploadDocumentForParentsResponse` | `studentDocumentUploadData` | — | ◦ |
| `UploadGBDocumentDataForStudentAssigment` | `gbDocumentDataObj` | — | ◦ |
| `UpdateDeviceToken` | push token, `reactNativeApp=false` | — (registers device for push) | ◦ |
| `UpdateNotificationPrefs` | `mobileUserSettings`, `notificationListing` | — | ◦ |
| `TestSystemCall` | *(none)* | connectivity smoke test (`data` is a bool) | ✅ |
| `GetSynergyMailGetConversations` | `pageToLoad=0` | `conversations`, `isLastPageLoaded`, `totalUnreadConversationMessages` | ◦ |
| `GetSynergyMailMessage` | message id fields | `synergyMailDataXML` | ◦ |
| `GetSynergyMailIGetMessageBody` | message id fields | `synergyMailMessageBodyXML` | ◦ |
| `GetSynergyMailGetAttachment` | attachment id fields | `attachmentXML` | ◦ |
| `GetSynergyMailInboxCount` | folder fields | `synergyMailInboxCountXML` | ◦ |
| `GetSynergyMailUnreadCount` | — | `messageCount` | ✅ |
| `GetSynergyMailGetContactList` | — | `contactGroupList` | ◦ |
| `GetSynergyMailGetSchoolList` | — | `organizationList` | ◦ |
| `GetSynergyMailGetStaffList` | — | `organizationStaffList` | ◦ |
| `GetSynergyMailGetStudentList` | — | `organizationStudentList` | ◦ |
| `GetSynergyMailGetTeacherList` | — | `studentClassScheduleForAllTerms` | ◦ |
| `GetSynergyMailRecipientSearch` | search fields | `studentCounselorInfo`, `studentGroupInfoDatas`, `studentInfoList` | ◦ |
| `GetSynergyMailRecipientAddressing` | `to`, `cc`, `bcc` | `to`, `cc`, `bcc`, `invalidRecipients` | ◦ |
| `GetSynergyMailSaveNewMessage` | `synergyEmailListing` | — | ◦ |
| `GetSynergyMailSaveReadOrDeleteMsg` | `synergyEmailMarkList` | — | ◦ |
| `GetSynergyMailMoveMessage` | message/folder fields | `synergyEmailSuccessMessage` | ◦ |
| `GetSynergyMailUpdateFolder` | `delete=false`, folder fields | `folderListViewXML` | ◦ |
| `GetSynergyMailUpdateSignatures` | `synergyMailSignatureXML` | — | ◦ |
| `GetForgotPasswordEmailToken` | account fields | `twoFactorTokenData` | ◦ |
| `ForceChangePassword` | `isStudent=false`, `oldApp=false`, credentials | `pxpForceChangePasswordResult` | ◦ |

\* SAML/SSO re-entry only.

### Examples
[Top](#TOC)

#### Child list

```json
{"arguments":{"request":"{\"legacyAppRequest\":false,\"secondaryLogin\":false}"}}
```
→
```json
{"error": null, "data": {"children": {
  "childInfos": [
    {"childIntID": 0, "studentGU": "<GUID>", "name": "<Student Name>", "schoolName": "<School Name>", "...": "..."}
  ],
  "allModules": [
    {"name": "Synergy Mail", "module": "21", "isEnabled": "Y",
     "iconUrl": "Images/PXP/ModuleIcons/icon_Messages.png", "moduleUrl": "PXP2_Messages.aspx", "moduleOrder": 0.0,
     "pxpModuleCfgGU": null, "organizationYearGU": null},
    {"name": "Attendance", "module": "0", "isEnabled": "Y",
     "iconUrl": "Images/PXP/ModuleIcons/icon_Attendance.png", "moduleUrl": "PXP2_Attendance.aspx", "moduleOrder": 5.0,
     "pxpModuleCfgGU": "<GUID>", "organizationYearGU": ""},
    {"name": "Grade Book", "module": "7", "isEnabled": "Y", "...": "..."}
  ]}}}
```

The `allModules` list is the district's enabled modules — useful to decide which calls to attempt.

Child selection is a parameter (`childIntID`), not a stateful step — there is no "select child" call. `0` is the logged-in student on StudentVUE. Don't confuse `childIntID` (request ordinal) with `studentGU` (response GUID).

#### Class schedule

```json
{"arguments":{"request":"{\"childIntID\":0,\"loadAllTerms\":true,\"conSchOrgYearGU\":\"\",\"conSchTermIndex\":\"-1\",\"termIndex\":\"-1\"}"}}
```
→ `data.studentClassScheduleForAllTerms` (trimmed):
```json
{
  "termIndex": 0,
  "termIndexName": "Semester 1",
  "termLists": [
    {"termIndex": 0, "termCode": 1, "termName": "Semester 1",
     "beginDate": "MM/DD/YYYY", "endDate": "MM/DD/YYYY",
     "schoolYearTrmCodeGU": "<GUID>", "schoolName": "<School Name>",
     "orgYearGU": "<GUID>", "termDefCodes": [{"termDefName": "S1"}]}
  ],
  "studentClassScheduleForTerms": [
    {"thisTermIndex": 0, "beginDate": "MM/DD/YYYY", "endDate": "MM/DD/YYYY",
     "orgYearGU": "<GUID>",
     "classLists": [
       {"period": "1", "courseTitle": "<Course Title>", "roomName": "<Room>",
        "teacher": "<Teacher Name>", "teacherEmail": "<teacher@example.com>",
        "sectionGU": "<GUID>", "teacherStaffGU": "<GUID>",
        "meetingDays": null, "excludePVUE": false,
        "additionalStaffInformation": [], "additionalStaffInformationXMLs": []}
     ]}
  ],
  "concurrentSchoolStudentClassScheduleForAllTermss": []
}
```

With `loadAllTerms: true` classes live under `studentClassScheduleForAllTerms.studentClassScheduleForTerms[].classLists[]`; the single-term mode (`loadAllTerms: false`) returns a flat `classLists` under `studentClassSchedule`. Handle both. `orgYearGU` here is the value `Gradebook` wants as `concurrentSchOrgYearGU`.

The single-term response also carries **`todayScheduleInfoData`** — see [Today's classes](#todays-classes) for the populated shape.

#### Today's classes

Today's bell-time schedule comes from two places, and they are not equivalent.

**`StudentClassList` with `loadAllTerms: false`** — the single-term response (see [Class schedule](#class-schedule)) includes **`todayScheduleInfoData`**, the populated shape:

```json
{"todayScheduleInfoData": {"date": "M/D/YYYY", "schoolInfos": [
  {"schoolName": "<School Name>", "bellSchedName": "",
   "classes": [
     {"period": "NN", "className": "<Course Title> - <Section ID>",
      "startTime": "H:MM AM", "endTime": "H:MM AM",
      "startDate": "MM/DD/YYYY H:MM:SS AM", "endDate": "MM/DD/YYYY H:MM:SS AM",
      "roomName": "<Room>",
      "teacherName": "<Teacher Name>", "teacherEmail": "<teacher@example.com>",
      "staffGU": "<GUID>", "sectionGU": "<GUID>",
      "emailSubject": "RE: Period NN, Section <Section ID>",
      "teacherURL": "<HTML snippet with SMApp.composeEx(...) mailto handlers>",
      "classURL": "", "attendanceCode": "", "hideClassStartEndTime": false}
   ]}]}}
```

`teacherURL` is an HTML fragment embedding JavaScript `SMApp.composeEx({to: [{RecipientList: 0, GU: "<GUID>"}], subject: "…", messageText: ""})` calls for Synergy Mail compose — `staffGU`/`emailSubject` carry the same data in usable form. Use this as the source for bell times.

**`GetStudentClasesForGivenDay` / `GetStudentClasesForGivenDayResponse`** — dedicated standalone calls (the app calls the `Response`-suffixed name):

```json
{"arguments":{"request":"{\"childIntID\":0,\"schDate\":\"MM/DD/YYYY\",\"dayType\":0}"}}
```
→ `data.todayScheduleInfo`:
```json
{"date": "M/D/YYYY", "dateToLoad": "YYYY-MM-DDT00:00:00-07:00",
 "attendance": null, "schools": []}
```

Reproducibly `attendance: null, schools: []` mid-class on a regular school day — across both method names, both `dayType` values (normal/alternate flex schedules), and arbitrary `schDate` values; the response always echoes the current day, so `schDate` appears to be ignored. Treat empty `schools` as "no data", not "no school". The `date` fields use `M/D/YYYY` (no zero-padding), while `dateToLoad` is ISO-8601 with the local UTC offset.

#### Current class

```json
{"arguments":{"request":"{\"childIntID\":0}"}}
```
→ `data.studentClassNow` (trimmed):
```json
{
  "period": "N", "courseTitle": "<Course Title>", "sectionID": "<Section ID>",
  "room": "<Room>", "staffName": "<Teacher Name>",
  "startTime": null, "endtime": null,
  "minutesRemainingToEndClass": "N", "minutesAfterClassStart": "NN",
  "resultCode": 1,
  "currentClass": "Period: N,  Course Title: <Course Title>,  Section ID: <Section ID>, Room: <Room>,  Staff Name: <Teacher Name>,  All Day Code: Present"
}
```

`currentClass` is a pre-rendered human-readable summary (it embeds today's attendance code); build your own UI from the individual fields, not by parsing it. `startTime`/`endtime` stay `null` even mid-class; the minute counters are strings.

Outside school hours every class field is `null` and `resultCode` != 0 (4 = nothing found) with a message in `currentClass`, e.g. `"For the current day, no Schedule Information was found for the selected student."`

#### Gradebook

```json
{"arguments":{"request":"{\"reportPeriod\":\"\",\"concurrentSchOrgYearGU\":\"\",\"childIntID\":0,\"languageCode\":\"en\"}"}}
```
→ `data.traditionalGradebook` (trimmed):
```json
{
  "type": "Traditional",
  "errorMessage": null,
  "reportingPeriods": [
    {"index": "0", "gradePeriod": "S1", "startDate": "MM/DD/YYYY", "endDate": "MM/DD/YYYY"}
  ],
  "reportingPeriod": {"...": "..."},
  "courses": [
    {"period": "1", "title": "<Course Title>", "courseName": "…", "courseID": "…",
     "room": "<Room>", "staff": "<Teacher Name>", "staffEMail": "<teacher@example.com>",
     "staffGU": "<GUID>", "imageType": null,
     "highlightPercentageCutOffForProgressBar": 50, "usesRichContent": false,
     "marks": [
       {"markName": "S1MT", "shortMarkName": "S1MT",
        "calculatedScoreString": "<grade>", "calculatedScoreRaw": "88.5",
        "standardViews": [],
        "gradeCalculationSummary": [
          {"type": "Classwork", "weight": "50%", "points": "14.00", "pointsPossible": "14.00",
           "weightedPct": "50.00%", "calculatedMark": "<grade>"},
          {"type": "Assessment", "weight": "50%", "points": "0.00", "pointsPossible": "0.00",
           "weightedPct": "0.00%", "calculatedMark": "0"},
          {"type": "TOTAL", "weight": "100%", "points": "14.00", "pointsPossible": "14.00",
           "weightedPct": "100.00%", "calculatedMark": "<grade>"}
        ],
        "assignments": [
          {"gradebookID": 0, "measure": "<Assignment Title>", "type": "Classwork",
           "date": "M/D/YYYY", "dueDate": "M/D/YYYY",
           "score": "3", "displayScore": "3 out of 3", "scoreCalValue": "3", "scoreMaxValue": "3",
           "scoreType": "Raw Score", "points": "3 / 3", "point": "3", "pointPossible": "3",
           "timeSincePost": "Nd", "totalSecondsSincePost": 200000.0,
           "notes": "", "measureDescription": "",
           "teacherID": 0, "studentID": 0,
           "hasDropBox": false, "dropStartDate": "M/D/YYYY", "dropEndDate": "M/D/YYYY",
           "resources": [], "standards": []}
        ],
        "assignmentsSinceLastAccess": []}
     ]}
  ],
  "standardsGradebook": null
}
```

Assignments are inline in `courses[].marks[].assignments[]` — no second fetch. The quarter is a request parameter: valid `reportPeriod` values come from `reportingPeriods[].index`, so a full year takes one call per quarter. `standardsGradebook` is a second mode some districts use. `assignmentsSinceLastAccess` reflects per-account read state tracked by the server.

Field quirks: mark names can be mid-term markers (`S1MT`), not just final-term codes; `calculatedScoreRaw` is a string; `totalSecondsSincePost` is a float; all score fields are duplicated in several forms (`score`/`displayScore`/`points`/`point`…) with `scoreType` distinguishing `Raw Score` from letter/percent grades; `gradebookID` here is the same id the calendar uses as `DGU`. Numeric lookups (`gradebookID`, `teacherID`, `studentID`) are numbers, not strings.

If a mark has no weighted categories `gradeCalculationSummary` is a list with just the `TOTAL` row.

#### Attendance

```json
{"arguments":{"request":"{\"childIntID\":0}"}}
```
→ `data` keys `dailyAttendance` and `periodAttendance` (summary/entry objects). `dailyAttendance` can be `null` with `periodAttendance` carrying everything:

```json
{"periodAttendance": {
  "type": "Period", "startPeriod": 0, "endPeriod": NN, "periodCount": NN,
  "schoolName": "<School Name>",
  "absences": [
    {"absenceDate": "MM/DD/YYYY", "reason": "<Reason>", "reason2": null,
     "note": "<Parent note text>", "dailyIconName": "icon_excused.gif",
     "codeAllDayReasonType": "icon_excused.gif", "codeAllDayDescription": "<Reason>",
     "periods": [
       {"number": "N", "name": "", "note": "", "reason": "",
        "course": "<Course Title>", "staff": "<Teacher Name>",
        "staffEMail": "<teacher@example.com>", "iconName": "",
        "schoolName": "<School Name>", "staffGU": "<GUID>", "orgYearGU": "<GUID>"}
     ]}
  ]}}
```

Whole-year history is a separate call, `GetStudentPastAttendanceData` (response key `reportPastAtteendanceXML` — Edupoint's spelling); schools that don't expose it return `error.code "400"`.

#### Student info

```json
{"arguments":{"request":"{\"childIntID\":0}"}}
```
→ `data.studentInfoXML` (photo, schedule summary) and `data.studentInfoDetailXML` — demographics, contacts, guardian info. Both are JSON objects despite the `XML` names. Contains PII.

#### Calendar

```json
{"arguments":{"request":"{\"childIntID\":0}"}}
```
→ `data.calendarListingData` (trimmed):
```json
{
  "schoolBegDate": "MM/DD/YYYY", "schoolEndDate": "MM/DD/YYYY",
  "monthBegDate": "MM/DD/YYYY", "monthEndDate": "MM/DD/YYYY",
  "eventLists": [
    {"date": "MM/DD/YYYY", "title": "First Day of the School", "icon": null,
     "AGU": null, "dayType": 0, "startTime": "", "link": null,
     "DGU": null, "dguInternal": null, "dgU2": null,
     "viewType": null, "addLinkData": null, "evtDescription": null},
    {"date": "MM/DD/YYYY", "title": "Holiday", "icon": null,
     "AGU": null, "dayType": 1, "startTime": "All Day", "link": null,
     "DGU": null, "dguInternal": null, "dgU2": null,
     "viewType": null, "addLinkData": null, "evtDescription": null},
    {"date": "MM/DD/YYYY",
     "title": "<Teacher Name>  <Course Title>(<period>) : <Assignment Title>  - Score: 100.00",
     "icon": "assignment.png",
     "AGU": "0", "dayType": 2, "startTime": "", "link": "ASSIGNMENTS",
     "DGU": "0", "dguInternal": "0", "dgU2": null,
     "viewType": "2", "addLinkData": "<GUID>", "evtDescription": null}
  ]
}
```

Called with just `childIntID` the server returns the current month window. Mid-semester the list mixes three event kinds — school events, holidays, and gradebook assignments (`period` inside a title is the class period). `dayType` doubles as an event-kind discriminator here: `0` plain school event, `1` no-school day, `2` assignment. On assignment events `DGU`/`dguInternal` is the gradebook assignment id (equal to `Gradebook`'s `assignments[].gradebookID`) and `addLinkData` is the section `GUID` — the inputs for `GetCalendarAssignmentDetails`. Assignment `title`s already embed the score; there is no separate score field.

Detail fetch: `GetCalendarAssignmentDetails`, passing the event's fields through:

```json
{"arguments":{"request":"{\"childIntID\":0,\"AGU\":\"0\",\"DGU\":\"0\",\"dguInternal\":\"0\",\"dgU2\":null,\"viewType\":\"2\",\"addLinkData\":\"<GUID>\",\"dayType\":2}"}}
```
→ `data.calendarAssignmentDetails`:
```json
{
  "classGU": null, "assignment": null,
  "hideStandardGraphInd": false, "hideMarksColumnElementary": false,
  "hidePointsColumnElementary": false, "displayStandardsData": false,
  "assignmentEventDetailLists": []
}
```

The container comes back with `assignmentEventDetailLists: []` for past assignments; the exact input combination that populates it is still unknown.

#### Documents

List: `GetStudentDocuments` with `{"childIntID":0,"languageCode":"en"}` → `data.studentDocuments`:

```json
{
  "showDateColumn": true, "showDocNameColumn": true, "showDocCatColumn": true,
  "studentGU": "<GUID>", "studentSSY": "<GUID>",
  "changesPending": false, "pendingChangeDate": null, "pendingChangesMessage": null,
  "studentDocumentDatas": [
    {"documentGU": "<GUID>", "documentFileName": "<GUID>.pdf",
     "documentDate": "MM/DD/YYYY", "documentType": "Report Card",
     "studentGU": "<GUID>", "documentComment": "<comment>"}
  ],
  "documentSetupDatas": {"...": "..."}
}
```

Fetch: `GetStudentDocumentContent` with `{"childIntID":0,"documentGU":"<GUID>"}` → `data.studentAttachedDocumentData.documentDatas[]`:

```json
{"documentGU": "<GUID>", "studentGU": "<GUID>", "docDate": "MM/DD/YYYY",
 "fileName": "<GUID>.pdf", "category": "Report Card", "notes": "<comment>",
 "docType": "Report Card", "base64Code": "JVBERi0xLjQKJ…",
 "gbStudentID": null, "gbTeacherID": null, "gbGradeBookID": null}
```

The PDF is inline base64 in `base64Code` (decodes to `%PDF-…`) — no URL, no second host. `documentType` values include: Report Card, Unofficial Transcript, Course History, Test Report, Parent Letter, attendance letters, surveys. `documentComment` carries the period label (e.g. `<School Year> Sem 1 Final Mark`). The `gb*` fields suggest gradebook attachments ride the same envelope.

Note `documentFileName` is a server-side GUID-named file, not the human title — the display name comes from `documentType`/`documentComment`.

#### Homework / class notes

```json
{"arguments":{"request":"{\"childIntID\":0,\"gu\":\"\"}"}}
```
→ `data.gbhwNotesDatas`:
```json
{"studentGU": "<GUID>", "sisNumber": "<id>", "studentSSY": "<GUID>", "gbHomeWorkNotesRecords": []}
```

`gu` selects a section (GUID from the schedule); empty returns the container only. Writing is `UpdateStudentHWNotes` with `gbhwNotesUpdateData`.

#### Messages

```json
{"arguments":{"request":"{\"childIntID\":0}"}}
```
→ `data.pxpMessagesData` (trimmed):
```json
{
  "supportingSynergyMail": true,
  "messageListings": [],
  "synergyMailMessageListingByStudents": [
    {"studentGU": "<GUID>",
     "synergyMailMessageListings": [
       {"attachmentDatas": [], "iconURL": "images/PXP/", "ID": null,
        "beginDate": "MM/DD/YYYY 00:00:00", "type": 1,
        "subject": "<Message Subject>", "content": null, "endDate": null,
        "read": false, "deletable": true, "from": null,
        "subjectNoHTML": "<Message Subject>",
        "module": 7, "email": null, "staffGU": null, "smMsgPersonGU": null}
     ]}]}
```

District/school alert messages. `messageListings` is the legacy shape (empty here); live messages arrive under `synergyMailMessageListingByStudents[]`, grouped per student. `module` identifies the emitting module (`7` = gradebook progress-report notices, `1` = attendance notes). `ID`/`from`/`content` can all be `null` for auto-generated notices. Replaces the SOAP `GetPXPMessages`; message attachments are fetched through the mail family below.

#### School information

```json
{"arguments":{"request":"{\"childIntID\":0}"}}
```
→ `data.studentSchoolInfoListing` (trimmed):
```json
{
  "school": "<School Name>", "principal": "<Principal Name>",
  "schoolAddress": "<Street Address>", "schoolAddress2": "",
  "schoolCity": "<City>", "schoolState": "<ST>", "schoolZip": "<ZIP>",
  "phone": "<phone>", "phone2": "<phone>", "URL": "<school website>",
  "principalEmail": "<principal@example.edu>", "principalGu": "<GUID>",
  "staffLists": [
    {"name": "<Staff Name>", "eMail": "<staff@example.edu>",
     "title": "HS TEACHER", "phone": "", "extn": "", "staffGU": "<GUID>"}
  ]}
```

A full staff directory — hundreds of entries at a large school, all teachers included with their `staffGU` (matching the schedule's `teacherStaffGU`). Titles are raw Synergy position codes (`HS TEACHER`, `HS SECRETARY OFFICE`, `HS SPED PARAEDUC`, …).

#### My account

`GetContentMyAccountData` with `{"childIntID":0}` → `data.pxpMyAccountData` — the account holder's contact/notification settings. Contains the student's name, address, phone and e-mail (all PII — handle accordingly); field groups:

- identity/contact: `formattedName`, `userID` (the student id string), `homeAddress`/`mailAddreess` (*sic*, `<br>`-separated HTML), `phoneNumbers` (HTML with a `*` primary marker), `eMail`…`eMail5` plus per-address `Enabled`/`Visible` flags.
- self-edit permissions: `canEditParentDemographicData`, `enableParent*Updates`, `firstNameUpdatePermission`, etc.
- notification toggles: `chkNotify*` flags each paired with a `chkNotify*Enabled` flag and `showNotification*Email`/`SMS`/`Voice` variants.
- misc: `hidePaperlessReportcard`, `paperlessReportcardSetting`, `supporting_Email_Notification_ForStudentVUE`, `adultID`.

#### Acknowledgements

```json
{"arguments":{"request":"{\"childIntID\":0}"}}
```
→ `data.parentAcknowledgementMain`:
```json
{"parentAcknowledgementDatas": []}
```

Empty outside signature windows.

#### Hall pass setup

```json
{"arguments":{"request":"{\"childIntID\":0}"}}
```
→ `data.hallPassRoomSetupXML` (a JSON object despite the name):
```json
{"hallPassMaxDaysForFuturePass": "NN", "hallPassCountUp": false,
 "allowStudentsToEndHallPass": false, "hallPassRoomTypes": []}
```

Returned even where `GetHallPassData` fails with the generic `500` error; empty `hallPassRoomTypes` means the feature has no rooms configured.

#### Student mail (Synergy Mail)

The whole mail stack is a `GetSynergyMail*` / `UpdateSynergyMail*` family (see the [Method catalog](#method-catalog)). Inbox paging:

```json
{"arguments":{"request":"{\"pageToLoad\":0}"}}
```
→
```json
{"error": null, "data": {"conversations": {"...": "..."},
  "isLastPageLoaded": false, "totalUnreadConversationMessages": 0}}
```

Compose: `GetSynergyMailRecipientSearch` / `GetSynergyMailRecipientAddressing` → `GetSynergyMailSaveNewMessage`. Read/delete: `GetSynergyMailSaveReadOrDeleteMsg`; move: `GetSynergyMailMoveMessage`; folders: `GetSynergyMailUpdateFolder`; signatures: `GetSynergyMailUpdateSignatures`. Unread counters: `GetSynergyMailUnreadCount` (→ `data.messageCount`, an int):

```json
{"error": null, "data": {"messageCount": 1}}
```

`GetSynergyMailInboxCount` and the address-book calls (`GetSynergyMailGetTeacherList`, `…GetSchoolList`, …) can return the generic `500` error even with mail enabled and unread conversations — don't assume the whole family works because `GetSynergyMailUnreadCount` does. Address books: teachers also come from the schedule (`GetSynergyMailGetTeacherList` returns `studentClassScheduleForAllTerms`).

#### Hall passes and health

Hall passes (district-dependent, often disabled): `GetHallPassSetup`, `GetHallPassData` (`getOnlyScheduedPases` — *sic*), `GetHallPassHistory` (`onlyOverTimeLimit`), `UpdateHallPass`. Health: `GetHealthData` with three booleans — `getDatahealthVisits`, `getDatahealthImmunizations`, `getDatahealthConditions` — returning `studentHealthData` with the requested sections; districts gate this heavily.
