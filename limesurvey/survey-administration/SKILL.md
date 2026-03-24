---
name: survey-administration
description: Administer and edit LimeSurvey surveys at limesurvey.svc.external.tba-hosting.de. Covers LSS file format (structure, groups, questions, branching, answers, l10ns), the RemoteControl 2 JSON-RPC API, survey import/export, permissions, and question types. Use when editing survey content, fixing LSS files, setting up branching logic, managing permissions, or automating changes via the API.
---
# LimeSurvey Survey Administration

Instance URL: `https://limesurvey.svc.external.tba-hosting.de`

## Authentication

- Login is via **LDAP** (same credentials as other DIPF services).
- Survey creators automatically get full permissions on surveys they create/import.
- Permissions for other users must be set manually in the UI: **Survey → Survey Permissions → Add user → check "All"**.

## RemoteControl 2 API

**Endpoint:** `POST /index.php/admin/remotecontrol`  
**Content-Type:** `application/json`

The RemoteControl 2 plugin must be **enabled by a LimeSurvey admin** in the plugin manager. If it returns an empty response body (0 bytes) with HTTP 200, the plugin is disabled.

### LDAP Authentication

```bash
curl -s -X POST https://limesurvey.svc.external.tba-hosting.de/index.php/admin/remotecontrol \
  -H "Content-Type: application/json" \
  -d '{"method":"get_session_key","params":["USERNAME","PASSWORD","AuthLDAP"],"id":1}'
```

The 3rd parameter `"AuthLDAP"` selects the LDAP plugin. Without it, internal DB auth is used.

### Common API Methods

| Method | Purpose |
|--------|---------|
| `get_session_key(user, pass, plugin)` | Authenticate, returns session key |
| `release_session_key(key)` | Logout |
| `add_group(key, sid, groupdata)` | Add a question group |
| `set_group_properties(key, gid, props)` | Update group (grelevance, name, etc.) |
| `set_question_properties(key, qid, props)` | Update question (gid, relevance, etc.) |
| `delete_question(key, qid)` | Delete a question (only safe when survey inactive) |
| `import_question(key, sid, gid, b64xml, format)` | Import LSQ XML as base64 |
| `list_surveys(key, username)` | List surveys |

⚠️ There is **no `add_answer` API method**. To change answer options, you must `delete_question` then `import_question` with full LSQ XML base64-encoded.

⚠️ `import_question` requires the survey to be **inactive** (`active=N`).

## LSS File Format

LimeSurvey export files (`.lss`) are XML with these top-level sections:

```
<surveys>             – Survey settings (sid, format, active, etc.)
<surveys_languagesettings>  – Title, welcome text, policy notice, emails
<groups>              – Group definitions (gid, grelevance, group_order)
<group_l10ns>         – Group localized names and descriptions
<questions>           – Question definitions (qid, gid, type, title, relevance)
<question_l10ns>      – Question localized text and help
<question_attributes> – Per-question attributes (text_input_width, rows, etc.)
<subquestions>        – Subquestions (for matrix/array types)
<answers>             – Answer options (qid, code, sortorder)
<answer_l10ns>        – Localized answer text
<conditions>          – Legacy conditions (use <relevance> instead)
<assessments>         – Assessment rules
```

### Critical Consistency Rules

Every `<groups>` row must have a matching `<group_l10ns>` row with the same `gid`. Missing l10n rows cause broken group names and silent failures in branching logic.

Every `<questions>` row must have a matching `<question_l10ns>` row.

The `grelevance` field appears in **both** `<groups>` and `<group_l10ns>`. They must match. LimeSurvey reads from `<groups>` at runtime; `<group_l10ns>` is for display metadata.

### Common Bug: Duplicate XML Tags in LSS

When manually editing LSS files, duplicate child tags in a `<row>` (e.g., two `<group_name>` elements) cause XML parsers to silently keep only the last value. This can cause a group's l10n entry to appear with wrong name/grelevance while the original group's l10n entry disappears entirely.

Always validate with:
```python
import xml.etree.ElementTree as ET
ET.parse('umfrage.lss')  # raises ParseError on malformed XML
```

## Group-by-Group Branching

Survey format `G` (Group-by-Group) shows one group per page. Branching is controlled by `grelevance` on the group.

Example for a Ja/Nein branch on Q01:
```
Group 1 (grelevance=1):          Q01 alone
Group 2 (grelevance=Q01.NAOK=='Y'):  Questions for "Ja" answers
Group 3 (grelevance=Q01.NAOK=='N'):  Questions for "Nein" answers
Group 4 (grelevance=1):          Always-visible closing questions
```

- `NAOK` suffix means "Not Applicable OK" — allows unanswered questions to pass relevance.
- Answer codes must match exactly: `'Y'` and `'N'` (uppercase, single quotes in expression).
- Individual question `relevance` conditions should mirror the group grelevance for extra safety.

## Question Types

| Type code | Name | Notes |
|-----------|------|-------|
| `Y` | Yes/No | Built-in toggle, stores Y/N. Hard to see if selected (visual issue). |
| `L` | List (Radio) | Requires explicit answer options. Preferred for Ja/Nein. |
| `T` | Long free text | Multi-line textarea. Use `question_attributes` `text_input_width=100`. |
| `S` | Short free text | Single line input. |
| `M` | Multiple choice | Checkboxes with subquestions. |

