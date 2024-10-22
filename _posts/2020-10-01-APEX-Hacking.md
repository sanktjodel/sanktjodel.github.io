---
layout: post
title:  Security Testing Oracle APEX Applications
---

This is a penetration tester's guide on how to test and review Oracle APEX applications.

Oracle Application Express (APEX) is a web-based software development environment that runs on an Oracle database. 
Its main benefit is to allow to build web applications rapidly based on an Oracle database. 
APEX has some unique security features and constraints. 
The goal of this document is to allow a security tester to get familiar with the peculiarities of APEX quickly and show what testing should focus on. 
The focus on this document is on a blackbox engagement, although there are some sections with information that are only relevant if you have access to the APEX workspace.

## Identifying APEX & Understanding the APEX URL Syntax

Oracle APEX web applications can be easily identified by their URL format. This is how an APEX URL looks like: 

```
http://apex.example.com/pls/apex/f?p=4350:1:120883407765693447
```

The URL syntax supports the following arguments: `f?p=AppId:PageId:Session:Request:Debug:ClearCache:itemNames:itemValues:PrinterFriendly&cs=`

- `pls` indicates the use of the Oracle HTTP Server with mod_plsql. This can be missing if the APEX Listener or Embedded PL/SQL Gateway is used.
- `apex` is the Database Access Descriptor (DAD) name. The DAD describes how the HTTP Server connects to the database server. The default value is apex.
- `f?p=` is a prefix used by Oracle Application Express to route the request to the correct engine process. `f` is a public procedure and is the main entry point for APEX.
- `AppId` (e.g. 4350) is the ID of the application being called. The application ID is a unique number that identifies each application. This can also be a name instead of a numeric ID.
- The `PageId` (e.g. 1) is the number of the page within the application. This can also be a page name instead of a numeric ID.
- The `Session` (e.g. 120883407765693447) is the session number. When you run an application, the Application Express engine generates a session number that serves as a key to the user's session state. This can be 0 for Public Pages or empty (then APEX creates a new Session).
- `Request` – a Request Keyword
- `Debug` – set to YES (uppercase!) switches on the Debug-Mode
- `ClearCache` – you can put a page id or a list of page ids here (comma-separated) to clear the cache for these pages (set session state to null, ...)
- `itemNames` – comma separated list of page-item names. In a regular web application this would be `?itemName1=itemValue1&itemName2=itemValue2`
- `itemValues` – comma separated list of values. Each value is assigned to the corresponding parameter provided in itemNames (first value assigned to first parameter, second value assigned to second parameter, and so on). You can't specify values containing either a comma (`,`) or a colon (`:`). Both would lead to side-effects and errors, as APEX gets confused when parsing the URL. Using a comma works, if enclosed by slashes: e.g. `\123,89\`.
- `PrinterFriendly` – set to YES (uppercase!) switches the page into PrinterFriendly-Mode, uses the Printerfriendly template to render the Page.
- `cs` - If Session State Protection (SSP) is enabled there may be a checksum at the end which you have to calculate for example by using `:w
apex_util.prepare_url()`.

See <https://docs.oracle.com/database/apex-18.1/HTMDB/understanding-url-syntax.htm#HTMDB03017> for the documentation of the URL syntax.

The `f` procedure will then call `wwv_flow.show`. For example, the URL
`http://host/apex/f?p=123:1:12345678963470::::P1_ITEM:MYVALUE`
becomes

```
http://host/apex/wwv_flow.show?
  p_flow_id=123                 Application ID
  &p_flow_step_id=1             Page ID
  &p_instance=12345678963470    Session ID
  &p_arg_name=P1_ITEM           Item Name
  &p_arg_value=MYVALUE          Item Value
```

This is useful for performing SQL injection attacks.

All page rendering is handled by `wwv_flow.show` (GET requests), whereas all page processing is handled by `wwv_flow.accept` (POST requests). There is a third endpoint called `wwv_flow.ajax` for Asynchronous JavaScript and XML (AJAX) JavaScript calls. `wwv_flow.ajax` supports both GET and POST requests.

## Session Management

APEX applications use cookies to keep track of user sessions. Common names are `ORA_WWV_USER_{instanceId}` and `ORA_WWV_APP_{applicationId}`. Additionally, APEX uses session IDs to segregate one user session from another. That is, one user can have multiple session IDs. That session ID is included in almost every APEX URL. For example, the following URL shows the session ID 17421763018053:

