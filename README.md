# Conpsirare
# DonorPerfect Airtable Integration Workflow

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

---

## Error Handling

If any unexpected error occurs during processing:

1. Catch the exception.
2. Update Airtable:

   * Status = `FAILED`
   * Message = Error text
   * Match Count = `0`
   * Last Sync Attempt = Current Timestamp
