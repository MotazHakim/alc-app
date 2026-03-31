# Database Schema & System Requirements Blueprint: Arab Labor Organization (ALO) Conference Management System

## 1. Project Context & Architecture Constraints
*   **Backend & Database:** The system will be powered by **PocketBase** (which uses an SQLite database under the hood). PocketBase will act as the unified backend handling the database operations, user authentication, role-based access, automated emails, and file storage. 
*   **Frontend & Development:** The frontend will be an internal web application built with the assistance of an Agentic IDE (e.g., GitHub Copilot, Cursor). This document serves as the master prompt/blueprint for the AI to generate the PocketBase collections and frontend interfaces.
*   **Scale & Volume:** The database will handle approximately **500 participants per yearly session**. Given this low volume, SQLite via PocketBase is highly optimal and complex performance indexing is not strictly required.
*   **Security & Compliance:** The application is strictly internal and accessible only via the organization's VPN. **No GDPR compliance or data anonymization is required.**
*   **Data Types Mapping (PocketBase Specifics):** 
    *   "Primary Keys" and "Foreign Keys" referenced below should be implemented using PocketBase's native `id` (15-character strings) and `relation` field types.
    *   PocketBase automatically handles `created` and `updated` timestamps, which partially satisfies the audit trail requirements natively.

***

## 2. Global System Rules
*   **Fresh Entries Per Session:** A physical person attending multiple yearly sessions will have a completely separate, fresh `Participant` record for each session. There is no global participant linking across different years.
*   **File Uploads:** All file/image uploads (passports, photos, QR codes, logos) must be stored via `rustfs S3` (integrated with PocketBase storage). The database will only store the **URL** (text) of the uploaded file.
*   **Localization:** All text data will be stored in Arabic. Database table and column names must be in English, with Arabic labels mapping to them in the frontend UI.
*   **Global Tracking (Audit Trails):** Every table/collection must include metadata for:
    *   `created_by` (Relation to Users collection - who created the record)
    *   `updated_by` (Relation to Users collection - who last edited the record)

***

## 3. Core Session & Session-Specific Tables

### `ALC_Sessions` (المؤتمرات)
The core table representing the yearly conference. Each session dictates its own unique resources.
*   `alc_id` (Primary Key) [required field]
*   `alc_number` (Number) - User-friendly identifier displayed in the UI. [required field]
*   `alc_country_id` (Relation -> `Cat1`) - Host country. [required field]
*   `alc_city` (Text) [required field]
*   `alc_hotel` (Text) [required field]
*   `alc_sponsor` (Text)
*   `alc_date` (Text) - Free text date representation (e.g., "من ١٥ حتي ١٧ مارس ٢٠٢٦"). [required field]
*   `alc_start_date` (Date - ISO YYYY-MM-DD) [required field] - Canonical session start date for programmatic queries and age calculations.
*   `alc_end_date` (Date - ISO YYYY-MM-DD) [required field] - Canonical session end date for programmatic queries.
*   `alc_site_url` (URL) [required field]
*   `alc_qrcode_url` (Text) - S3 URL.

### Session-Specific Resources
These tables are strictly tied to a specific `ALC_Session`.
*   **`Guest_Hotels` (فنادق الإقامة الإضافية):** 
        `hotel_id` (PK)
        `alc_id` (Relation)
        `hotel_name` (Text) [required field]
*   **`Cars` (السيارات):**
         `car_id` (PK)
         `alc_id` (Relation)
         `car_model` (Text) [required field]
*   **`Receptionists` (موظفي الاستقبال):**
         `receptionist_id` (PK)
         `alc_id` (Relation)
         `receptionist_name` (Text) [required field]
*   **`Tech_Committees` (اللجان الفنية):** 
         `committee_id` (PK)
         `alc_id` (Relation)
         `committee_index` (Number) - Strictly 1, 2, 3, or 4. (Max4 and we might add only 3 in some sessions) Used by the UI to dynamically rename the `tech_committee_X_role` columns in the participant table. [required field]
         `committee_name` (Text) [required field]