`http://host/ords/f?p=4500:1000:17421763018053:::::`

Both the session ID and session cookie are required to access an APEX application. 
Having access to a session ID only or a session cookie only does not give access to the application. 
Both values are required. 
This session ID in the URL effectively prevents CSRF attacks. 
Internally, APEX maps the session ID (`ID`) to the cookie value (`COOKIE_VALUE`) in the `WWV_FLOW_SESSION$` table. 
Session state values are not stored in a cookie or in the browser. 

## Instance Hardening

### Runtime Mode

Production instances should be run in "Runtime Mode" which removes the APEX development environment and disables the APEX workspace login page.  This can be tested by trying to access the APEX workspace login page which is usually at `/apex` or `/pls/apex`:w
 (no slash at the end or it may not work).

## Enumerating pages

Pages have numeric values by default starting with 1. Pages can also have names assigned (alphanumeric). These pages still can be accessed by their numeric page ID. Discovering pages that are not linked from the application is therefore pretty simple and should be performed during a blackbox assessment. This allows to discover pages that are hidden or don’t have an authentication or authorization scheme assigned (i.e. accessible either unauthenticated or using any user).

## SQL Injection

APEX applications can be vulnerable to SQL injection as any other web application using a SQL based database. As best practice to prevent SQL injection, the bind variable syntax should be used. 

Testing for SQL injection is similar to any other web application: all input parameters need to be tested for potential injection using characters such as single quote (`'`) and double quote (`"`).

Exploiting SQLi using the `f?p` URL syntax can be tricky. Fortunately these requests can be rewritten using GET requests to `wwv_flow.show` as shown above in the APEX URL syntax section.

## XSS

Recent versions of APEX automatically HTML encode page items making reflected XSS rare. Stored XSS can still be possible and should be tested for the same way as any other web application (all input should be tested — forms, URL parameters, etc.).

## Session State Protection : Preventing (URL) Parameter Tampering

Session State Protection is a feature of APEX that prevents a user from changing URL parameters. This will add an additional checksum as part of
the URL (the `cs` URL parameter). The idea is to prevent URL tampering.

In APEX, it is common to have an URL in the format of `/pls/apex/f?p=123:28:104906907705719::::P1_EMPLOYEE_ID:4589`. If Session State Protection is not enabled, a user can change the employee ID (4589) to any value. Session State Protection prevents a user from changing arbitrary parameters. 

Session State Protection needs to enabled in the following three locations:

1. Enable Session State Protection at the application level
2. Each page that needs to be protected needs to be configured ("Page Access Protection")
3. Each item (Page -> Content Body -> Items -> [ITEM]) that needs to be protected needs to be configured ("Session State Protection")

To make things more complex, hidden items (items that are submitted in a hidden HTML form field) should be marked as "Value Protected" (in Settings) which will prevent them from being changed. This only applies to hidden form items submitted in an HTML form and not to (hidden) items that can be modified in the URL.

During a blackbox engagement, all parameters (e.g. in the URL, in forms, hidden form elements) should be tested if they can be changed or tampered with.

## Oracle APEX Information

### Versions

Releases: <https://en.wikipedia.org/w/index.php?title=Oracle_Application_Express&oldid=1131746440#Releases>

Make sure the APEX version is a recent one. The version can usually be discovered by looking at the HTML/JavaScript source code, for example:

~~~
gApexVersion !== "19.2.0.00.15"
~~~

### Testing APEX

Oracle has a free online test instance at apex.oracle.com where you can request access. Request a free workspace here: <https://apex.oracle.com/en/learn/getting-started/> Access is usually granted within a day or so.

Oracle also offers pre-built developer VMs (Virtual Box) at <https://www.oracle.com/downloads/developer-vm/community-downloads.html>. Choose the one containing "Oracle Application Express" called "Database App Development VM" (<https://www.oracle.com/database/technologies/databaseappdev-vm.html>).

## References

- Book: Expert Oracle Application Express Security (Author: Spendolini, Scott): <https://www.apress.com/gp/book/9781430247319>
- Book: Hands-On Oracle Application Express Security: Building Secure Apex Applications (Author: Recx/Tim Austwick): <https://www.wiley.com/en-us/Hands+On+Oracle+Application+Express+Security%3A+Building+Secure+Apex+Applications-p-9781118686133>


