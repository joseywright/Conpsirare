# Conpsirare
# DonorPerfect Airtable Integration Workflow

# FormSubmissionDPAPIInterface Script 
## 1. Receive Input from Airtable

Retrieve the following values from the Airtable record:

* Record ID
* First Name
* Last Name
* Email
* Zip Code
* RSVP Response

Set the event name:

* `Test Event`

---

## 2. Prepare Data

* Escape apostrophes in all text fields to prevent DonorPerfect query errors.
* Define required configuration values:

  * DonorPerfect API Endpoint
  * API Key
  * User ID (`AirtableIntegration`)

---

## 3. Determine Activity Code

Based on the RSVP response:

| RSVP Value | Activity Code         |
| ---------- | --------------------- |
| Yes        | `TEST_EVENT_RSVP_YES` |
| No         | `TEST_EVENT_RSVP_NO`  |

If the RSVP value is anything other than **Yes** or **No**:

* Stop processing.
* Update Airtable:

  * Status = `FAILED`

---

## 4. Search for Donors

### Search #1: Existing Donor Search

Perform a SQL search in DonorPerfect using:

* Email
* Last Name

**If a donor is found:**

* Store the Donor ID.
* Continue to Contact Creation.

---

### Search #2: Secondary Existing Donor Search

If Search #1 returns no donor:

Perform a SQL search using:

* Email
* Last Name
* First Name

**If a donor is found:**

* Store the Donor ID.
* Continue to Contact Creation.

---

### Search #3: Fallback Donor Search

If both SQL searches fail:

Call:

`dp_donorsearch`

Using:

* First Name
* Last Name
* Zip Code

#### Possible Results

##### Multiple Matches Found

* Record the number of matches.
* Update Airtable:

  * Status = `REVIEW`
  * Message = `"Multiple donor matches found. Manual review required."`
* Stop processing.

##### One Match Found

* Store the Donor ID.
* Continue processing.

##### No Matches Found

* Proceed to donor creation.

---

## 5. Create New Donor

If no donor was found through any search:

Call:

`dp_savedonor`

Create a new donor with:

* First Name
* Last Name
* Email
* Zip Code
* Country = `US`

Additional default values:

* Organization Record = `N`
* Donor Type = `IN`
* No Mail = `N`
* Receipt Type = `C`
* Narrative = `"Created via Airtable API Integration"`

### If Creation Fails

Update Airtable:

* Status = `FAILED`
* Message contains the DonorPerfect response.

Stop processing.

### If Creation Succeeds

* Save the returned Donor ID.

---

## 6. Create Contact Record

Build the contact comment:

```text
Event: Test Event | RSVP: [Yes/No]
```

Set the contact date to today's date.

Call:

`dp_savecontact`

Using:

* Contact ID = `0` (new contact)
* Donor ID
* Activity Code
* Contact Date
* Comment
* User ID

Required null fields:

* `due_date`
* `due_time`
* `completed_date`
* `document_path`

---

## 7. Validate Contact Creation

### If Contact Creation Fails

Update Airtable:

* Status = `FAILED`
* Message contains the raw DonorPerfect response.

Stop processing.

### If Contact Creation Succeeds

* Store the Contact ID.

---

## 8. Update Airtable Status

### If a New Donor Was Created

Update Airtable:

* Status = `SUCCESS`
* Message:

```text
New donor created and contact record added.
Donor ID: [ID]
```

### If an Existing Donor Was Found

Update Airtable:

* Status = `SUCCESS`
* Message:

```text
Existing donor matched and contact record added.
Donor ID: [ID]
```

Also store:

* Match Count (if applicable)
* Timestamp of Sync Attempt
* Last Synched RSVP field  (for RecordUpdateDPAPIInterface script to find)
* Last Synched Attended field (for RecordUpdateDPAPIInterface script to find)

---

## Error Handling

If any unexpected error occurs during processing:

1. Catch the exception.
2. Update Airtable:

   * Status = `FAILED`
   * Message = Error text
   * Match Count = `0`
   * Last Sync Attempt = Current Timestamp


# RecordUpdateDPAPIInterface Script

## 1. Receive Input from Airtable

Retrieve the following values from the Airtable record:

* Record ID
* First Name
* Last Name
* Email
* Zip Code
* RSVP Response
* Attended Event
* Last Synced RSVP
* Last Synced Attendance