***

## 4. Reference & Categorization Tables

### Cascade Categorization (`Cat1` -> `Cat2` -> `Cat3`)
A 3-level strict cascade system defining Countries/Organizations, their sub-parties, and their respective institutions.

*   **`Cat1` (الدول / المنظمات - Countries/Organizations):**
        `cat1_id` (PK)
        `cat1_order` (Number) [required field]
        `cat1_name` (Text) [required field]
        `cat1_shortname` (Text) [required field]
        `cat1_nationality` (Text) [required field]
        `cat1_flag_url` (Text - S3 URL) [required field]
        `clearance_required` (Boolean) - Identifies if the country requires security clearance.
        `no_clearance_age` (Number) - Age threshold above which clearance is bypassed. Calculated by comparing the participant's birth year to the session `alc_start_date` year (age = YEAR(alc_start_date) - YEAR(birthday)). [Nullable]
*   **`Cat2` (أطراف الإنتاج - Production Parties):**
        `cat2_id` (PK)
        `cat1_id` (Relation -> `Cat1`) [required field]
        `cat2_order` (Number) [required field]
        `cat2_name` (Text) [required field]
    *   *Examples: Government (حكومات), Employers (أصحاب أعمال), Workers (عمال).*
*   **`Cat3` (المؤسسات - Institutions):**
        `cat3_id` (PK)
        `cat2_id` (Relation -> `Cat2`) [required field]
        `cat3_order` (Number) [required field]
        `cat3_name` (Text) [required field]
        `is_hidden` (Boolean) - Allows hiding entities from UI droplists.

### Other Reference Tables
*   **`Airlines`:** `airline_id` (PK)
        `airline_order` (Number) [required field]
        `airline_name_ar` (Text) [required field]
        `airline_name_en` (Text) [required field]
        `iata_code` (Text) [required field]
        `airline_logo_url` (Text - S3 URL) [required field]

***

## 5. Roles & Formations Tables

These tables define participant statuses. The frontend UI must filter available options based on the participant's `Cat2` selection using the `cat2_filter` column; `cat2_filter` stores Cat2 names (strings) such as 'حكومات', 'أصحاب أعمال', 'عمال'.

*   **`Conference_Roles` (صفة المشارك في المؤتمر):** 
        `role_id` (PK)
        `role_name` (Text) [required field]
        `cat2_filter` (Text/Array of valid Cat2 names). [required field]

*   **`Presidency_Formation` (تشكيل الرئاسة):** 
        `presidency_id` (PK)
        `presidency_name` (Text) [required field]
        `cat2_filter` (Text/Array). [required field]

*   **`Team_Formation` (تشكيل المجموعات):** 
        `team_id` (PK)
        `team_name` (Text) [required field]
        `cat2_filter` (Text/Array). [required field]

***

## 6. Main Participants Table (`Participants`)

**Identifiers & Basic Info:**
*   `participant_id` (Primary Key)
*   `alc_id` (Relation -> `ALC_Sessions`) [required field]
*   `cat1_id` (Relation -> `Cat1`) - Denormalized for direct querying. [required field]
*   `cat2_id` (Relation -> `Cat2`) - Denormalized for direct querying. [required field]
*   `cat3_id` (Relation -> `Cat3`) - Drives the cascade. [required field]
*   `order_number` (Number) NULLS LAST
*   `prefix` (Text) [required field]
*   `full_name` (Text) [required field]
*   `gender` (Select/Enum) - `['ذكر', 'أنثى']` [required field]
*   `job_title` (Text) [Nullable]
*   `conference_role_id` (Relation -> `Conference_Roles`)
*   `team_formation_id` (Relation -> `Team_Formation` - Nullable)
*   `presidency_formation_id` (Relation -> `Presidency_Formation` - Nullable)
*   `team_head` (Boolean)
*   `honor` (Boolean)
*   `special_request` (Text) - Free text description of requests. [Nullable]
*   `hosting` (Select/Enum) - `['غير مستضاف', 'من الدولة', 'من المنظمة', 'من الاتحاد']`. Default: `غير مستضاف`.
*   `note` (Text) [Nullable]
*   `participant_photo_url` (Text - S3 URL) [Nullable]

