# Zoom

The Zoom source connector syncs users, meetings, webinars, meeting and webinar reports, daily usage reports, Team Chat channels, and Zoom Events from your [Zoom](https://zoom.us) account. It supports two authentication methods: OAuth2.0 and Server-to-Server OAuth.

:::warning Upgrading to 2.0.0
Version 2.0.0 changes primary keys and column types for most streams, and the `report_meetings` and `report_meeting_participants` streams now need the `report:read:user:admin` (granular) or `report:read:admin` (classic) scope. Add the scope before you upgrade. See the [migration guide](https://docs.airbyte.com/integrations/sources/zoom-migrations#upgrading-to-200).
:::

## Prerequisites

- A Zoom account. Some streams depend on your Zoom plan:
  - Report streams, such as `report_meetings`, `report_meeting_participants`, `report_webinars`, `report_webinar_participants`, `report_user_meetings`, and `report_daily`, need a Pro or higher plan.
  - Webinar streams need the [Zoom Webinars](https://zoom.us/webinar) add-on.
  - The `zoom_events` stream needs a Zoom Webinars Plus or Zoom Events license.
- A Zoom account owner or admin, or a user with a role that can create account-level apps in the Zoom App Marketplace and add the admin scopes in [Required scopes](#required-scopes).

## Setup guide

The connector reads account-wide data, so both authentication methods need an account-level Zoom app with the [required scopes](#required-scopes).

### Required scopes

Add these granular scopes to your Zoom app for the streams you want to sync. If your app uses Zoom's classic scopes instead, add `user:read:admin`, `meeting:read:admin`, `webinar:read:admin`, and `report:read:admin`, plus `chat_channel:read:admin` for `chat_channels` and `zoom_events_basic:read:admin` for `zoom_events`.

| Streams | Granular scopes |
| :------ | :-------------- |
| Connection test and all per-user streams (lists the account's users) | `user:read:list_users:admin` |
| `meetings`, `meeting_registrants`, `meeting_polls`, `meeting_poll_results`, `meeting_registration_questions` (lists each user's meetings) | `meeting:read:list_meetings:admin` |
| `meetings` | `meeting:read:meeting:admin` |
| `meeting_registrants` | `meeting:read:list_registrants:admin` |
| `meeting_polls` | `meeting:read:list_polls:admin` |
| `meeting_poll_results` | `meeting:read:list_poll_results:admin` |
| `meeting_registration_questions` | `meeting:read:list_registration_questions:admin` |
| All `webinar*` and `report_webinar*` streams (lists each user's webinars) | `webinar:read:list_webinars:admin` |
| `webinars` | `webinar:read:webinar:admin` |
| `webinar_panelists` | `webinar:read:list_panelists:admin` |
| `webinar_registrants` | `webinar:read:list_registrants:admin` |
| `webinar_absentees` | `webinar:read:list_absentees:admin` |
| `webinar_polls` | `webinar:read:list_polls:admin` |
| `webinar_poll_results` | `webinar:read:list_past_polls:admin` |
| `webinar_registration_questions` | `webinar:read:list_registration_questions:admin` |
| `webinar_tracking_sources` | `webinar:read:list_tracking_sources:admin` |
| `webinar_qna_results` | `webinar:read:past_qa:admin` |
| `report_user_meetings`, `report_meetings`, `report_meeting_participants` (lists each user's past meeting sessions) | `report:read:user:admin` |
| `report_meetings` | `report:read:meeting:admin` |
| `report_meeting_participants` | `report:read:list_meeting_participants:admin` |
| `report_webinars` | `report:read:webinar:admin` |
| `report_webinar_participants` | `report:read:list_webinar_participants:admin` |
| `report_daily` | `report:read:daily_usage:admin` |
| `chat_channels` | `team_chat:read:list_user_channels:admin` |
| `zoom_events` | `zoom_events:read:list_events:admin` |

The connection test only reads the `users` stream, so it passes even if a scope for another stream is missing. That stream's sync fails instead.

If you add scopes to an existing app, re-authenticate the source if you use OAuth2.0 so the new token includes them. Server-to-Server OAuth requests a new token for every sync, so you don't need to reconnect.

### Option 1: OAuth2.0

OAuth2.0 is the default authentication method.

1. In Airbyte, create a new Zoom source and select **OAuth2.0**.
2. If Airbyte asks for a **Client ID** and **Client Secret**, [create a General app](https://developers.zoom.us/docs/integrations/create/) in the Zoom App Marketplace first:
   1. Select **Admin-managed**. A user-managed app can't list your account's users, so the connection test fails.
   2. Add the redirect URL that Airbyte shows to the app's OAuth redirect URL and OAuth allow list.
   3. Add the [required scopes](#required-scopes), then enter the app's client ID and client secret in Airbyte.
3. Click **Authenticate your Zoom account** and sign in as a Zoom account owner or admin.
4. Click **Set up source**.

Zoom refresh tokens expire after 90 days, so re-authenticate the source if it doesn't sync for that long. Authorizing the same Zoom app again, for example for a second source, revokes the tokens issued earlier.

### Option 2: Server-to-Server OAuth

1. Follow Zoom's instructions to [create a Server-to-Server OAuth app](https://developers.zoom.us/docs/internal-apps/s2s-oauth/).
2. On the app's **App credentials** page, copy the **Account ID**, **Client ID**, and **Client secret**.
3. On the **Scopes** page, add the [required scopes](#required-scopes).
4. Activate the app. Zoom doesn't issue access tokens for apps that aren't activated.
5. In Airbyte, create a new Zoom source, select **Server-to-Server OAuth**, and enter the **Account ID**, **Client ID**, and **Client Secret**.
6. Click **Set up source**.

### Optional: Start Date

**Start Date** (`start_date`) is a UTC date-time in the format `YYYY-MM-DDTHH:MM:SSZ`, for example `2026-01-01T00:00:00Z`. It affects these streams:

- `meetings` and `webinars` include only records created on or after the start date, in both Full Refresh and Incremental syncs.
- `report_user_meetings`, `report_meetings`, and `report_meeting_participants` start from the start date. `report_daily` starts at the beginning of the month that contains the start date.

Zoom keeps report data for about six months, so the report streams never request data older than that, even if you set an earlier start date. `report_webinars` and `report_webinar_participants` ignore the start date. Leave the field empty to sync everything Zoom still serves.

### Configure the source with the Airbyte API or Terraform

Before version 1.4.0, the Server-to-Server OAuth settings were top-level fields. Airbyte migrates existing configurations automatically. If you manage the source through the Airbyte API or Terraform, put the settings inside the `credentials` object:

- For OAuth2.0, set `auth_type` to `oauth2.0` and provide `client_id`, `client_secret`, and `refresh_token`.
- For Server-to-Server OAuth, set `auth_type` to `server_to_server` and provide `account_id`, `client_id`, and `client_secret`.

A configuration with only the old top-level `account_id`, `client_id`, `client_secret`, and `authorization_endpoint` fields is rejected. Once `credentials.auth_type` is set, the connector removes those top-level fields.

## Supported sync modes

| Feature | Supported? |
| :------ | :--------- |
| Full Refresh Sync | Yes |
| Incremental Sync | Yes, for `meetings`, `webinars`, `report_meetings`, `report_meeting_participants`, and `report_user_meetings` |
| Namespaces | No |

## Supported streams

Most streams start from the account's users. The connector lists every user, then each user's meetings, webinars, past meeting sessions, and chat channels, then reads the details for each one. `report_daily` and `zoom_events` are read at the account level. For endpoint details, see the [Zoom API reference](https://developers.zoom.us/docs/api/).

| Stream | Zoom endpoint | Primary key | Incremental cursor |
| :----- | :------------ | :---------- | :----------------- |
| `users` | `GET /users` | `id` | |
| `meetings` | `GET /meetings/{meetingId}` | `id` | `created_at` |
| `meeting_registrants` | `GET /meetings/{meetingId}/registrants` | `id` | |
| `meeting_polls` | `GET /meetings/{meetingId}/polls` | `meeting_id`, `id` | |
| `meeting_poll_results` | `GET /past_meetings/{meetingId}/polls` | None | |
| `meeting_registration_questions` | `GET /meetings/{meetingId}/registrants/questions` | `meeting_id` | |
| `webinars` | `GET /webinars/{webinarId}` | `id` | `created_at` |
| `webinar_panelists` | `GET /webinars/{webinarId}/panelists` | `webinar_id`, `id` | |
| `webinar_registrants` | `GET /webinars/{webinarId}/registrants` | `webinar_id`, `id` | |
| `webinar_absentees` | `GET /past_webinars/{webinarUUID}/absentees` | `id` | |
| `webinar_polls` | `GET /webinars/{webinarId}/polls` | `webinar_id`, `id` | |
| `webinar_poll_results` | `GET /past_webinars/{webinarUUID}/polls` | None | |
| `webinar_registration_questions` | `GET /webinars/{webinarId}/registrants/questions` | `webinar_id` | |
| `webinar_tracking_sources` | `GET /webinars/{webinarId}/tracking_sources` | `id` | |
| `webinar_qna_results` | `GET /past_webinars/{webinarUUID}/qa` | None | |
| `report_meetings` | `GET /report/meetings/{meetingUUID}` | `uuid` | `end_time` |
| `report_meeting_participants` | `GET /report/meetings/{meetingUUID}/participants` | `meeting_uuid`, `user_id` | `meeting_end_time` |
| `report_webinars` | `GET /report/webinars/{webinarUUID}` | `uuid` | |
| `report_webinar_participants` | `GET /report/webinars/{webinarUUID}/participants` | `webinar_uuid`, `user_id` | |
| `report_user_meetings` | `GET /report/users/{userId}/meetings` | `uuid` | `end_time` |
| `report_daily` | `GET /report/daily` | `date` | |
| `chat_channels` | `GET /chat/users/{userId}/channels` | `id`, `user_id` | |
| `zoom_events` | `GET /zoom_events/events` | `event_id` | |

`meeting_poll_results`, `webinar_poll_results`, and `webinar_qna_results` have no primary key, so they don't support deduplicated sync modes.

### Stream behavior

- **`meetings` and `webinars`**: Incremental syncs use `created_at` with a 30-day lookback. Edits to meetings or webinars created before that window, including reschedules, settings changes, and new occurrences, aren't re-read. Run a Full Refresh to pick them up.
- **`report_user_meetings`**: Reads Zoom's report of past meetings (`type=past`) for each user, in 28-day windows. Incremental syncs use the meeting `end_time` as the cursor and re-read the last day before the saved cursor, so use **Incremental | Append + Deduped** to avoid duplicate rows.
- **`report_meetings` and `report_meeting_participants`**: Read one report per past meeting session, including each session of a recurring meeting. The connector finds sessions through the same past-meetings report as `report_user_meetings`. Zoom's past-meetings report omits sessions with fewer than two participants, so those sessions produce no rows. Incremental syncs only request reports for sessions that ended after the saved cursor, with a one-day lookback. In Full Refresh, these streams request a report for every session in the last six months, so use Incremental.
- **`report_webinars` and `report_webinar_participants`**: Full Refresh only. Each sync requests a report for every past webinar.
- **`report_daily`**: Requests one daily usage report per month for the current month and the five months before it. Each record is one day, keyed by `date`. This stream is Full Refresh only and re-reads the full window on every sync.
- **`chat_channels`**: Lists the Team Chat channels each user belongs to. A channel with several members appears once per member, which is why the primary key is `id` plus `user_id`.
- **`zoom_events`**: Lists Zoom Events that you host in the `upcoming`, `past`, `draft`, and `cancelled` statuses.

## Rate limits

Zoom applies [rate limits](https://developers.zoom.us/docs/api/rate-limits/) per account, shared by all apps installed on the account, and the limits depend on your Zoom plan. The report endpoints are in Zoom's Heavy category, which has a daily cap. Because the connector makes several requests for every meeting, webinar, and meeting session, large accounts can reach these limits. If syncs fail with rate limit errors, deselect streams you don't need and use Incremental sync for `report_meetings` and `report_meeting_participants`.

When Zoom returns HTTP `429`, the connector waits for the time in Zoom's `Retry-After` header and retries. If the header asks for a wait of 10 minutes or more, the stream fails with the error `The rate limit is greater than max waiting time has been reached.` instead of waiting. If the header is missing, the connector retries with exponential backoff.

## Limitations and troubleshooting

- **Missing scopes.** If the access token lacks a scope, Zoom returns error code `4711` and the sync fails with a configuration error. Add the scope from [Required scopes](#required-scopes) to your Zoom app, then re-authenticate the source if you use OAuth2.0. Streams that ignore HTTP `400` responses, described below, sync zero records instead.
- **Authentication errors.** If Zoom rejects the connector's token request, the connection test or sync fails with a configuration error that includes Zoom's error code:
  - `invalid_grant` (OAuth2.0 only): The refresh token expired or was revoked. Re-authenticate the source.
  - `invalid_client`: The client ID or client secret is wrong, or the app is deactivated.
  - `invalid_request`: For Server-to-Server OAuth, the account ID is wrong.
  - `unsupported_grant_type`: The credentials belong to a different type of Zoom app than the authentication method you selected.
- **Reports aren't available on the plan.** If Zoom returns error code `200` for `report_user_meetings`, `report_meetings`, `report_meeting_participants`, or `report_daily`, the sync fails with a configuration error. These reports need a Pro or higher plan and the report scopes in [Required scopes](#required-scopes).
- **Report data older than six months is skipped.** If saved state or a start date points to report data that Zoom no longer serves, Zoom returns error code `300`. The connector skips that date window or month and continues.
- **Users who can't host meetings are skipped.** If Zoom returns error code `3161` for a user, the connector skips that user's meetings instead of failing.
- **Deleted meetings and webinars are skipped.** If Zoom returns error code `3001` because a meeting or webinar no longer exists or has no report data, the connector skips it instead of failing.
- **Users removed during a sync are skipped.** If a user is removed from the account after the connector lists users, Zoom returns error code `1001` for that user, and `report_user_meetings` and `chat_channels` skip that user.
- **Some streams sync zero records instead of failing.** Most webinar, poll, registration question, and report streams ignore HTTP `400` and `404` responses. For example, if a user doesn't have the Webinars add-on, Zoom returns a `400` error when the connector lists that user's webinars, and the connector syncs no webinar records for that user. If a stream you expect to have data is empty, check your Zoom plan, the host's license, and your app's scopes.

## IP allow list

If you use Airbyte Cloud and your organization restricts access to specific IPs, add the [Airbyte Cloud IP addresses](https://docs.airbyte.com/platform/operating-airbyte/ip-allowlist) to your allow list.

## Changelog

<details>
  <summary>Expand to review</summary>

| Version | Date | Pull Request | Subject |
| :------ | :--------- | :------------------------------------------------------- | :--------------------------------------------------- |
| 2.0.0 | 2026-11-03 | [89722](https://github.com/airbytehq/airbyte/pull/89722) | Add session-based meeting reports, composite keys and date-time typing; retain Full Refresh for webinar reports; honor Retry-After on 429 |
| 1.5.1 | 2026-11-03 | [90076](https://github.com/airbytehq/airbyte/pull/90076) | Update dependencies |
| 1.5.0 | 2026-10-26 | [89564](https://github.com/airbytehq/airbyte/pull/89564) | Add report_user_meetings, report_daily, chat_channels and zoom_events streams; the new streams need additional Zoom app scopes |
| 1.4.1 | 2026-10-23 | [89087](https://github.com/airbytehq/airbyte/pull/89087) | Add registry metadata, spec titles and clearer Server-to-Server field descriptions |
| 1.4.0 | 2026-10-21 | [88359](https://github.com/airbytehq/airbyte/pull/88359) | Add OAuth2.0 authentication as the default, with automatic migration of existing Server-to-Server configurations |
| 1.3.2 | 2026-10-20 | [89459](https://github.com/airbytehq/airbyte/pull/89459) | Update dependencies |
| 1.3.1 | 2026-10-13 | [88881](https://github.com/airbytehq/airbyte/pull/88881) | Update dependencies |
| 1.3.0 | 2026-10-08 | [87561](https://github.com/airbytehq/airbyte/pull/87561) | Use the declarative OAuth authenticator, refresh tokens that expire mid-sync and report invalid credentials as configuration errors |
| 1.2.67 | 2026-10-08 | [87005](https://github.com/airbytehq/airbyte/pull/87005) | Fix sync failures on users without meeting hosting rights, deleted meetings and webinars, and webinar UUIDs containing slashes |
| 1.2.66 | 2026-10-06 | [88100](https://github.com/airbytehq/airbyte/pull/88100) | Update dependencies |
| 1.2.65 | 2026-09-29 | [87422](https://github.com/airbytehq/airbyte/pull/87422) | Update dependencies |
| 1.2.64 | 2026-09-22 | [86858](https://github.com/airbytehq/airbyte/pull/86858) | Update dependencies |
| 1.2.63 | 2026-09-15 | [86307](https://github.com/airbytehq/airbyte/pull/86307) | Update dependencies |
| 1.2.62 | 2026-09-08 | [85742](https://github.com/airbytehq/airbyte/pull/85742) | Update dependencies |
| 1.2.61 | 2026-08-18 | [84815](https://github.com/airbytehq/airbyte/pull/84815) | Update dependencies |
| 1.2.60 | 2026-08-11 | [84173](https://github.com/airbytehq/airbyte/pull/84173) | Update dependencies |
| 1.2.59 | 2026-07-28 | [83194](https://github.com/airbytehq/airbyte/pull/83194) | Update to CDK 7.23.8 and remove the Cloud version override |
| 1.2.58 | 2026-07-28 | [1082](https://github.com/airbytehq/airbyte-python-cdk/issues/1082) | Roll Cloud back to 1.2.56 |
| 1.2.57 | 2026-07-28 | [83156](https://github.com/airbytehq/airbyte/pull/83156) | Update dependencies |
| 1.2.56 | 2026-07-21 | [82672](https://github.com/airbytehq/airbyte/pull/82672) | Update dependencies |
| 1.2.55 | 2026-07-14 | [82073](https://github.com/airbytehq/airbyte/pull/82073) | Update dependencies |
| 1.2.54 | 2026-06-30 | [81304](https://github.com/airbytehq/airbyte/pull/81304) | Update dependencies |
| 1.2.53 | 2026-06-23 | [80722](https://github.com/airbytehq/airbyte/pull/80722) | Update dependencies |
| 1.2.52 | 2026-06-16 | [80114](https://github.com/airbytehq/airbyte/pull/80114) | Update dependencies |
| 1.2.51 | 2026-06-09 | [79582](https://github.com/airbytehq/airbyte/pull/79582) | Update dependencies |
| 1.2.50 | 2026-06-02 | [79076](https://github.com/airbytehq/airbyte/pull/79076) | Update dependencies |
| 1.2.49 | 2026-04-28 | [77500](https://github.com/airbytehq/airbyte/pull/77500) | Update dependencies |
| 1.2.48 | 2026-04-21 | [76821](https://github.com/airbytehq/airbyte/pull/76821) | Update dependencies |
| 1.2.47 | 2026-03-31 | [75837](https://github.com/airbytehq/airbyte/pull/75837) | Update dependencies |
| 1.2.46 | 2026-03-24 | [75052](https://github.com/airbytehq/airbyte/pull/75052) | Update dependencies |
| 1.2.45 | 2026-03-10 | [74448](https://github.com/airbytehq/airbyte/pull/74448) | Update dependencies |
| 1.2.44 | 2026-02-24 | [73991](https://github.com/airbytehq/airbyte/pull/73991) | Update dependencies |
| 1.2.43 | 2026-02-17 | [73592](https://github.com/airbytehq/airbyte/pull/73592) | Update dependencies |
| 1.2.42 | 2026-02-10 | [73211](https://github.com/airbytehq/airbyte/pull/73211) | Update dependencies |
| 1.2.41 | 2026-02-06 | [72082](https://github.com/airbytehq/airbyte/pull/72082) | Update dependencies |
| 1.2.40 | 2026-01-14 | [71380](https://github.com/airbytehq/airbyte/pull/71380) | Update dependencies |
| 1.2.39 | 2025-12-18 | [70796](https://github.com/airbytehq/airbyte/pull/70796) | Update dependencies |
| 1.2.38 | 2025-11-25 | [70145](https://github.com/airbytehq/airbyte/pull/70145) | Update dependencies |
| 1.2.37 | 2025-11-18 | [69428](https://github.com/airbytehq/airbyte/pull/69428) | Update dependencies |
| 1.2.36 | 2025-10-29 | [68840](https://github.com/airbytehq/airbyte/pull/68840) | Update dependencies |
| 1.2.35 | 2025-10-21 | [68502](https://github.com/airbytehq/airbyte/pull/68502) | Update dependencies |
| 1.2.34 | 2025-10-14 | [67984](https://github.com/airbytehq/airbyte/pull/67984) | Update dependencies |
| 1.2.33 | 2025-10-07 | [67477](https://github.com/airbytehq/airbyte/pull/67477) | Update dependencies |
| 1.2.32 | 2025-09-30 | [66942](https://github.com/airbytehq/airbyte/pull/66942) | Update dependencies |
| 1.2.31 | 2025-09-24 | [66312](https://github.com/airbytehq/airbyte/pull/66312) | Update dependencies |
| 1.2.30 | 2025-09-09 | [65697](https://github.com/airbytehq/airbyte/pull/65697) | Update dependencies |
| 1.2.29 | 2025-08-24 | [65399](https://github.com/airbytehq/airbyte/pull/65399) | Update dependencies |
| 1.2.28 | 2025-08-10 | [64833](https://github.com/airbytehq/airbyte/pull/64833) | Update dependencies |
| 1.2.27 | 2025-08-02 | [64316](https://github.com/airbytehq/airbyte/pull/64316) | Update dependencies |
| 1.2.26 | 2025-07-27 | [64092](https://github.com/airbytehq/airbyte/pull/64092) | Update dependencies |
| 1.2.25 | 2025-07-20 | [63671](https://github.com/airbytehq/airbyte/pull/63671) | Update dependencies |
| 1.2.24 | 2025-07-12 | [63182](https://github.com/airbytehq/airbyte/pull/63182) | Update dependencies |
| 1.2.23 | 2025-07-05 | [62729](https://github.com/airbytehq/airbyte/pull/62729) | Update dependencies |
| 1.2.22 | 2025-06-28 | [62221](https://github.com/airbytehq/airbyte/pull/62221) | Update dependencies |
| 1.2.21 | 2025-06-21 | [61777](https://github.com/airbytehq/airbyte/pull/61777) | Update dependencies |
| 1.2.20 | 2025-06-16 | [61205](https://github.com/airbytehq/airbyte/pull/61205) | Update dependencies |
| 1.2.19 | 2025-05-24 | [60763](https://github.com/airbytehq/airbyte/pull/60763) | Update dependencies |
| 1.2.18 | 2025-05-10 | [59936](https://github.com/airbytehq/airbyte/pull/59936) | Update dependencies |
| 1.2.17 | 2025-05-04 | [59545](https://github.com/airbytehq/airbyte/pull/59545) | Update dependencies |
| 1.2.16 | 2025-04-27 | [58534](https://github.com/airbytehq/airbyte/pull/58534) | Update dependencies |
| 1.2.15 | 2025-04-13 | [58032](https://github.com/airbytehq/airbyte/pull/58032) | Update dependencies |
| 1.2.14 | 2025-04-06 | [57394](https://github.com/airbytehq/airbyte/pull/57394) | Update dependencies |
| 1.2.13 | 2025-03-30 | [56840](https://github.com/airbytehq/airbyte/pull/56840) | Update dependencies |
| 1.2.12 | 2025-03-23 | [56339](https://github.com/airbytehq/airbyte/pull/56339) | Update dependencies |
| 1.2.11 | 2025-03-09 | [55666](https://github.com/airbytehq/airbyte/pull/55666) | Update dependencies |
| 1.2.10 | 2025-03-02 | [54642](https://github.com/airbytehq/airbyte/pull/54642) | Update dependencies |
| 1.2.9 | 2025-02-18 | [54113](https://github.com/airbytehq/airbyte/pull/54113) | Update dependencies |
| 1.2.8 | 2025-02-09 | [53595](https://github.com/airbytehq/airbyte/pull/53595) | Update dependencies |
| 1.2.7 | 2025-02-02 | [53122](https://github.com/airbytehq/airbyte/pull/53122) | Update dependencies |
| 1.2.6 | 2025-01-26 | [52548](https://github.com/airbytehq/airbyte/pull/52548) | Update dependencies |
| 1.2.5 | 2025-01-19 | [51931](https://github.com/airbytehq/airbyte/pull/51931) | Update dependencies |
| 1.2.4 | 2025-01-12 | [51467](https://github.com/airbytehq/airbyte/pull/51467) | Update dependencies |
| 1.2.3 | 2024-12-29 | [50835](https://github.com/airbytehq/airbyte/pull/50835) | Update dependencies |
| 1.2.2 | 2024-12-22 | [50394](https://github.com/airbytehq/airbyte/pull/50394) | Update dependencies |
| 1.2.1 | 2024-12-15 | [49445](https://github.com/airbytehq/airbyte/pull/49445) | Update dependencies |
| 1.2.0 | 2024-10-29 | [47299](https://github.com/airbytehq/airbyte/pull/47299) | Migrate to manifest only format |
| 1.1.22 | 2024-10-29 | [47755](https://github.com/airbytehq/airbyte/pull/47755) | Update dependencies |
| 1.1.21 | 2024-10-28 | [47094](https://github.com/airbytehq/airbyte/pull/47094) | Update dependencies |
| 1.1.20 | 2024-10-13 | [46824](https://github.com/airbytehq/airbyte/pull/46824) | Update dependencies |
| 1.1.19 | 2024-10-06 | [46412](https://github.com/airbytehq/airbyte/pull/46412) | Update dependencies |
| 1.1.18 | 2024-09-29 | [46196](https://github.com/airbytehq/airbyte/pull/46196) | Update dependencies |
| 1.1.17 | 2024-09-22 | [45737](https://github.com/airbytehq/airbyte/pull/45737) | Update dependencies |
| 1.1.16 | 2024-09-15 | [45523](https://github.com/airbytehq/airbyte/pull/45523) | Update dependencies |
| 1.1.15 | 2024-09-08 | [45220](https://github.com/airbytehq/airbyte/pull/45220) | Update dependencies |
| 1.1.14 | 2024-09-01 | [45037](https://github.com/airbytehq/airbyte/pull/45037) | Update dependencies |
| 1.1.13 | 2024-08-25 | [44676](https://github.com/airbytehq/airbyte/pull/44676) | Update dependencies |
| 1.1.12 | 2024-08-18 | [44249](https://github.com/airbytehq/airbyte/pull/44249) | Update dependencies |
| 1.1.11 | 2024-08-11 | [43578](https://github.com/airbytehq/airbyte/pull/43578) | Update dependencies |
| 1.1.10 | 2024-08-04 | [43149](https://github.com/airbytehq/airbyte/pull/43149) | Update dependencies |
| 1.1.9 | 2024-07-28 | [42652](https://github.com/airbytehq/airbyte/pull/42652) | Update dependencies |
| 1.1.8 | 2024-07-21 | [42212](https://github.com/airbytehq/airbyte/pull/42212) | Update dependencies |
| 1.1.7 | 2024-07-14 | [41813](https://github.com/airbytehq/airbyte/pull/41813) | Update dependencies |
| 1.1.6 | 2024-07-11 | [41486](https://github.com/airbytehq/airbyte/pull/41486) | Update dependencies |
| 1.1.5 | 2024-07-10 | [41316](https://github.com/airbytehq/airbyte/pull/41316) | Update dependencies |
| 1.1.4 | 2024-07-07 | [40986](https://github.com/airbytehq/airbyte/pull/40986) | Update dependencies |
| 1.1.3 | 2024-06-27 | [40509](https://github.com/airbytehq/airbyte/pull/40509) | Update dependencies |
| 1.1.2 | 2024-06-23 | [40141](https://github.com/airbytehq/airbyte/pull/40141) | Update dependencies |
| 1.1.1 | 2024-06-11 | [39279](https://github.com/airbytehq/airbyte/pull/39279) | [autopull] Upgrade base image to v1.2.2 |
| 1.1.0 | 2024-03-12 | [35369](https://github.com/airbytehq/airbyte/pull/35369) | Publish S2S Oauth connector with fixed authenticator |
| 1.0.0 | 2023-08-01 | [25308](https://github.com/airbytehq/airbyte/pull/25308) | Replace JWT Auth methods with server-to-server Oauth |
| 0.1.1 | 2022-12-02 | [19939](https://github.com/airbytehq/airbyte/pull/19939) | Upgrade CDK version to fix bugs with SubStreamSlicer |
| 0.1.0   | 2022-10-25 | [18179](https://github.com/airbytehq/airbyte/pull/18179) | Initial Release                                      |

</details>
