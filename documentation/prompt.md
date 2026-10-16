# The prompt behind this example

This example was built with an AI coding agent in VS Code (Keil Studio with
the CMSIS Developer Assistant, Claude Code) over about two weeks, in a
conversation that went far beyond one prompt: every step was built, loaded
and checked on the board, and what the measurements said shaped the next
step. The text below is that conversation reduced to one prompt, simplified
and shortened, for anyone who wants to make something like it. The commit
messages of the repository record the actual steps in order.

## The prompt

> Fork the Arm example `cmsis-executorch` (ExecuTorch on the Ethos-U through
> a CMSIS solution) into a traffic counter for the Alif Ensemble E7 AppKit
> (AK-E7-AIML, Gen 2): Cortex-M55 HP core with the Ethos-U55-256, the
> kit's MIPI camera and its 480x800 panel.
>
> **Model.** Use Ultralytics YOLO26n. Cut it at its NMS-free one-to-one
> head, keep the five COCO vehicle classes (bicycle, car, motorcycle, bus,
> truck), export it with ExecuTorch's Arm backend and compile it with Vela
> for the Ethos-U55-256 so that every operator runs on the NPU; input
> 416x416 RGB888, int8 with the COCO128 images for calibration. Make the
> outputs anchor-major, as the NPU lays them out, so Vela adds no
> transposes. Keep to the repository's export flow (`create_ai_layer.py`
> from the solution's `mlops:` node); no custom passes, no patched
> backend. Measure the NPU time on the board with its PMU and fix the
> bottleneck you find there, not the one you expect: if the attention
> blocks are what the U55 cannot do well, run them on the CPU, or train a
> variant without them on the COCO images with vehicles and compare the
> vehicle AP50 of float and int8.
>
> **Application.** A small tracker on the Cortex-M55 matches detections to
> tracks by IoU with a constant-velocity prediction, confirms a track after
> three detections and counts it once, per class and direction, when its
> centre crosses a line on the picture. The panel shows the camera picture,
> the boxes, the line and the tallies; the joystick moves the line and its
> centre button turns it. Convert the camera frame while the NPU works on
> the previous one. The console is a log buffer in the DTCM that the
> debugger reads, not a UART.
>
> **Camera.** Bring the kit's camera up on the E7 (find out what module it
> is before writing a driver: probe it over I2C), one snapshot per frame
> into SRAM0, with the gain following the picture's brightness.
>
> **Recording.** Over the board's User USB, use the SDS-Framework to record
> the model inputs and the detections, to play a recorded clip or a video
> back into the detector instead of the camera, to record the panel for a
> video, and to compare the board's detections with the float model's on
> the same inputs on the host.
>
> **Without a board.** Add a Corstone-300 FVP target with the same AI
> layer that runs the test image through the detector and the tracker and
> ends with `Test_result: PASS`, and a GitHub Actions workflow that exports
> the AI layer from a fresh clone, builds both targets and runs the FVP.
>
> **Rules.** Talk to the board only through the CMSIS Developer Assistant
> tools (build, flash, debug, memory, serial), never through a shell
> debugger. Validate every step on the board and keep the firmware
> identical when a change is only a refactor. Do not power-cycle or
> replug the board when a tool can do the job. Keep the licences straight:
> Apache-2.0 for the code, the YOLO26n weights under Ultralytics' AGPL-3.0,
> the ExecuTorch sources under BSD-3-Clause, and no image in the
> repository that is not ours to redistribute. Write the README and a
> technical document that someone with the kit can follow from an empty
> machine, with the measurements, the memory map and what went wrong on
> the way.

## What the answers to it turned into

- The attention of YOLO26n's two PSA blocks took 50 of 80 ms on the
  Ethos-U55 (the U55 has no matrix multiply; the PMU showed its AXI port
  saturated). First the attention moved to the CPU between three NPU
  methods, then YOLO26n without attention was fine-tuned on the COCO images
  with vehicles: 31 ms per frame, 25 fps with the camera, and a better
  vehicle AP50 than the COCO-pretrained model.
- The kit's camera was an MT9M114, not the module the pack's layer named;
  the E7's camera and display DMA do not reach SRAM1.
- SDS went from J-Link RTT to the User USB: a 40-frame clip in 8 s instead
  of 5 s per frame, once the USB buffers sat in the non-secure DTCM region.
- The repository was then cut down to the traffic counter alone, with the
  weights and the pack lock file committed so that a clone builds the same
  image.

The details are in [traffic-counter.md](traffic-counter.md).