**Logistics (Hotel & Travel):**
*   `hotel_id` (Relation -> `Guest_Hotels` - Nullable)
*   `hotel_room_number` (Text), `hotel_room_type` (Text) [Nullable]
*   `car_id` (Relation -> `Cars` - Nullable)
*   `receptionist_id` (Relation -> `Receptionists` - Nullable)
*   `departure_airline_id` (Relation -> `Airlines` - Nullable)
*   `departure_date` (Date: dd/mm/yyyy) [Nullable]
*   `departure_time` (Text/Time) [Nullable]
*   `departure_flight_number` (Text) [Nullable]
*   `arrival_airline_id` (Relation -> `Airlines` - Nullable)
*   `arrival_date` (Date: dd/mm/yyyy) [Nullable]
*   `arrival_time` (Text/Time) [Nullable]
*   `arrival_flight_number` (Text) [Nullable]

**Passport & Security Details:**
*   `name_in_passport` (Text) [Nullable]
*   `mother_name` (Text) [Nullable]
*   `passport_number` (Text) [Nullable]
*   `passport_issue_date` (Date) [Nullable]
*   `passport_expire_date` (Date) [Nullable]
*   `birthday` (Date: dd/mm/yyyy) [Nullable]
*   `nationality_id` (Relation -> `Cat1`) [Nullable]
*   `passport_type` (Select/Enum) - `['عادي', 'خاص', 'دبلوماسي', 'مهمة', 'أمم متحدة أحمر', 'أمم متحدة ازرق', 'اتحاد اوروبي']` [Nullable]
*   `passport_copy_url` (Text - S3 URL) [Nullable]
*   `clearance_needed` (Boolean) - Manual override for special cases.
*   `sent_to_foreign_affairs` (Boolean)
*   `clearance_done` (Boolean)

**Badge & Speech Management:**
*   `badge` (Boolean) - Printed status.
*   `show_title_in_badge` (Boolean)
*   `speech_request_date` (Date: dd/mm/yyyy) [Nullable]
*   `speech_day` (Select/Enum) - `['لم يحدد', 'اليوم الأول', 'اليوم الثاني', 'اليوم الثالث', 'اليوم الرابع', 'اليوم الأول أو الثاني', 'اليوم الثاني أو الثالث']`. Default: `لم يحدد`.
*   `speech_order` (Number) [Nullable]

**Committee Roles:**
*The UI will map `tech_committee_X_role` to the specific committee name based on `Tech_Committees.committee_index`. Values must strictly conform to allowed roles.*
*   `tech_committee_1_role` (Text - Nullable)
*   `tech_committee_2_role` (Text - Nullable)
*   `tech_committee_3_role` (Text - Nullable)
*   `tech_committee_4_role` (Text - Nullable)
*   `org_committee_role` (Text - Nullable) - Role in اللجنة التنظيمية.
*   `draft_committee_role` (Text - Nullable) - Role in لجنة الصياغة.
*   `finance_committee_role` (Text - Nullable) - Role in اللجنة المالية.
*   `membership_committee_role` (Text - Nullable) - Role in لجنة اعتماد العضوية.

***
## Canonical Formats & Enums

- **Date/time formats:** All dates MUST be stored in ISO date format `YYYY-MM-DD` (example: 2026-03-30). Times MUST use `HH:MM` (24-hour). Use full ISO8601 datetimes `YYYY-MM-DDTHH:MM:SSZ` when timestamps are required.
- **Enums (exact Arabic strings):** Use PocketBase `select` fields with the following exact values.
  - `gender`: ['ذكر', 'أنثى']
  - `hosting`: ['غير مستضاف', 'من الدولة', 'من المنظمة', 'من الاتحاد']
  - `passport_type`: ['عادي', 'خاص', 'دبلوماسي', 'مهمة', 'أمم متحدة أحمر', 'أمم متحدة ازرق', 'اتحاد اوروبي']
  - `speech_day`: ['لم يحدد', 'اليوم الأول', 'اليوم الثاني', 'اليوم الثالث', 'اليوم الرابع', 'اليوم الأول أو الثاني', 'اليوم الثاني أو الثالث']
