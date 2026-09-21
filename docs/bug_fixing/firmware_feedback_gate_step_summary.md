# Firmware telemetry cadence investigation summary

## Scope
This note captures the results of the Step 1–5 investigation path focused on the firmware-side telemetry cadence problem. It is intentionally limited to the evidence found in the repository and does not propose a firmware edit yet.

## Step 1: feedback gate hypothesis

### What the code does
The telemetry publisher gate is implemented in [General_Driver/ugv_advance.h](../../General_Driver/ugv_advance.h):

- `baseInfoFeedback()` begins with:

```cpp
if (millis() - last_feedback_time < feedbackFlowExtraDelay) {
    return;
}
```

- `last_feedback_time` is updated immediately before the JSON is formatted and emitted.
- The actual publish writes `jsonInfoHttp["T"] = FEEDBACK_BASE_INFO;` and `Serial.println(getInfoJsonString);`

The relevant globals are defined in [General_Driver/ugv_config.h](../../General_Driver/ugv_config.h):

```cpp
bool baseFeedbackFlow = 0;
int feedbackFlowExtraDelay = 0;
```

### Mutation sites
The interval value is set by:

```cpp
void setFeedbackFlowInterval(int inputCmd) {
    feedbackFlowExtraDelay = abs(inputCmd);
}
```

This is reached from the JSON command dispatcher in [General_Driver/uart_ctrl.h](../../General_Driver/uart_ctrl.h):

```cpp
case CMD_FEEDBACK_FLOW_INTERVAL:
    setFeedbackFlowInterval(jsonCmdReceive["cmd"]);
    break;
```

and the command ID is defined in [General_Driver/json_cmd.h](../../General_Driver/json_cmd.h):

```cpp
#define CMD_FEEDBACK_FLOW_INTERVAL 142
```

`CMD_BASE_FEEDBACK_FLOW` also toggles the enable flag:

```cpp
void setBaseInfoFeedbackMode(bool inputCmd) {
    if (inputCmd == 1) {
        baseFeedbackFlow = 1;
    } else if (inputCmd == 0) {
        baseFeedbackFlow = 0;
    }
}
```

### Verdict
This does not look like a built-in non-zero default or a boot-time 7-second gate:

- `baseFeedbackFlow` defaults to `0`
- `feedbackFlowExtraDelay` defaults to `0`
- there is no source-level startup mission assignment found that sets a non-zero interval
- the repo contains no obvious hardcoded 7000 ms feedback interval

However, the runtime command path is real and can absolutely inject a positive delay at any time. So Step 1 rules out a static default gate, but it does not rule out a dynamic runtime gate.

## Step 2: main loop ordering and synchronous blocking

### Loop order
The main loop is in [General_Driver/General_Driver.ino](../../General_Driver/General_Driver.ino):

1. `serialCtrl();`
2. `server.handleClient();`
3. module branch:
   - `moduleType_RoArmM2();`
   - or `moduleType_Gimbal();`
4. `if (runNewJsonCmd) { jsonCmdReceiveHandler(); ... }`
5. speed/PID/IMU work
6. `oledInfoUpdate();`
7. `updateIMUData();`
8. `if (baseFeedbackFlow) { baseInfoFeedback(); }`
9. `heartBeatCtrl();`

### Module-specific work
Gimbal:

- [General_Driver/General_Driver.ino](../../General_Driver/General_Driver.ino) calls `moduleType_Gimbal()`, which invokes `getGimbalFeedback();`
- [General_Driver/gimbal_module.h](../../General_Driver/gimbal_module.h) does two servo feedback transactions per loop

RoArm:

- [General_Driver/General_Driver.ino](../../General_Driver/General_Driver.ino) calls `moduleType_RoArmM2()`, which invokes `RoArmM2_getPosByServoFeedback();`
- [General_Driver/RoArm-M2_module.h](../../General_Driver/RoArm-M2_module.h) does four feedback reads per loop

### Verdict
The telemetry publisher shares the same synchronous main loop as servo-bus reads and motion handlers. This is a strong candidate for bursty, sparse telemetry even when `feedbackFlowExtraDelay` is zero.

## Step 3: blocking operations and timing inventory

### Servo read timeout stack
The critical timeout behavior is in [SCServo/SCSerial.cpp](../../SCServo/SCSerial.cpp):

```cpp
SCSerial::SCSerial()
{
    IOTimeOut = 100;
    pSerial = NULL;
}
```

and:

```cpp
while(1){
    ComData = pSerial->read();
    ...
    if(t_user>IOTimeOut){
        break;
    }
}
```

This means a missing servo reply can stall a read transaction for roughly 100 ms before returning.

### Multiple reads per loop
The observed structure in the code matches the host handoff:

- gimbal path: 2 servo feedback reads per loop
- arm path: 4 servo feedback reads per loop

The bus-layer protocol then wraps each read in a full command/ACK/payload/checksum exchange in [SCServo/SCS.cpp](../../SCServo/SCS.cpp).

