# Opt-in Project Updates

Use this reference when a plugin reads or writes records, models, uploads, or datetimes through the CMA, or queries the CDA. A plugin must work in every project it's installed in, and projects differ in which opt-in updates they have activated.

## Why plugins see different behavior

DatoCMS ships some API changes as opt-in updates ("Available updates" in an environment's configuration). Projects created after an update ships get it automatically. Older projects keep the old behavior until someone activates it, and activation is one-way. The API docs describe the updated behavior only.

So a plugin tested on a fresh project sees every update active, while a customer's older project may have none. Write code that behaves the same in both states.

## Portable patterns

Prefer an explicit parameter or a format-tolerant read over branching on a flag. Branch only where the two states need different payloads.

| Flag in `ctx.site.meta` | What differs without the update | Code that works in both states |
| - | - | - |
| `improved_items_listing` | Listing records (`GET /items`) without `version` returns only published records, not drafts. | Pass `version: 'current'` or `version: 'published'` on every list. Single-record reads (`items.find`) always return `current`. |
| `draft_mode_default` | A model created without `draft_mode_active` gets draft mode off; with the update, on. | Pass `draft_mode_active` when creating a model. |
| `improved_boolean_fields` | Boolean field values can be `null`, both on existing records and after writing `null`. | Read with `value === true`. Write `true` or `false`, never `null`. |
| `milliseconds_in_datetime` | Datetimes serialize without milliseconds (`…T14:30:00+00:00` vs `…T14:30:00.345+00:00`). | Parse with `new Date(value)` and compare instants. Don't compare or regex-match datetime strings across reads that can come from different projects. |
| `improved_timezone_management` | The CMA ignores the offset you send in a DateTime field: it stores the wall-clock time in the project timezone. | Convert the value to the project timezone (`ctx.site.attributes.timezone`) before writing, with that zone's offset. Both behaviors then store the same instant. |
| `improved_validation_at_publishing` | Publishing a record that fails current validations can succeed. With the update, publish fails with `INVALID_DRAFT`. | Handle `INVALID_DRAFT` on every publish call, and show the editor which record failed. |
| `non_localized_focal_points` | Upload `default_field_metadata` is locale-keyed (`{ en: { alt, title, custom_data, focal_point } }`). With the update it's field-keyed (`{ alt: { en }, title: { en }, custom_data: { en }, focal_point }`), and the CMA rejects the locale-keyed shape on write. | Branch on the flag: there's no parameter to pick the shape. Read either shape, and write the shape the flag selects. Needs `@datocms/cma-client` 5.5.0 or later for the field-keyed types. |

These only matter when the plugin queries the CDA (GraphQL): `improved_hex_management` (color `hex` gains an alpha pair), `improved_gql_multilocale_fields` (`locale` and `fallbackLocales` take effect on multi-locale and meta fields), `improved_gql_visibility_control` (limited tokens no longer see out-of-scope schema), and `improved_exposure_of_inline_blocks_in_cda` (inline blocks move from `blocks` to `inlineBlocks`). Read color channels (`red`, `green`, `blue`, `alpha`) instead of `hex`, and query `blocks` and `inlineBlocks` defensively, or branch on the flag.

## Reading the flags

`ctx.site.meta` carries every flag in every hook and render context, with no extra permission. The flags apply per environment, and `ctx.site` reflects the environment the editor has open.

```ts
const isFieldKeyedMetadata = ctx.site.meta.non_localized_focal_points;
```

`client.site.find()` returns the same `meta` for any role with access to the environment. It needs `currentUserAccessToken`, so prefer `ctx.site.meta`.

## Testing both states

New projects get every update, and no API call deactivates one, so a fresh test project can't reproduce the old behavior.

1. Keep each flag-dependent branch in a pure function that takes the flag as an argument.
2. Unit-test that function with the flag `true` and `false`.
3. Test the rest of the plugin on a fresh project, which covers the updated state.