- **Cat2 filter:** `cat2_filter` stores Cat2 names (strings) (e.g., 'حكومات', 'أصحاب أعمال', 'عمال'); front-end must use canonical Cat2 names when filtering.
- **Tech committees:** `Tech_Committees.committee_index` MUST be 1..4. Sessions may define 3 or 4 committees; the UI should only expose `tech_committee_X_role` columns for indexes defined in a session. Keep `tech_committee_X_role` fields (X=1..4) on `Participants` for simplicity and fast querying.
- **Indexes & constraints:** Recommend indexes on `Participants.alc_id`, `Participants.cat1_id`, `Participants.cat2_id`, `Participants.arrival_date`, and `Participants.departure_date` to speed reporting. Set a unique constraint on `ALC_Sessions.alc_number` to avoid duplicate session numbers. Note: `passport_number` is NOT required to be unique per session or globally.
- **Files & storage:** Continue storing only S3 URLs (rustfs) in record fields. Enforce MIME type and size limits at upload time (policy/config in PocketBase hooks).

---

## 7. Audit System (`Audit_Log`)

Since PocketBase handles `created` and `updated` fields by default, this distinct collection is required specifically to track **field-level** diffs (who changed what specific value) to satisfy the strict audit requirement.
*   `log_id` (Primary Key)
*   `table_name` (Text)
*   `record_id` (Text) - The PocketBase ID of the impacted row.
*   `field_name` (Text) - The specific column updated.
*   `old_value` (Text)
*   `new_value` (Text)
*   `action_type` (Select/Enum) - `INSERT`, `UPDATE`, `DELETE`.
*   `changed_by` (Relation -> Users)
*   `changed_at` (Date/Time)

***

## 8. Backend Querying Rules for Committee Role Sorting

Because roles represent a strict hierarchy rather than an alphabetical order, standard sorting will fail. Since PocketBase uses SQLite, all API endpoints or frontend data parsing logic that queries participants by their committee roles **must** use an explicit `CASE` statement (or frontend array-mapping equivalent if filtering post-query) to enforce strict hierarchical weighting.

**Rule for Technical Committees Sorting:**
```sql
ORDER BY 
  CASE tech_committee_1_role
    WHEN 'رئيس اللجنة' THEN 1
    WHEN 'رئيس اللجنة / مقرر' THEN 2
    WHEN 'نائب رئيس اللجنة' THEN 3
    WHEN 'نائب رئيس اللجنة / مقرر' THEN 4
    WHEN 'مقرر' THEN 5
    WHEN 'أصيل' THEN 6
    WHEN 'مناوب' THEN 7
    WHEN 'مراقب' THEN 8
    ELSE 9
  END;
```

**Rule for Regular Committees Sorting:**
```sql
ORDER BY 
  CASE org_committee_role
    WHEN 'رئيس اللجنة' THEN 1
    WHEN 'رئيس اللجنة / مقرر' THEN 2
    WHEN 'نائب رئيس اللجنة' THEN 3
    WHEN 'نائب رئيس اللجنة / مقرر' THEN 4
    WHEN 'مقرر' THEN 5
    WHEN 'عضو' THEN 6
    ELSE 7
  END;
```

Below is an updated, full concrete lists for `Conference_Roles`, `Presidency_Formation`, and `Team_Formation`, plus a detailed section describing the required reports/queries and how they relate to the schema and fields.

***

## Conference Roles Definitions (`Conference_Roles`)

