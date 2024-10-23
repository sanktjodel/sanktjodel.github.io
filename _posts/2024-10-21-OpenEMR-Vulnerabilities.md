---
layout: post
title: OpenEMR Security Vulnerabilities
---

During research on [OpenEMR](https://github.com/openemr/openemr), a free and open source electronic health records and medical practice management application, I discovered the following authorization and cross-site scripting (XSS) security vulnerabilities resulting in 11 CVEs:

- CVE-2024-46021 [Stored XSS in MedEx Username](#CVE-2024-46021)
- CVE-2024-46022 [Stored XSS in Forms Administration Nicknames](#CVE-2024-46022)
- CVE-2024-46023 [Stored XSS in Patient Flow Board Report](#CVE-2024-46023)
- CVE-2024-46024 [Stored XSS in Telephone Country Code](#CVE-2024-46024)
- CVE-2024-46025 [Stored XSS in Portal Mail / Secure Messaging](#CVE-2024-46025)
- CVE-2024-46026 [Stored XSS in Facility Phone](#CVE-2024-46026)
- CVE-2024-46027 [Lack of Vertical Authorization in Carecoordination Setup](#CVE-2024-46027) / [Lack of Vertical Authorization in View and Edit of ACLs of Modules](#CVE-2024-46027B)
- CVE-2024-46029 [Lack of Horizontal Authorization in the Patient Portal](#CVE-2024-46029)
- CVE-2024-46030 [Lack of Vertical Authorization in View Facilities](#CVE-2024-46030)
- CVE-2024-46031 [Lack of Vertical Authorization in Patient Reports](#CVE-2024-46031)
- CVE-2024-46032 [Lack of Vertical Authorization in Carecoordination ](#CVE-2024-46032)

All issues have been fixed in [version 7.0.2 Patch 1](https://community.open-emr.org/t/openemr-7-0-2-patch-1-has-been-released/22695).

---



### <a id="CVE-2024-46021"></a>Stored XSS in MedEx Username and Unauthenticated Access to MedEx Preferences (CVE-2024-46021)

#### Description

The MedEx Username is vulnerable to a stored cross-site scripting (XSS) vulnerability. An administrator can submit arbitrary values in the username field. The preferences page does not perform output encoding when displaying the MedEx username. 
Note that the UI prevents submitting invalid email addresses. This can be bypassed by crafting a POST request or intercepting and modifying a POST request to `/interface/main/messages/save.php?MedEx=start`. 

Additionally to the XSS, once MedEx has been enabled, the page containing the stored XSS is also exposed without authentication at `http://openemr/interface/main/messages/messages.php?go=setup&stage=2`.

#### Reproduction Steps

1. Login as an administrator and then browse to Admin -> Config -> Connectors.

2. Enable MedEx by checking the box for "Enable MedEx Communication Service" and click Save.

3. Refresh/open the Messages tab.

4. Navigate to the Messages tab's new sub-menu: File->Setup MedEx.

5. Fill out the form fields. Before submitting the form, intercept the POST request to `/interface/main/messages/save.php?MedEx=start` and replace the `new_email` parameter with `test%40example.com<svg/onload=alert('XSS-in-medex-email')>`.

```
POST /interface/main/messages/save.php?MedEx=start HTTP/1.1
Host: openemr3
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
Content-Length: 170
Cookie: OpenEMR=ygYkNg6-%2C5vwwg19IBTGO3wMtcrCFMSCtWotEMPaDfMrctyK

new_email=test%40example.com<svg/onload=alert('XSS-medex-email')>&new_password=1q2w3e4r!Q%40W%23E%24R&new_rpassword=1q2w3e4r!Q%40W%23E%24R&TERMS_yes=on&BusAgree_yes=on
```

Browse to the Messages tab and select File->Preferences and notice the JavaScript popup.

Note that the API may return an error (`Username/Password mismatch`) if submitting multiple times. Try a different email address in that case.

6. Open a new private/incognito browser window and browse to `http://openemr/interface/main/messages/messages.php?go=setup&stage=2`. Note that you can access the page without authentication and the JavaScript alert popup.

#### Location 

<https://github.com/openemr/openemr/blob/5a7dfb8cfe4c072ce37c43cd5fde4880e5527764/library/MedEx/API.php#L1668>



### <a id="CVE-2024-46022"></a>Stored Cross-Site Scripting (XSS) in Forms Administration Nicknames (triggered in Manage Modules) (CVE-2024-46022)

#### Description

The Nickname fields in the Forms Administration (Admin -> Forms -> Forms Administration) are vulnerable to stored XSS. 
An administrator can modify the Nickname of Care Plan for example (`MyCarePlanNickname<svg/onload=alert('CarePlanNickname')>`) so that the JavaScript triggers when a user browses to the Manage Modules section (Modules -> Manage Modules -> Carecoordination -> Config).

#### Reproduction Steps

1. Login as an administrator and browse to the Forms Administration (Admin -> Forms -> Forms Administration). Enter the following in the Nickname field of Care Plan: `MyCarePlanNickname<svg/onload=alert('CarePlanNickname')>` and click Save.

2. Browse to the Manage Modules section (Modules -> Manage Modules -> Carecoordination -> Config) and notice the JavaScript alert popup.

#### Location

<https://github.com/openemr/openemr/blob/5f4307231e72b8c267a67e1773bc4af121b90fff/interface/modules/zend_modules/module/Carecoordination/view/carecoordination/setup/index.phtml#L138>




### <a id="CVE-2024-46023"></a>Stored Cross-Site Scripting (XSS) in Patient Flow Board Report (CVE-2024-46023)

#### Description

The Patient Flow Board Report (Reports -> Visits -> Patient Flow Board) is vulnerable to stored XSS. The vulnerable fields are the Provider's first and last names. 
An OpenEMR user that can edit users (Admin -> Users) can change the "First Name" and "Last Name" fields to contain JavaScript (e.g. `user<svg/onload=alert('xss')>`) that will execute when the victim browses to the Patient Flow Board Report and the search results include patients that have this provider assigned. 
Exploiting this vulnerability requires the privileges to edit users (Admin -> Users). The victim where the JavaScript triggers can be any user (e.g. Front Office) that has the privilege to view the Patient Flow Board.

#### Reproduction Steps

1. Create a new user or modify an existing OpenEMR user (Admin -> Users) with the following First Name: `user<svg/onload=alert('xss')>`

2. Make sure that this user is assigned as the Provider (Care Team) of a patient. This can be achieved by selecting a patient (via Finder), browse to the patient's Demographics, select the Choices tab, click the edit button and add the patient to the Care Team (Provider) field.

3. Browse to the Patient Flow Board Report (Reports -> Visits -> Patient Flow Board). Change the From and To fields to the last few days so that you will get results. Click the Submit button and notice the JavaScript alert popup, showing that arbitrary JavaScript can be run in the victim's OpenEMR session.

#### Location

<https://github.com/openemr/openemr/blob/995e8779312a94faad2a1486343604fca86040c7/interface/reports/patient_flow_board_report.php#L473>




### <a id="CVE-2024-46024"></a>Stored Cross-Site Scripting (XSS) in Telephone Country Code (CVE-2024-46024)

#### Description

The Telephone Country Code global settings field is vulnerable to stored XSS. An administrator can add the following JavaScript to the Telephone Country Code in the global settings (Admin -> Globals -> Locale -> Telephone Country Code): `1';alert('phone');//`. 

The JavaScript from the Telephone Country Code form triggers on the following pages:
- Add New Event: `http://openemr/interface/main/calendar/add_edit_event.php`
- Order Results: `http://openemr/interface/orders/single_order_results.php`
- Patient Portal Results: `http://openemr/interface/orders/single_order_results.php`

#### Reproduction Steps

1. As an administrator, add the following JavaScript to the Telephone Country Code in the global settings (Admin -> Globals -> Locale -> Telephone Country Code): `1';alert(1);//`. 
   Note that the UI restricts the input field length to 15 characters. This is a client side restriction only. This can be easily bypassed to submit any input length for example by using the browser's web developer tools or by manually crafting a POST request to `/interface/super/edit_globals.php`.

2. Browse to any of the following pages to trigger the JavaScript:
- Add New Event: `http://openemr/interface/main/calendar/add_edit_event.php`
- Order Results: `http://openemr/interface/orders/single_order_results.php`
- Patient Portal Results: `http://openemr/interface/orders/single_order_results.php`

#### Location

- <https://github.com/openemr/openemr/blob/17db3fa29058ddc1c19236c7a8cadca4f2145ec5/interface/main/calendar/add_edit_event.php#L1013>
- <https://github.com/openemr/openemr/blob/420e6ed57c98dfec4a084d08e879fbc41aba7b90/interface/orders/single_order_results.inc.php#L439>
- <https://github.com/openemr/openemr/blob/995e8779312a94faad2a1486343604fca86040c7/portal/report/portal_patient_report.php#L227>




### <a id="CVE-2024-46025"></a>Stored Cross-Site Scripting (XSS) in Portal Mail / Secure Messaging (CVE-2024-46025)

#### Description

The Secure Messaging functionality can be exploited by a low privileged portal user (patient) to perform XSS attacks on privileged OpenEMR users such as Administrators or Clinicians. The vulnerability is in the body of the message (`inputBody` parameter). See the reproduction steps below to create a POST request to `/portal/messaging/handle_note.php` that includes a sample JavaScript payload. The vulnerability triggers when a user views the message in the Portal Mail / Secure Messaging functionality (e.g. at `http://<openemr>/portal/messaging/messages.php`). 

#### Reproduction Steps

Modify the following POST request to create a message that includes a JavaScript `alert()` in the message body. Change the following values:

- Change the `PortalOpenEMR` cookie value to a valid session cookie of a portal user.
- Update the `csrf_token_form` to a valid CSRF token for this user.
- Change the `recipient_id`, `recipient_name`, `sender_id`, and `sender_name` parameters to valid users.

```
POST /portal/messaging/handle_note.php HTTP/1.1
Host: openemr
Content-Type: application/x-www-form-urlencoded
Content-Length: 402
Cookie: PortalOpenEMR=G4MCbtpsdHNRAJxdLXo8dWaFSlVbajZDxcHl-zJJ1QVHjzaw

title=subject:XSStest&csrf_token_form=938a0bc17c9d424edf1a988351b74090145cdf82&noteid=&replyid=&recipient_id=user2&recipient_name=user2&sender_id=John1&sender_name=John+Doe&task=add&inputBody=%3Cp%3EtestXSS:<img%20src=1%20onerror=alert('XSS')>%3C%2Fp%3E%0D%0A&pid=&submit=messages.php
```

Alternatively, intercept a POST request to `/portal/messaging/handle_note.php` when sending a new message in Secure Messaging and add `<img%20src=1%20onerror=alert('XSS')>` to the `inputBody` parameter in the body of the POST request. 

Notice that the JavaScript `alert()` triggers immediately when the sender or recipient of the message browses to the Secure Mail functionality. 

#### Impact

An attacker can exploit cross-site scripting to perform privileged actions in the context of a victim's session.
This stored cross-site scripting (XSS) vulnerability can be exploited by low privileged portal users to perform targeted attacks on privileged users such as Administrators or Clinicians through the Secure Messaging functionality.

#### Location

<https://github.com/openemr/openemr/blob/db73b557f63b0dbb7cb75cff933bcdec1a6e1605/portal/messaging/messages.php#L332>



### <a id="CVE-2024-46026"></a>Stored Cross-Site Scripting (XSS) in Facility Phone (CVE-2024-46026)

#### Description

The Facility phone number field is vulnerable to XSS. A user with the privilege to edit Facilities can add malicious code to the phone number field which is triggered on the patient portal custom report page (e.g. `http://openemr/portal/report/portal_custom_report.php?printable=1`).

#### Reproduction Steps

1. As an administrator, add the following to the phone number field of a Facility (Admin -> Clinic -> Facilities -> Phone): `<svg/onload=alert('FacPhone')>`.

2. Add the Facility with the malicious phone number from step 1 to a new or existing patient. This can be achieved by selecting a patient (via Finder), browse to the patient's Demographics, select the Choices tab, click the edit button and add the Facility to this patient.

3. Login to the patient portal with the patient user assigned to this Facility and browse to the portal custom report page: `http://openemr/portal/report/portal_custom_report.php?printable=1`. Notice the JavaScript alert popup, showing that arbitrary JavaScript can be run in the victim's OpenEMR session.

#### Location

<https://github.com/openemr/openemr/blob/995e8779312a94faad2a1486343604fca86040c7/portal/report/portal_custom_report.php#L584>




### <a id="CVE-2024-46027"></a>Lack of Vertical Authorization in Carecoordination Setup (CVE-2024-46027)

#### Description

The Mapper tab in the Carecoordination Setup pages (Modules -> Manage Modules -> Carecoordination Config -> Mapper) lack authorization checks and allow any OpenEMR user to view and edit Carecoordination configurations.

#### Reproduction Steps

1. Login as an administrator and browse to the Mapper tab in the Carecoordination Setup pages (Modules -> Manage Modules -> Carecoordination Config -> Mapper). Notice that administrators can view and edit Carecoordination configurations.

2. Login as a lower privileged user (e.g. Front Office) that should not have read access to the Carecoordination Setup pages. Note that the low privileged user does not have a menu entry for Modules.

3. Browse to `http://openemr/interface/modules/zend_modules/public/carecoordination/setup` and notice that the low privileged user can view the Mapper tab of the Carecoordination Setup pages.

#### Location

<https://github.com/openemr/openemr/blob/6552c4e897986417584491e994e77293ff4b3ab2/interface/modules/zend_modules/module/Carecoordination/src/Carecoordination/Controller/SetupController.php#L59>




### <a id="CVE-2024-46027B"></a>Lack of Vertical Authorization in View and Edit of ACLs of Modules (CVE-2024-46027)

#### Description

The Access Control pages of Modules such as Carecoordination (Modules -> Manage Modules -> Carecoordination Config) do not perform authorization controls allowing any OpenEMR user to view and edit Access Control Lists (ACLs) of certain components. 
The ACL page can be accessed at `http://openemr/interface/modules/zend_modules/public/acl/acl?module_id=5`. This allows lower privileged users such as Front Office users that should not have access to the Manage Modules pages to view and edit ACLs to potentially expand their privileges.

#### Reproduction Steps

1. Login as an administrator and browse to the Access Control pages of the Carecoordination Module (Modules -> Manage Modules -> Carecoordination Config). Notice that administrators can access the Notice that administrators can view and edit Carecoordination configurations.

2. Login as a lower privileged user (e.g. Front Office) and notice that this user does not have a menu entry for Modules and therefore should not have the privileges to manage Modules.

3. As the low privileged user, browse to the following URL and notice that they can view and edit Access Control Lists (ACLs) of the Carecoordination Module: `http://openemr/interface/modules/zend_modules/public/acl/acl?module_id=5`.

#### Location

<https://github.com/openemr/openemr/blob/5f4307231e72b8c267a67e1773bc4af121b90fff/interface/modules/zend_modules/module/Acl/view/acl/acl/acl.phtml>




### <a id="CVE-2024-46029"></a>Lack of Horizontal Authorization in the Patient Portal Leaks Patient Names (CVE-2024-46029)

#### Description

The <https://github.com/openemr/openemr/blob/c1c0805696ca68577c37bf30e29f90e5f3e0f1a9/portal/add_edit_event_user.php> PHP script does not perform authorization controls and allows authenticated patient portal users to view other patient names. 
This can be achieved by browsing to `http://openemr/portal/add_edit_event_user.php?pid=1` and incrementing the pid URL parameter.

#### Reproduction Steps

1. Login to the patient portal.

2. Browse to the following URL: `http://openemr/portal/add_edit_event_user.php?pid=1`

3. Increment the `pid` URL parameter and notice that you can view all patient portal user first and last names.

#### Location

<https://github.com/openemr/openemr/blob/c1c0805696ca68577c37bf30e29f90e5f3e0f1a9/portal/add_edit_event_user.php>



### <a id="CVE-2024-46030"></a>Lack of Vertical Authorization in View Facilities (CVE-2024-46030)

#### Description

The Facility admin PHP script (<https://github.com/openemr/openemr/blob/5ba868ea40d4215725dc9eabd6ca930804ae8f60/interface/usergroup/facility_admin.php>) lacks authorization checks and allows any authenticated OpenEMR user to view all Facilities by enumerating the fid URL parameter. 
Other Facility PHP pages check if the user is an administrator using `AclMain::aclCheckCore('admin', 'users')`. The Facility admin PHP script lacks such authorization checks. 
This allows any OpenEMR user, including lower privileged users such as Front Office, to view all Facilities by browsing to `http://openemr/interface/usergroup/facility_admin.php?fid=4`.

#### Reproduction Steps

1. Login as an administrator and browse to the Facilities administration page (Admin -> Clinic -> Facilities), e.g. at `http://openemr/interface/usergroup/facilities.php`.

2. Login as a lower privileged user (e.g. Front Office) and note that this user is not authorized to view or edit Facilities (`http://openemr/interface/usergroup/facilities.php`).

3. Browse to the following URL as a low privileged user (e.g. Front Office) and notice that they can view and enumerate all Facilities by increasing the fid URL parameter: `http://openemr/interface/usergroup/facility_admin.php?fid=4`.

#### Location

<https://github.com/openemr/openemr/blob/5ba868ea40d4215725dc9eabd6ca930804ae8f60/interface/usergroup/facility_admin.php>



### <a id="CVE-2024-46031"></a>Lack of Vertical Authorization in Patient Reports (Continuity of Care Record (CCR) & Continuity of Care Document (CCD)) (CVE-2024-46031)

#### Description

The Continuity of Care Record (CCR) and Continuity of Care Document (CCD) patient reports lack authorization checks. This allows any authenticated OpenEMR user to view patient details of any patient, including lower privileged users (e.g. Front Office) that should not have read access to patients CCR and CCD reports. 
The CCR and CCD patient reports are implemented in the <https://github.com/openemr/openemr/blob/995e8779312a94faad2a1486343604fca86040c7/ccr/createCCR.php> PHP script.

#### Reproduction Steps

1. Login as an administrator and browse to the patient reports page: Finder -> [SELECT A PATIENT] -> Report [TAB]. Click 'Generate Report' under 'Continuity of Care Record (CCR)' or 'Continuity of Care Document (CCD)'.

2. Note that administrators can generate and view patient reports.

3. Login as a lower privileged user (e.g. Front Office) that should not have read access to patients CCR and CCD reports.

4. Open a patient (Finder -> [SELECT A PATIENT]) and note that there is no Report tab, showing that this user is not supposed to have access to patient reports.

5. Create the following POST request to `/ccr/createCCR.php`. Change the OpenEMR session cookie value to a valid cookie for the low privileged user and notice that the server successfully responds with the patient data of the currently selected patient:

```
POST /ccr/createCCR.php HTTP/1.1
Host: openemr
Content-Type: application/x-www-form-urlencoded
Content-Length: 37
Cookie: OpenEMR=rTXIYgDUg94tLYNy8jKrePcWmN9hWoAhDa%2CzRNvlNg1mLu4%2C

ccrAction=generate&raw=no&Start=&End=
```

Alternatively to crafting your own POST request, intercept the POST request to `/ccr/createCCR.php` from an authorized user and replace the OpenEMR session cookie with the value of a lower privileged user.

#### Location

<https://github.com/openemr/openemr/blob/995e8779312a94faad2a1486343604fca86040c7/ccr/createCCR.php>



### <a id="CVE-2024-46032"></a>Lack of Vertical Authorization in Carecoordination Uploaded Content Exposure (CVE-2024-46032)

#### Description

The Carecoordination Module (Modules -> Carecoordination -> Import) lacks authorization controls and allows any OpenEMR user to view the content of imported (uploaded) Consolidated Clinical Document Architecture (CCDA) or Quality Reporting Document Architecture (QRDA) files. 
These files can contain patient information that low privileged users such as Front Office users are not supposed to have access to. Note that lower privileged users can't import or download CCDA or QRDA files, they can only view the content of already uploaded files through the user interface.

#### Reproduction Steps

1. Login as an administrator and browse to the Carecoordination Module (Modules -> Carecoordination -> Import).

2. Import a sample CCDA file (e.g. <https://github.com/chb/sample_ccdas/blob/master/Allscripts%20Samples/Enterprise%20EHR/b2%20Adam%20Everyman%20ToC.xml>) to populate some data.

3. Notice that administrators can view the content of imported (uploaded) Consolidated Clinical Document Architecture (CCDA) or Quality Reporting Document Architecture (QRDA) files.

4. Login as a lower privileged user (e.g. Front Office) that should not have read access to the imported CCDA or QRDA content.

5. Browse to `http://openemr/interface/modules/zend_modules/public/carecoordination/upload` and notice that the low privileged user can view imported CCDA or QRDA content.

#### Location

<https://github.com/openemr/openemr/blob/b454fb61669072ee7a0e40331bd5d932ba088744/interface/modules/zend_modules/module/Carecoordination/view/carecoordination/carecoordination/upload.phtml>


