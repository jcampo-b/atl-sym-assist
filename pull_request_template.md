## Description
<!-- One or two sentences: what this PR does and why. -->

## Ticket
- [TICKET_ID](TICKET_URL) - TICKET_TITLE

## Type of Change
- [ ] Feature (`feat:`)
- [ ] Bug fix (`fix:`)
- [ ] Refactor (`refactor:`)
- [ ] Documentation (`docs:`)
- [ ] Test (`test:`)
- [ ] Chore (`chore:`)

## Changes Made
<!-- 4-5 bullets max. No implementation details — those are in the diff. -->
- 
- 
- 

## Testing
- [ ] Unit tests pass
- [ ] Feature tests pass
- [ ] Manual testing completed
- [ ] Tested in local environment
- [ ] API documentation updated (if applicable)

## Checklist
- [ ] Code follows project conventions and naming standards
- [ ] Tests added/updated for new functionality
- [ ] Documentation updated (if needed)
- [ ] No breaking changes (or documented if needed)
- [ ] Code is self-documenting and readable
- [ ] Dependencies updated (if applicable)
- [ ] Migration files created/updated (if applicable)

## Testing Endpoints
<!-- Add one block per affected endpoint. Omit only for PRs with zero API surface. -->

### 1. <brief description>
<!-- Requires: <data or state needed, if any> -->
```
<METHOD> http://127.0.0.1:8000/api/<resource>/<id>
Content-Type: application/json
Authorization: Bearer {{token}}

{ "field": "value" }
```
```json
{
  "data": { "id": 123, "field": "value" }
}
```

### 2. <error case>
```
<METHOD> http://127.0.0.1:8000/api/<resource>/<id>
Content-Type: application/json
Authorization: Bearer {{token}}

{ "field": "invalid_value" }
```
```json
{ "message": "The field is invalid." }
```
Expected: 422

### 3. <side-effect / auto-transition case>
<!-- Requires: <specific data state needed> -->
```
<METHOD> http://127.0.0.1:8000/api/<resource>/<id>/action
Content-Type: application/json
Authorization: Bearer {{token}}

{ "field": "value" }
```
Expected: 200. Verify with `GET /api/<resource>/<id>` after — `<field>` should be `<expected_value>`.

## Additional Notes
<!-- Only for decisions, known gaps, or follow-ups not captured elsewhere. Delete this section if empty. -->