This table defines all possible “conference roles” (صفة المشارك في المؤتمر) a participant can have, with filters that control which roles are valid for each `Cat2` party.

**Table: `Conference_Roles`**

Columns:
- `role_id` (Primary Key)
- `role_name` (Text)
- `cat2_filter` (Text/Array of Cat2 names; used in UI filtering)

**Rows (complete list):**

For `Cat2 = حكومات` (Governments):
1. `role_id = 1`, `role_name = 'وزير'`, `cat2_filter = ['حكومات']`
2. `role_id = 2`, `role_name = 'وزير / مندوب'`, `cat2_filter = ['حكومات']`
3. `role_id = 3`, `role_name = 'رئيس وفد'`, `cat2_filter = ['حكومات']`
4. `role_id = 4`, `role_name = 'رئيس وفد / مندوب'`, `cat2_filter = ['حكومات']`

Shared roles for multiple parties:

5. `role_id = 5`, `role_name = 'مندوب'`, `cat2_filter = ['حكومات', 'أصحاب أعمال', 'عمال']`
6. `role_id = 6`, `role_name = 'مستشار'`, `cat2_filter = ['حكومات', 'أصحاب أعمال', 'عمال']`
7. `role_id = 7`, `role_name = 'مكرم'`, `cat2_filter = ['حكومات', 'أصحاب أعمال', 'عمال', 'منظمات']`
8. `role_id = 8`, `role_name = 'مرافق'`, `cat2_filter = ['حكومات', 'أصحاب أعمال', 'عمال', 'منظمات']`
9. `role_id = 9`, `role_name = 'اعتذار'`, `cat2_filter = ['حكومات', 'أصحاب أعمال', 'عمال', 'منظمات']`

Specific to organizations (`Cat2 = منظمات`):

10. `role_id = 10`, `role_name = 'مراقب'`, `cat2_filter = ['منظمات']`

This covers all valid role combinations:
- For `حكومات`: وزير، وزير / مندوب، رئيس وفد، رئيس وفد / مندوب، مندوب، مستشار، مكرم، مرافق، اعتذار.
- For `أصحاب أعمال` or `عمال`: مندوب، مستشار، مكرم، مرافق، اعتذار.
- For `منظمات`: مراقب، مكرم، مرافق، اعتذار.

***

## Presidency Formation Definitions (`Presidency_Formation`)

This table defines the available roles in the conference presidency structure (تشكيل رئاسة المؤتمر), filtered by `Cat2`.

**Table: `Presidency_Formation`**

Columns:
- `presidency_id` (Primary Key)
- `presidency_name` (Text)
- `cat2_filter` (Text/Array of Cat2 names)

**Rows (complete list as specified):**

1. `presidency_id = 1`, `presidency_name = 'رئيس المؤتمر'`, `cat2_filter = ['حكومات']`
2. `presidency_id = 2`, `presidency_name = 'نائب رئيس المؤتمر عن فريق أصحاب الاعمال'`, `cat2_filter = ['أصحاب أعمال']`
3. `presidency_id = 3`, `presidency_name = 'نائب رئيس المؤتمر عن فريق الحكومات'`, `cat2_filter = ['حكومات']`
4. `presidency_id = 4`, `presidency_name = 'نائب رئيس المؤتمر عن فريق العمال'`, `cat2_filter = ['عمال']`

The `Participants.presidency_formation_id` field references this table and is filtered by the participant’s `cat2_id`.

***

## Team Formation Definitions (`Team_Formation`)

This table defines the team structure per party (تشكيل الفرق), again filtered by `Cat2`.

**Table: `Team_Formation`**

Columns:
- `team_id` (Primary Key)
- `team_name` (Text)
- `cat2_filter` (Text/Array of Cat2 names)

**Rows (complete list as specified):**

For `حكومات`:
1. `team_id = 1`, `team_name = 'رئيس فريق الحكومات'`, `cat2_filter = ['حكومات']`
2. `team_id = 2`, `team_name = 'نائب رئيس فريق الحكومات'`, `cat2_filter = ['حكومات']`
3. `team_id = 3`, `team_name = 'مقرر فريق الحكومات'`, `cat2_filter = ['حكومات']`