Set the event name:

* `Test Event`

---

## 2. Prepare Data

* Escape apostrophes in all text fields to prevent DonorPerfect query errors.
* Define required configuration values:

  * DonorPerfect API Endpoint
  * API Key
  * User ID (`AirtableIntegration`)
* Connect to the Airtable table:

  * `Form Responses`

---

## 3. Determine Whether a Sync Is Required

The script compares current values to previously synced values.

### RSVP Changed

If the RSVP value differs from `Last Synced RSVP`:

| RSVP Value | Activity Code         |
| ---------- | --------------------- |
| Yes        | `TEST_EVENT_RSVP_YES` |
| No         | `TEST_EVENT_RSVP_NO`  |

Create a contact comment:

```text
Event: Test Event | RSVP: [Value]
```

---

### Attendance Marked Yes

If:

```text
Attended Event = Yes
AND
Last Synced Attendance ≠ Yes
```

Create:

| Activity Code         |
| --------------------- |
| `TEST_EVENT_ATTENDED` |

Create a contact comment:

```text
Event: Test Event | Attended
```

---

### Attendance Marked No

If:

```text
Attended Event = No
AND
Last Synced Attendance ≠ No
```

The script:

* Updates `Last Synced Attendance` to `No`
* Does not create a DonorPerfect contact record
* Marks the sync as successful

---

### No Changes Detected

If neither RSVP nor Attendance require syncing:

* No DonorPerfect API call is made
* Sync status is updated to:

```text
SUCCESS
No changes requiring DonorPerfect sync.
```

---

## 4. Search for a Matching Donor

### Primary Match

Search DonorPerfect using SQL:

```text
Email + Last Name
```

---

### Secondary Match

If no donor is found:

```text
Email + Last Name + First Name
```

---

### Fallback Match

If SQL searches fail, use DonorPerfect's standard donor search:

```text
First Name + Last Name + Zip Code
```

---

### Match Outcomes

#### Single Match Found

Proceed with contact creation.

---

#### Multiple Matches Found

Update Airtable:

```text
DP Sync Status = REVIEW
```

Message:

```text
Multiple donor matches found.
Manual review required.
```

---

#### No Match Found

Update Airtable:

```text
DP Sync Status = REVIEW
```

Message:

```text
No matching donor found in DonorPerfect.
```

---

## 5. Create DonorPerfect Contact Record

If a donor match is found, create a new contact record using:

```text
action=dp_savecontact
```

Fields submitted:

| Field            | Value                    |
| ---------------- | ------------------------ |
| `contact_id`     | `0`                      |
| `donor_id`       | Matched donor            |
| `activity_code`  | Determined activity code |
| `mailing_code`   | `null`                   |
| `by_whom`        | `AirtableIntegration`    |
| `contact_date`   | Today's Date             |
| `due_date`       | `null`                   |
| `due_time`       | `null`                   |
| `completed_date` | `null`                   |
| `document_path`  | `null`                   |
| `comment`        | Event comment text       |
| `user_id`        | `AirtableIntegration`    |

---

## 6. Verify Contact Creation

The script examines the DonorPerfect response for a valid Contact ID.

### Contact Created Successfully

Continue processing.

---

### Contact Creation Failed

Update Airtable:

```text
DP Sync Status = FAILED
```

Store the full DonorPerfect response in:

```text
DP Sync Message
```

This allows troubleshooting of API errors.

---

## 7. Update Airtable Sync Tracking Fields

### RSVP Sync

If an RSVP activity was created:

```text
Last Synced RSVP = Current RSVP
```

---

### Attendance Sync

If an attendance activity was created:

```text
Last Synced Attendance = Yes
```

---

## 8. Record Success Status

Update Airtable:

```text
DP Sync Status = SUCCESS
```

Example message:

```text
Contact record added successfully.
Donor ID: 12345
Activity Code: TEST_EVENT_RSVP_YES
```

The script also records:

```text
Last Sync Attempt
DP Match Count
```

when applicable.

---

## 9. Error Handling

Any unexpected error is caught and written back to Airtable.

Updates:

```text
DP Sync Status = FAILED
DP Sync Message = [Error Details]
DP Match Count = 0
Last Sync Attempt = [Timestamp]
```

This ensures all failures are visible directly from the Airtable record.
