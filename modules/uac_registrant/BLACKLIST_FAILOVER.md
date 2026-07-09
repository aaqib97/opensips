# Blacklist-Aware Failover for uac_registrant

## Overview

This customization makes the `uac_registrant` module check the destination IP
against the blacklist and fail over to the first non-blacklisted resolved IP.

Previously the module always sent to the **first** address returned when the
registrar (or outbound proxy) URI was resolved, with no blacklist check for
these locally generated requests. A blacklisted or dead registrar IP would keep
being used. With this feature enabled, the module resolves the next hop, walks
the resolved addresses (A / SRV chain), skips blacklisted ones, and pins the
first clean IP as the send destination.

The check is deliberately **not** run on every send. It runs only when it can
actually change the outcome — see *When It Runs* below — so a healthy registrant
pays no extra cost on its periodic refresh.

## Backward Compatibility

This feature is **disabled by default**. The module parameter
`enable_blacklist_failover` must be explicitly set to `1` to activate it. When
disabled (the default), the send path is byte-for-byte the original behavior.

## How It Works

- The module resolves the next hop — the outbound proxy (`proxy`) if configured
  for the registrant, otherwise the registrar.
- It iterates the resolved destinations and checks each against the core
  blacklist facility (`check_against_blacklist()`), the same facility that the
  `tm` module consults during forwarding.
- The first non-blacklisted destination is pinned via the dialog's
  `forced_to_su`, which `tm`'s `t_uac()` honors as the send target.
- If **all** resolved IPs are blacklisted, or the next hop cannot be resolved,
  the module leaves destination selection to the normal first-address logic and
  still attempts the request (so the registrant does not go silent).

The blacklist check runs in a private, temporary processing context so it works
correctly from the module's timer process; nothing is added to the blacklist by
this check (`add_to_bl = 0`).

## When It Runs

The selection is driven from the registrant's periodic timer
(`run_timer_check`), only at the moments where a different IP can help:

| Situation | Registrant state | Check runs? |
|-----------|------------------|-------------|
| Fresh registration (startup, reload, enable, after unregister) | `NOT_REGISTERED_STATE` (genuine) | **Yes** |
| Healthy periodic refresh | `REGISTERED_STATE` → timeout | No |
| Retry after 408 / no reply | `REGISTER_TIMEOUT_STATE` | **Yes** |
| Retry after 503 / other registrar error | `REGISTRAR_ERROR_STATE` | **Yes** |
| Auth challenge (401/407) re-send, 423 re-send | — | No |
| Wrong credentials / internal error retry | `WRONG_CREDENTIALS_STATE` / `INTERNAL_ERROR_STATE` | No (a different IP won't fix credentials) |

> **Note:** re-selection only fails over to a different IP if the failed IP is
> actually marked blacklisted by something (e.g. `tm` DNS failover with the core
> `disable_dns_blacklist = 0`, external monitoring, or an MI blacklist rule). If
> nothing populates the blacklist, the retry resolves the same clean set and
> keeps using the same IP, bounded by the existing `failed_attempts` cap.

## Enabling the Feature

### Step 1: Enable in OpenSIPS config

```
modparam("uac_registrant", "enable_blacklist_failover", 1)
```

### Step 2: Make sure a blacklist is actually populated

The check searches the **default** blacklists (`BL_BY_DEFAULT`). In a typical
setup that is the built-in `"dns"` failover blacklist, which is only created and
auto-populated when the **core** global parameter is enabled:

```
disable_dns_blacklist = 0   # default is 1 (off)
```

With this off (`disable_dns_blacklist = 0`), `tm` adds failed destinations to
the `"dns"` blacklist (with an expiry), and the registrant will then avoid those
IPs on the next send.

> If no default blacklist is populated, `enable_blacklist_failover` is a no-op:
> the first resolved address is used, exactly as before.

### Step 3: Restart or reload

Restart OpenSIPS to apply the module parameter.

## Module Parameters

### enable_blacklist_failover (integer, default 0)

Controls whether the pre-send blacklist check / failover is active. Set to `1`
to enable. When `0` (default), the module does not perform any blacklist check
and behaves exactly as before.

```
modparam("uac_registrant", "enable_blacklist_failover", 1)
```

## Scope

- Applies to **REGISTER** only. Un-REGISTER (teardown) is not failover-driven
  and is left unchanged.
- The check is triggered from the periodic timer at the fresh-registration and
  failure-retry points described in *When It Runs* — not on every send. Other
  fresh-register entry points (e.g. the `reg_enable` MI command) are
  intentionally not hooked to keep the change minimal.

## Operational Notes / Caveats

- **Prerequisite:** requires a populated default blacklist to have any effect
  (see Step 2). This is an operational/config requirement, not something the
  module populates on its own.
- **Performance:** with the feature enabled, the next hop is DNS-resolved on
  every send (the same class of resolution `t_uac()` already performs). The
  OpenSIPS DNS cache mitigates the cost. When disabled, no extra resolution is
  done.
- The check runs in the registrant timer process; `mk_proxy()` performs a
  (potentially blocking) DNS lookup, consistent with the existing send path.

## Files Modified

| File | Change |
|------|--------|
| `registrant.c` | Added `enable_blacklist_failover` global + module parameter, added `select_non_blacklisted_dst()` helper, called it (when enabled) from `run_timer_check()` at the fresh-registration and failure-retry (408/503/no-reply) points; added includes for `proxy.h`, `resolve.h`, `blacklists.h` |
| `doc/uac_registrant_admin.xml` | Documented the `enable_blacklist_failover` parameter |