For `أصحاب أعمال`:
4. `team_id = 4`, `team_name = 'رئيس فريق أصحاب الاعمال'`, `cat2_filter = ['أصحاب أعمال']`
5. `team_id = 5`, `team_name = 'نائب رئيس فريق أصحاب الاعمال'`, `cat2_filter = ['أصحاب أعمال']`
6. `team_id = 6`, `team_name = 'مقرر فريق أصحاب الاعمال'`, `cat2_filter = ['أصحاب أعمال']`

For `عمال`:
7. `team_id = 7`, `team_name = 'رئيس فريق العمال'`, `cat2_filter = ['عمال']`
8. `team_id = 8`, `team_name = 'نائب رئيس فريق العمال'`, `cat2_filter = ['عمال']`
9. `team_id = 9`, `team_name = 'مقرر فريق العمال'`, `cat2_filter = ['عمال']`

The `Participants.team_formation_id` field references this table according to `cat2_id`.

***

## Reporting & Query Requirements

All reporting logic is driven by the schema and fields described earlier. Unless otherwise stated, **all participant-based reports must be ordered** by:
1. `Cat1.cat1_order` (ascending), then
2. `Cat2.cat2_order` (ascending), then
3. `Cat3.cat3_order` (ascending), then
4. `Participants.order_number` (ascending).

Exceptions (team formation, presidency, committees, speech, and flights) use custom ordering rules described below.

### Report 1: Full Participants List per Session

**Goal:** List all participants in a specific `ALC_Session`, grouped and ordered by country and party.

**Filters:**
- `Participants.alc_id = :selected_session`

**Grouping/Ordering:**
- First by `Cat1.cat1_order` (group by country)
- Within each country, by `Cat2.cat2_order`
- Within each `Cat2`, order rows by `Participants.order_number`

**Columns to display:**
- `prefix`
- `full_name`
- `Cat3.cat3_name`
- `job_title`
- `Conference_Roles.role_name`

This report explains why `order_number`, `cat1_id`, `cat2_id`, `cat3_id`, and `conference_role_id` are all stored on the participant record.

***

### Report 2: Participants with Specific Conference Roles (مرافق / مكرم / اعتذار / arbitrary words)

**Goal:** Filter participants by conference role, supporting exact or “like” textual search.

**Filters:**
- By exact role: `Conference_Roles.role_name IN ('مرافق', 'مكرم', 'اعتذار', ...)`
- Or by substring: `Conference_Roles.role_name LIKE '%<search_text>%'`

**Ordering:**
- Use the standard Cat1/Cat2/Cat3/participant order.

This justifies having Conference Roles normalized and available for textual filtering.

***

### Report 3: Participants with Roles Like وزير or رئيس وفد

**Goal:** Find participants in leadership-type roles (e.g., any role containing 'وزير' or 'رئيس وفد').

**Filters (examples):**
- `Conference_Roles.role_name LIKE '%وزير%'`
- Or `Conference_Roles.role_name LIKE '%رئيس وفد%'`

**Ordering:**
- Standard Cat1/Cat2/Cat3/participant order.

***

### Report 4: Team Heads

**Goal:** List all team heads in a session.

**Filters:**
- `Participants.alc_id = :selected_session`
- `Participants.team_head = true`

**Ordering:**
- Standard Cat1/Cat2/Cat3/participant order.

This report uses the boolean `team_head` field.

***

### Report 5: Participants with Photos

**Goal:** List participants who have a profile photo uploaded.

**Filters:**
- `participant_photo_url IS NOT NULL` and not empty.

**Ordering:**
- Standard Cat1/Cat2/Cat3/participant order.

***

### Report 6: Participants by Country Range or Specific Country

**Goal:** Run queries by country ID range or specific IDs.

