# TinyML & Edge AI

Running a trained model on a microcontroller — no cloud round-trip, no GPU
— is a different discipline from training one. This module covers what
changes when inference has to fit in kilobytes of RAM and run in
microseconds of CPU time: quantization, fixed-point arithmetic, and a
from-scratch inference loop small enough to see every operation it performs.

## Why float32 models don't just "run" on a microcontroller

A typical trained neural network stores weights as 32-bit floats. A modest
model with 50,000 parameters is 200 KB of weights alone — more RAM than many
Cortex-M0/M3 chips have in total, before accounting for activations, stack,
and everything else the firmware needs. **Quantization** — representing
weights (and often activations) as 8-bit integers instead of 32-bit floats
— is not an optional optimization for TinyML, it's usually the difference
between fitting and not fitting at all.

## Quantization: the mapping between float and int8

```c
#include <stdint.h>
#include <math.h>

typedef struct {
    float scale;      /* float_value = (int8_value - zero_point) * scale */
    int8_t zero_point;
} quant_params_t;

int8_t quantize(float value, quant_params_t q) {
    int32_t q_val = (int32_t)roundf(value / q.scale) + q.zero_point;
    if (q_val > 127) q_val = 127;
    if (q_val < -128) q_val = -128;
    return (int8_t)q_val;
}

float dequantize(int8_t value, quant_params_t q) {
    return (value - q.zero_point) * q.scale;
}
```

`scale` and `zero_point` are computed once, offline, from the actual range
of values a given weight tensor or activation takes across a representative
dataset — this is why quantization is normally done as a training-pipeline
step (with calibration data), not something firmware invents at runtime.
Firmware only ever does the cheap integer arithmetic; the expensive
statistical work happens once, on a workstation.

## A minimal quantized inference step: one dense layer

The core of `int8` inference is a multiply-accumulate over quantized
weights and inputs, with the result rescaled back afterward:

```c
/* single dense layer: out[j] = sum_i(in[i] * weight[i][j]) + bias[j],
   computed in int32 to avoid overflow, then requantized to int8 for output */
void dense_layer_int8(const int8_t *input, int in_len,
                       const int8_t *weights,      /* in_len x out_len, row-major */
                       const int32_t *bias, int out_len,
                       quant_params_t in_q, quant_params_t w_q, quant_params_t out_q,
                       int8_t *output) {
    for (int j = 0; j < out_len; j++) {
        int32_t acc = bias[j];
        for (int i = 0; i < in_len; i++) {
            /* int8 x int8 accumulated in int32 — this is the operation that
               makes int8 inference fast: no FPU needed at all */
            acc += (int32_t)input[i] * (int32_t)weights[i * out_len + j];
        }
        float real_value = acc * in_q.scale * w_q.scale;   /* rescale to float domain */
        output[j] = quantize(real_value, out_q);            /* requantize for the next layer */
    }
}
```

Everything inside the inner loop is integer multiply-accumulate — the
reason `int8` inference runs meaningfully faster on a chip with no floating
point unit (many Cortex-M0/M3 parts) than the equivalent float32 computation
would.

## Verifying the quantize/dequantize round-trip and a tiny layer

Compiled and run with `gcc`:

```c
#include <stdio.h>
#include <assert.h>
#include <math.h>
#include <stdint.h>

/* quantize/dequantize/dense_layer_int8 as above */

int main(void) {
    quant_params_t q = { .scale = 0.1f, .zero_point = 0 };

    int8_t qv = quantize(1.5f, q);
    float back = dequantize(qv, q);
    assert(fabsf(back - 1.5f) < 0.1f);   /* within one quantization step */

    /* saturation: a value far outside range clamps instead of wrapping */
    int8_t clamped = quantize(100.0f, q);
    assert(clamped == 127);

    /* tiny 2-input, 1-output dense layer: out = 2*1 + 3*1 + bias(0) */
    int8_t input[2]   = {1, 1};
    int8_t weights[2] = {2, 3};   /* in_len=2, out_len=1 */
    int32_t bias[1]   = {0};
    quant_params_t unit_q = { .scale = 1.0f, .zero_point = 0 };
    int8_t output[1];
    dense_layer_int8(input, 2, weights, bias, 1, unit_q, unit_q, unit_q, output);
    assert(output[0] == 5);

    printf("quantization + dense layer model OK\n");
    return 0;
}
```

## Traps in TinyML on real hardware

- **Overflow in the accumulator**: `int8 x int8` products summed over a
  large layer can exceed `int32` for pathologically large layers or badly
  chosen scale factors — always accumulate in a wider type than the
  product, and sanity-check the maximum possible accumulator value for a
  given layer size against the type's range.
- **Scale/zero_point mismatch between layers**: each layer's output
  quantization parameters must match what the *next* layer was calibrated to
  expect — mixing parameters from different calibration runs silently
  produces plausible-looking but wrong outputs, with no crash to flag it.
- **RAM for activations, not just weights**: weights can live in flash and
  be read directly; intermediate activation buffers must live in RAM and
  are sized by the *largest* single layer's output, not the total model —
  a model can fit in flash and still fail to run for lack of activation RAM.
- **Assuming quantization is free of accuracy loss**: int8 quantization
  typically costs a small, measurable amount of accuracy versus the float32
  original — validating on a held-out dataset after quantization (not just
  "it compiles and produces a plausible-looking number") is part of the
  job, not optional.

## Cheat sheet

| Concept | Detail |
|---|---|
| Quantization | Map float weights/activations to int8 via `scale`/`zero_point`, computed offline |
| Why it matters | Often the difference between a model fitting in MCU flash/RAM and not fitting at all |
| Accumulator width | int8 x int8 products summed in int32 to avoid overflow |
| Requantization | Each layer's int32 accumulator is rescaled back to int8 before the next layer |
| No FPU needed | int8 MAC operations run on cores without a floating point unit |
| RAM sizing | Driven by the largest single layer's activation buffer, not total model size |
| Verification here | Quantize/dequantize and one dense layer compiled/run with `gcc`; no real trained model or NN framework used |

## Exercise

Extend the dense-layer model with a simple ReLU activation
(`max(0, x)`) applied in the int8 domain after requantization, and chain two
dense layers (2 inputs -> 3 hidden -> 1 output) into a tiny hand-built
"network," verifying the full forward pass against a manually computed
expected output using small integer weights you choose yourself. Compile and
run it with `gcc`, and in a comment estimate — from the code alone, without
running on any real chip — the total RAM needed for weights, biases, and
activation buffers for this specific tiny network, showing the arithmetic.