For a type `L` Ja/Nein question, add answer options in `<answers>` and `<answer_l10ns>`:
```xml
<answers>
  <rows>
    <row><aid>1</aid><qid>1</qid><code>Y</code><sortorder>0</sortorder><assessment_value>0</assessment_value><scale_id>0</scale_id></row>
    <row><aid>2</aid><qid>1</qid><code>N</code><sortorder>1</sortorder><assessment_value>0</assessment_value><scale_id>0</scale_id></row>
  </rows>
</answers>
<answer_l10ns>
  <rows>
    <row><id>1</id><aid>1</aid><answer><![CDATA[Ja]]></answer><language>de</language></row>
    <row><id>2</id><aid>2</aid><answer><![CDATA[Nein]]></answer><language>de</language></row>
  </rows>
</answer_l10ns>
```

## Survey Import and SID Preservation

When importing an `.lss` file:
- LimeSurvey tries to use the `<sid>` from the file.
- If that SID is already taken, a **new SID is assigned** → public survey URL changes.
- To preserve the SID: delete the old survey first, then import.
- ⚠️ Deleting a survey destroys all response data and token participants.

## Adding the Privacy Notice

The `surveyls_policy_notice` field in `<surveys_languagesettings>` renders a consent/privacy checkbox before the survey starts. Set it in the LSS file:

```xml
<surveyls_policy_notice><![CDATA[<p>Your HTML privacy text here.</p>]]></surveyls_policy_notice>
```

Or in the UI: **Survey Settings → General → Policy notice**.

## Typical Workflow: Update Survey via LSS Import

1. Edit the `.lss` file locally.
2. Validate XML: `python3 -c "import xml.etree.ElementTree as ET; ET.parse('umfrage.lss')"`.
3. Run consistency checks (all gids in `<groups>` have matching `<group_l10ns>`, all qids have `<question_l10ns>`, etc.).
4. In LimeSurvey UI: note existing permissions → delete old survey → import `.lss`.
5. Re-apply permissions for other users (UI: Survey Permissions → Add user → All).
6. Test with **Preview Survey** before activating.

## Consistency Validation Script

```python
import xml.etree.ElementTree as ET, re

def validate_lss(path):
    tree = ET.parse(path)
    root = tree.getroot()
    errors = []

    groups = {r.findtext('gid'): r for r in root.findall('.//groups/rows/row')}
    group_l10ns = {r.findtext('gid'): r for r in root.findall('.//group_l10ns/rows/row')}
    questions = {r.findtext('qid'): r for r in root.findall('.//questions/rows/row')}
    q_l10ns = {r.findtext('qid'): r for r in root.findall('.//question_l10ns/rows/row')}
    answers = {}
    for r in root.findall('.//answers/rows/row'):
        answers.setdefault(r.findtext('qid'), []).append(r.findtext('code'))

    for gid in groups:
        if gid not in group_l10ns:
            errors.append(f"Group gid={gid} missing in group_l10ns")
    for qid, q in questions.items():
        title = q.findtext('title')
        if qid not in q_l10ns:
            errors.append(f"Question {title} missing in question_l10ns")
        if q.findtext('gid') not in groups:
            errors.append(f"Question {title} references unknown gid={q.findtext('gid')}")
        if q.findtext('type') == 'L' and qid not in answers:
            errors.append(f"Question {title} type=L has no answer options")

    valid_titles = {q.findtext('title') for q in questions.values()}
    for gid, g in groups.items():
        for ref in re.findall(r'(\w+)\.NAOK', g.findtext('grelevance') or ''):
            if ref not in valid_titles:
                errors.append(f"Group gid={gid} grelevance refs unknown question '{ref}'")

    return errors

errors = validate_lss('umfrage.lss')
print('\n'.join(errors) if errors else '✅ All checks passed')
```

## Known Quirks and Limitations

- ⚠️ **RemoteControl 2 plugin is not enabled** on this instance. All changes must be made via LSS import or the web UI. A LimeSurvey admin would need to enable it in Plugin Manager.
- ⚠️ **group_l10ns duplicate tag bug**: When manually building LSS XML and merging two rows accidentally into one `<row>`, the second set of tags silently overrides the first. The group appears with wrong name and wrong grelevance, and another group's l10n row is missing entirely. Always use a script to validate.
- ⚠️ **grelevance mismatch**: `<groups>` and `<group_l10ns>` both contain `grelevance`. If they differ, behavior is undefined. Keep them in sync.
- ⚠️ After selecting Ja/Nein on a type-Y question with broken group structure, the survey ends immediately without showing follow-up questions. Root cause is always missing or wrong grelevance on groups 2/3.
- The `question_order` values must be unique across the entire survey (not just within a group).
- `<conditions>` (legacy condition system) and `<relevance>` (ExpressionScript) can coexist but may conflict. Prefer `relevance` exclusively.
- Survey links use the SID: `https://limesurvey.svc.external.tba-hosting.de/index.php/123456`. If the SID changes on import, all shared links break.

## References

- [LimeSurvey RemoteControl 2 API docs](https://api.limesurvey.org/classes/remotecontrol_handle.html)
- [LimeSurvey Manual – ExpressionScript](https://manual.limesurvey.org/ExpressionScript_-_Presentation)
- [LimeSurvey Manual – Survey import/export](https://manual.limesurvey.org/Surveys_-_introduction#Import_a_survey)
- [LimeSurvey Manual – Question types](https://manual.limesurvey.org/Question_types)
