# MyAquaTrax — Home Assistant Integration

Pull water usage, cost, and comparison data from [myaquatrax.com](https://myaquatrax.com) into Home Assistant.

## About this fork

This is a fork of [HenrikJohnson/myaquatrax-ha](https://github.com/HenrikJohnson/myaquatrax-ha) with one fix: accounts whose meter isn't a smart (AMI) meter could not be set up. The rest of this README is unchanged from the original and describes its behavior, apart from the bi-monthly sensors below.

To install this fork through HACS or the Marketplace, add `dmkonovalov83/myaquatrax-ha` as a custom repository instead of the URL in [Installation](#installation).

**Full diff against the original:** [compare view](https://github.com/HenrikJohnson/myaquatrax-ha/compare/main...dmkonovalov83:myaquatrax-ha:main)

### The problem

A City of San Jose account with a non-AMI meter failed setup with "Invalid authentication", even though its email and password were correct. There were two separate causes:

1. **Wrong login form.** The integration showed the SSO form (Account + Email + Street) whenever a utility had `ssoenable = 1`. The portal's own login page only uses SSO when `ssoenable = 1` **and** `IsRegistration = false`. San Jose has `IsRegistration = true`, so the portal uses email + password, which the integration offered no way to enter.
2. **Empty meter list reported as an auth failure.** After login, the integration only looked up meters with `mode=H`. For a non-AMI meter that returns no rows, and `config_flow.py` treats "no meters" as `InvalidAuth`. The portal's usage page opens in bi-monthly view and requests meters with an empty `mode`, which does list the meter. For this account, only bi-monthly (`mode=B`) usage returned any data.

### What changed

| File | Change |
|------|--------|
| `config_flow.py` | Shows the SSO form only when `sso_enabled and not is_registration`, the same rule as the portal's login page |
| `myaquatrax_client.py` | `login()` looks up the utility and sends the hidden field values the portal's page sets for it (`Mfe`, `InvoiceCoudstatus`, `IsRegistration`, `IsSSOEnabled`) instead of fixed values. Raises `AquaTraxError` for an unknown utility |
| `myaquatrax_client.py` | `get_usage_meters()` falls back to the empty-`mode` lookup when `mode=H` returns nothing |
| `myaquatrax_client.py` | Adds `Utility.mfa_enabled` and `UsageReading.month` (from `MonthUsageDate`) |
| `coordinator.py` | On the first update, treats the meter as bi-monthly if `mode=H` lists no meters, and then fetches only `mode=B` usage in gallons and CCF |
| `sensor.py` | Adds a bi-monthly sensor set, used instead of the standard sensors for those meters, plus a `usage_month` attribute when a reading has a month |
| `sensor.py` | Changes the CCF sensors' unit from `hcf` to `CCF`. Home Assistant logged `hcf` as invalid for water sensors, and listed `CCF` as valid |

### Bi-monthly sensors

These are created for meters with no hourly data, in both gallons and CCF:

| Sensor | Description |
|--------|-------------|
| Latest Bi-monthly Usage | Usage in the most recent bi-monthly reading |
| Bi-monthly Average | `AVERAGE` from the portal |
| Bi-monthly Highest | `HIGHEST` from the portal |
| Bi-monthly Lowest | `LOWEST` from the portal |

They have no `state_class`, so they don't record long-term statistics or feed the Energy Dashboard. New readings only appear about every two months. Cost, comparison and flow sensors aren't created for these meters. Dollar amounts were hidden for the tested account (`IsShow = 0`), and comparison data wasn't tested for non-AMI meters.

Meters that do list under `mode=H` take the same code path and keep the same sensors as before.

### Effects on other accounts

- Every email + password login now makes one extra `GetMasterUtilityList` request and sends that utility's field values.
- Any utility with `ssoenable = 1` and `IsRegistration = true` now gets the email + password form during setup. Existing SSO config entries keep logging in with SSO.
- An AMI account makes one extra `mode=H` lookup on its first update. If that lookup ever returned no rows, the account would be treated as bi-monthly until Home Assistant restarts.
- The unit change applies to every CCF sensor, so an account with long-term statistics recorded under `hcf` may see a unit-change prompt in Home Assistant.

### Testing

- **Live portal:** the patched client was run against myaquatrax.com with one City of San Jose non-AMI account. Login, account and meter discovery, and bi-monthly usage in gallons and CCF all worked.
- **Offline:** the config flow's form choice, the coordinator and the sensors were tested with Home Assistant stubbed out, using responses captured from the portal.
- **Real Home Assistant:** on Home Assistant OS with Core 2026.11.0, installed from this fork through the Marketplace, the same account completed setup with email + password. The integration loaded and all eight bi-monthly sensors showed the portal's values. After updating to the CCF unit change and restarting, the CCF sensors reported `CCF` with the same values, and the unit warnings were gone.
- **Not tested yet:** SSO logins, AMI accounts, and other utilities.

### Left as is

These looked worth a follow-up but weren't needed for the fix:

- `_do_login_post()` treats the word `Invalid` anywhere in the returned page as a failed login, which could misreport a successful login.
- There's no `translations/` folder, so setup form labels may show as raw field names in custom installs.

## Features

- **Gallons** and **CCF/HCF** usage sensors (billing period total, daily, average, highest, lowest)
- **Dollar** cost sensors (billing period total, daily)
- **Comparison sensors** vs. previous year, utility average, and zip code average
- Energy Dashboard compatible (`device_class=WATER` + `TOTAL_INCREASING`)
- Billing Dashboard compatible (`device_class=MONETARY`)
- Supports both **email/password** and **SSO** (Account + Email + Street) logins
- Auto-discovers water meters after login

## Supported Water Utilities

Any water district that uses the MyAquaTrax portal:

> **Note:** While no providers currently enforce MFA, this integration does not support it. If your provider enables MFA in the future, login will fail.

- City of Newport Beach
- City of American Canyon
- City of Morgan Hill
- City of Gilroy
- City of Goodyear
- City of La Marque
- City of White Plains
- City of San Jose
- Rowland Water District
- Fairview City
- Marshall & Polk Rural Water System
- Myoma Dunes Water Company
- City of Cedar Hills
- Crestline Village Water District
- Laguna Beach County Water District
- City of Lindon
- City of Buena Park
- City of Fountain Valley
- East Orange County Water District

## Installation

### HACS (Recommended)

1. In HACS, click **⋮ → Custom repositories**
2. Add `https://github.com/HenrikJohnson/myaquatrax-ha` as an **Integration** type repository
3. Search for "MyAquaTrax" and install
4. Restart Home Assistant

### Manual

1. Copy the `custom_components/myaquatrax` folder into your Home Assistant `config/custom_components` directory
2. Restart Home Assistant

## Configuration

1. Go to **Settings → Devices & Services → Add Integration**
2. Search for **"MyAquaTrax"**
3. Select your **Water Utility** from the dropdown
4. Enter credentials:
   - **SSO utilities** (e.g. Newport Beach): Account Number, Email, Street Number
   - **Standard utilities**: Email, Password
5. Select your **Water Meter** from the discovered list

## Sensors

### Water Usage (per unit)

| Sensor | Description |
|--------|-------------|
| `sensor.g_billing_period_total` | Billing period cumulative water |
| `sensor.g_daily_usage` | Most recent day's water usage |
| `sensor.g_average` | Average usage |
| `sensor.g_highest` | Highest usage |
| `sensor.g_lowest` | Lowest usage |
| `sensor.w_billing_period_total` | Same in CCF/HCF |
| `sensor.w_daily_usage` | Same in CCF/HCF |

### Cost

| Sensor | Description |
|--------|-------------|
| `sensor.d_billing_period_cost` | Billing period cumulative cost |
| `sensor.d_daily_cost` | Most recent day's cost |

### Comparison (Gallons)

| Sensor | Description |
|--------|-------------|
| `sensor.compare_current` | Current month vs. your usage |
| `sensor.compare_previous` | Current month vs. same month last year |
| `sensor.compare_utility` | Current month vs. utility average |
| `sensor.compare_zip` | Current month vs. zip code average |

## Energy Dashboard

To track water usage in the Energy Dashboard:

### Step 1: Add the Water Source

1. Go to **Settings → Devices & Services → Energy**
2. Click **Add water source**
3. Select one of these sensors:

| Sensor | Unit | When to use |
|--------|------|-------------|
| `sensor.g_billing_period_total` | Gallons | Best if your water bill is in gallons |
| `sensor.w_billing_period_total` | CCF | Best if your water bill is in hundred cubic feet |

### Step 2: Add Cost Tracking

1. In the Energy Dashboard, under your water source, click **Add cost**
2. Select `sensor.d_billing_period_cost`

### Optional: Flow Rate Monitoring

Add these sensors to your main dashboard for a near-real-time view. These appear in the **Flow Rate** sensor picker in the Energy Dashboard:

| Sensor | Unit | HA Picker | Description |
|--------|------|-----------|-------------|
| `sensor.g_flow_current_hour_flow` | gal/h | ✅ | Current hour's water flow rate |
| `sensor.g_flow_peak_flow_today` | gal/h | ✅ | Highest hour flow today |
| `sensor.w_flow_current_hour_flow` | ft³/h | ✅ | Current hour's flow in cubic feet (converted from CCF) |
| `sensor.w_flow_peak_flow_today` | ft³/h | ✅ | Peak flow in cubic feet (converted from CCF) |

> **Note:** Flow rate is synthesized from hourly usage data (each hour's TotalValue). The actual meter does not provide a live flow rate. CCF flow values are converted from hundred-cubic-feet to cubic feet (1 CCF = 100 ft³).

## API Endpoints

| Endpoint | Purpose |
|----------|---------|
| `POST /Account/Login` | Standard email/password login |
| `POST /Account/LoginWithSso` | SSO login (Account + Email + Street) |
| `GET /common/GetMasterUtilityList` | List water utilities |
| `GET /MVPDashboard/GetAccountList` | Get customer accounts |
| `GET /MVPUsage/GetMeterDetails` | Discover water meters |
| `GET /MVPUsage/GetWaterUsage` | Usage data (mode=H/D/M/B, type=G/W/D/C) |
| `GET /MVPCompare/GetWaterCompareSpending` | Comparison data |

## Development

```bash
# List all utilities
python myaquatrax_client.py list-utilities

# Test SSO login + meter discovery
python myaquatrax_client.py sso <utility_id> <account> <email> <street>

# Test standard login
python myaquatrax_client.py <utility_id> <email> <password>
```

## Requirements

- Home Assistant 2024.1+ (for `ConfigEntry.runtime_data`)
- `requests` Python library

## License

MIT