**Filters (examples):**
- Range: `cat1_id BETWEEN 1 AND 21`
- Single: `cat1_id = 22`

**Ordering:**
- Standard Cat1/Cat2/Cat3/participant order.

This directly uses the denormalized `cat1_id` on `Participants`.

***

### Report 7: Participants with Note or Special Request

**Goal:** Identify participants who have internal notes or special requests.

**Filters:**
- `(note IS NOT NULL AND note <> '')`
- OR `(special_request IS NOT NULL AND special_request <> '')`

**Ordering:**
- Standard Cat1/Cat2/Cat3/participant order.

This explains why `note` and `special_request` are stored as free text fields.

***

### Report 8: Badge Printing Status

**Goal:** List all participants by badge printing status.

**Filters:**
- Either `badge = true` or `badge = false` depending on the view.

**Ordering:**
- Standard Cat1/Cat2/Cat3/participant order.

This uses the `badge` boolean to drive printing workflows.

***

### Report 9: Country-Level Statistics by Party and Role

**Goal:** Produce a statistical summary per country, showing counts by party (حكومات / أصحاب أعمال / عمال) and by specific conference roles inside each party, plus counts of “مندوب”-like roles.

**Grouping:**
- Group by `Cat1.cat1_id` (country).

**Columns (example):**
For each country row:
- Under حكومات:
  - Count of participants where:
    - `cat2_name = 'حكومات'`
    - `Conference_Roles.role_name` in:
      - 'وزير'
      - 'وزير / مندوب'
      - 'رئيس وفد'
      - 'رئيس وفد / مندوب'
- Under أصحاب أعمال:
  - Counts for relevant roles (e.g., مندوب، مستشار، مكرم، مرافق، اعتذار) where `cat2_name = 'أصحاب أعمال'`.
- Under عمال:
  - Same counting logic where `cat2_name = 'عمال'`.

**Additional metric:**
- Count per country of participants whose role name contains 'مندوب' (any `Cat2`).

This is why the role names must be consistently stored and filterable.

***

### Report 10: Participants in Regular Committees (Organizing/Drafting/Finance/Membership)

**Goal:** List all participants in each regular committee, ordered by role hierarchy.

**Filters:**
- For organizing: `org_committee_role IS NOT NULL`
- For drafting: `draft_committee_role IS NOT NULL`
- For finance: `finance_committee_role IS NOT NULL`
- For membership: `membership_committee_role IS NOT NULL`

**Ordering:**
- First by **role hierarchy** using a fixed order:
  - 'رئيس اللجنة' → 1
  - 'رئيس اللجنة / مقرر' → 2
  - 'نائب رئيس اللجنة' → 3
  - 'نائب رئيس اللجنة / مقرر' → 4
  - 'مقرر' → 5
  - 'عضو' → 6
- Then by standard Cat1/Cat2/Cat3/participant order if needed.

**Columns:**
- Participant name
- Cat1 name
- Cat2 name
- Committee role

This uses the four regular-committee fields on `Participants`.

***

### Report 11: Participants in Technical Committees

**Goal:** List all participants in each technical committee, ordered by role hierarchy, excluding those with conference role `اعتذار` or `مرافق`.

**Filters:**
- Participant must have a non-null role in `tech_committee_1_role`, `tech_committee_2_role`, `tech_committee_3_role`, or `tech_committee_4_role`.
- Exclude:
  - `Conference_Roles.role_name = 'اعتذار'`
  - `Conference_Roles.role_name = 'مرافق'`

**Ordering per committee:**
- Use the technical-committee role hierarchy:
  - 'رئيس اللجنة' → 1
  - 'رئيس اللجنة / مقرر' → 2
  - 'نائب رئيس اللجنة' → 3
  - 'نائب رئيس اللجنة / مقرر' → 4
  - 'مقرر' → 5
  - 'أصيل' → 6
  - 'مناوب' → 7
  - 'مراقب' → 8

**Columns:**
- Participant name
- Cat1 name
- Cat2 name
- Committee role
- Committee index (to identify which tech committee, 1–4, mapped through `Tech_Committees`).

