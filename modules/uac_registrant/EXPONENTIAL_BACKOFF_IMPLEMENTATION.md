# Exponential Backoff Implementation for uac_registrant Module

## Overview
This implementation adds exponential backoff retry logic to the uac_registrant module to improve reliability and reduce server load during registration failures.

## Changes Made

### 1. Configuration Parameters
Added four new module parameters:
- `retry_base_delay` (default: 5 seconds) - Initial retry delay
- `retry_max_delay` (default: 300 seconds) - Maximum retry delay
- `retry_max_attempts` (default: 10) - Maximum number of retry attempts
- `retry_backoff_multiplier` (default: 2) - Multiplier for exponential backoff

### 2. Data Structure Updates
Extended `reg_record_t` structure with:
- `next_retry_time` - Timestamp when next retry should occur
- `current_retry_delay` - Current calculated retry delay in seconds

### 3. Core Functions
- `calculate_exponential_backoff_delay()` - Calculates delay using formula: base_delay * (multiplier ^ attempt)
- `apply_jitter()` - Adds random jitter (up to 25%) to prevent thundering herd

### 4. Retry Logic Updates
Modified `run_timer_check()` function to:
- Check if retry time has arrived before attempting retry
- Calculate exponential backoff delay for each retry attempt
- Reset retry state on successful registration
- Preserve retry state during module reload

## Retry Behavior

### Delay Progression
- Attempt 1: 5 seconds (base delay)
- Attempt 2: 10 seconds (5 * 2^1)
- Attempt 3: 20 seconds (5 * 2^2)
- Attempt 4: 40 seconds (5 * 2^3)
- Attempt 5: 80 seconds (5 * 2^4)
- Attempt 6: 160 seconds (5 * 2^5)
- Attempt 7+: 300 seconds (capped at max_delay)

### Jitter
Each calculated delay has up to 25% random jitter added to prevent multiple registrants from retrying simultaneously.

### State Management
- Retry state is preserved during module reload
- Retry state is reset on successful registration
- Failed attempts counter is incremented on each retry

## Configuration Example
```
modparam("uac_registrant", "retry_base_delay", 10)
modparam("uac_registrant", "retry_max_delay", 600)
modparam("uac_registrant", "retry_max_attempts", 15)
modparam("uac_registrant", "retry_backoff_multiplier", 2)
```

## Benefits
1. **Reduced Server Load**: Longer delays between retries reduce server stress
2. **Better Reliability**: Exponential backoff handles temporary failures gracefully
3. **Thundering Herd Prevention**: Jitter prevents synchronized retries
4. **Configurable**: All parameters can be tuned for specific environments
5. **Backward Compatible**: Existing functionality is preserved

## Debug Information
Enhanced debug logging includes:
- Retry attempt number
- Calculated delay
- Next retry timestamp
- Current retry delay

The implementation maintains full backward compatibility while providing robust retry mechanisms for improved registration reliability.