### Long-running blocking handlers
The code also contains several blocking loops and delays, including:

- `waitMove2Goal(...)` in [General_Driver/RoArm-M2_module.h](../../General_Driver/RoArm-M2_module.h)
- `missionPlay(...)` in [General_Driver/ugv_advance.h](../../General_Driver/ugv_advance.h)
- direct `delay(...)` calls in motion and init routines

This continuously starves the main loop and therefore suppresses telemetry publication whenever a handler is active.

### Verdict
This strongly supports a shared-thread starvation mechanism as the main firmware-side cause of bursty telemetry cadence.

## Step 4: publish path and request/response behavior

The actual publisher is `baseInfoFeedback()` in [General_Driver/ugv_advance.h](../../General_Driver/ugv_advance.h):

```cpp
jsonInfoHttp["T"] = FEEDBACK_BASE_INFO;
serializeJson(jsonInfoHttp, getInfoJsonString);
Serial.println(getInfoJsonString);
```

The direct request path is in [General_Driver/uart_ctrl.h](../../General_Driver/uart_ctrl.h):

```cpp
case CMD_BASE_FEEDBACK:
    baseInfoFeedback();
    break;
```

The periodic flow path is in [General_Driver/General_Driver.ino](../../General_Driver/General_Driver.ino):

```cpp
if (baseFeedbackFlow) {
    baseInfoFeedback();
}
```

The key detail is that both paths share the same loop and therefore are sensitive to earlier blocking work. A host request does not guarantee immediate emission when the main loop is occupied by servo or mission work.

## Step 5: serial/bus scheduling behavior

The bus layer is the clearest root-cause candidate for the observed cadence:

- `SCSerial::readSCS()` has a 100 ms timeout loop
- failed servo reads are expensive and synchronous
- multiple servo reads happen in a single main-loop pass
- telemetry is emitted only after the loop reaches the publish branch
- mission/motion handlers can delay the loop further

This explains the bursty valid replies and the long gaps between them without requiring a fixed 7-second timer in source.

## Most likely firmware mechanism
The strongest repository-backed interpretation is:

1. telemetry publishing is not a dedicated background task
2. servo-bus reads are synchronous and can wait ~100 ms per timeout
3. gimbal/arm loops perform multiple such reads each cycle
4. mission or motion handlers can block the loop for much longer
5. `baseInfoFeedback()` is emitted only when the loop eventually reaches it
6. therefore the host observes a sparse, bursty telemetry cadence

This is a firmware-side cadence issue caused by loop starvation and bus timeout behavior, not just host polling.

## Remaining unmeasured item
The repository does not show a source-level fixed 7-second feedback gate or a startup mission that writes a non-zero interval. That means the remaining unmeasured check is whether a runtime command or external mission state is setting a positive `feedbackFlowExtraDelay` value dynamically, or whether the observed ~7s pattern is entirely a product of loop starvation and timeouts.

## Suggested next steps

### 1) Confirm runtime command/config state
- Capture the actual runtime value of `feedbackFlowExtraDelay` and `baseFeedbackFlow` on the target
- Check whether a boot mission, web UI, or a host command is reintroducing a positive interval value
- If the value is zero, treat the loop starvation hypothesis as primary

### 2) Instrument the firmware minimally
Add low-rate, compact timing markers around:
- entry to `baseInfoFeedback()`
- exit from `getGimbalFeedback()` / `RoArmM2_getPosByServoFeedback()`
- before/after each `st.FeedBack(...)` call
- time spent in `missionPlay()` or long `delay(...)` branches

This should be minimal and low-rate to avoid perturbing timing.

### 3) Measure the loop budget directly
Log:
- loop start timestamp
- loop end timestamp
- total time spent in module feedback read branches
- total time spent in mission or motion handlers

This will confirm whether telemetry is simply delayed behind bus read timeouts.

### 4) Gate test
Force the runtime interval to zero and compare the burst gap pattern with the prior run.

- If the gap pattern collapses, the feedback gate is active at runtime.
- If it remains, starvation/bus blocking is still the primary explanation.

### 5) Bus timing test
Measure individual `st.FeedBack(...)` calls and observe whether timeout clusters correlate with the observed response gaps.

## Suggested acceptance criteria for a fix
A firmware-side fix is considered viable when telemetry-only validation shows:

- no recurring ~7s fresh-sample ceiling
- median inter-success gap under 0.5 s
- reply latency under 500 ms for fresh samples
- no repeated stale pan value dominating the run
- gimbal `T=1005` diagnostics remain absent during normal operation

## Summary
The code strongly supports a firmware-side telemetry cadence problem caused by synchronous servo-bus reads and a shared main loop, with a possible runtime feedback interval gate as a secondary dynamic factor. The repository does not show a built-in fixed gate, but it does show a loop architecture and bus timeout behavior that can readily produce the observed sparse, bursty cadence.