This justifies the four `tech_committee_X_role` fields and the fixed role order.

***

### Report 12: Arrival and Departure Flights

**Goal:** Manage logistics by listing flights, grouped by date and ordered by time.

**Arrival report:**

**Filters:**
- `arrival_date IS NOT NULL`

**Grouping/Ordering:**
- Group by `arrival_date`
- Within each date, order by `arrival_time` ascending

**Columns:**
- Participant name
- Cat1 name
- Cat2 name
- Job title
- `arrival_airline_id` (resolved to airline name)
- `arrival_flight_number`
- `arrival_date`
- `arrival_time`

**Departure report:**

Same structure but with `departure_*` fields:
- Group by `departure_date`
- Order by `departure_time`

**Optional filter:**
- Only participants with a certain conference role, e.g. `Conference_Roles.role_name = 'وزير'`.

This clarifies why each flight-related field exists (airline, date, time, flight number).

***

### Report 13: Participants by Hosting Type

**Goal:** Find participants based on who is hosting them.

**Filters:**
- `hosting = 'من المنظمة'`
- Or any of the enum values:
  - 'غير مستضاف'
  - 'من الدولة'
  - 'من المنظمة'
  - 'من الاتحاد'

**Ordering:**
- Standard Cat1/Cat2/Cat3/participant order.

***

### Report 14: Participants by Hotel

**Goal:** See all participants per guest hotel for rooming lists and logistics.

**Filters:**
- `hotel_id IS NOT NULL`

**Grouping/Ordering:**
- Group by `Guest_Hotels.hotel_name`
- Within each hotel, order by:
  - `Cat1.cat1_order`
  - then `Cat2.cat2_order`
  - then `Participants.order_number`

**Columns:**
- Participant name
- Cat1 name
- Cat2 name
- Cat3 name
- Job title
- Room type
- Hosting

This uses the `hotel_id`, `hotel_room_type`, `hosting`, and cascade categories.

***

### Report 15: Speech Requests

**Goal:** List all participants who requested a speech slot and manage schedule by day and order.

**Filters:**
- `speech_request_date IS NOT NULL` (any participant with a filled request date)

**Grouping/Ordering:**
- Group by `speech_day`
- Within each `speech_day`, order by `speech_order` ascending

**Columns:**
- Participant name
- Cat1/Cat2/Cat3 names
- Conference role
- `speech_request_date`
- `speech_day`
- `speech_order`

This explains the combination of `speech_request_date`, `speech_day`, and `speech_order`.

***

### Report 16: Security Clearance and Passport Status

**Goal:** Track participants requiring security clearance and monitor passport data and clearance process.

**Main filters (clearance required):**
- Participants where:
  - Either `Cat1.clearance_required = true`
  - Or `Participants.clearance_needed = true` (manual override for special cases)

Additional breakdown:
- Sub-report to see who has missing passport files:
  - Clearance-required participants where `passport_copy_url IS NULL` or empty.
- Sub-report for Foreign Affairs flow:
  - Participants with `sent_to_foreign_affairs = true`
  - And `clearance_done = true` (approved).

**Columns:**
- Participant name
- Nationality (from `Cat1` / `nationality_id`)
- Cat3 name (institution)
- Conference role
- Passport details:
  - `birthday`
  - `mother_name`
  - `name_in_passport`
  - `passport_issue_date`
  - `passport_expire_date`
- `passport_copy_url` presence
- Clearance flags (`sent_to_foreign_affairs`, `clearance_done`, `clearance_needed`)

This set of reports explains the logic of `clearance_required` and `no_clearance_age` in `Cat1`, and `clearance_needed`, `sent_to_foreign_affairs`, `clearance_done`, and `passport_copy_url` in `Participants`.

***

These explicit role lists and reporting requirements now fully justify the presence and structure of all the fields in the schema, and they define the expected behavior of queries and UI filters for the internal PocketBase-based web application.