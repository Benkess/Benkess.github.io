---
title: 'VECTOR: Real-World Validation'
date: 2026-03-15
permalink: /posts/2026/03/crl-vector/
tags:
  - robotics
  - research
  - multi-agent systems
---

As a research assistant in the UVA Collaborative Robotics Lab, I implemented the real-world validation for the experimental section of an IROS 2026 submission. I translated simulation pipelines into a real multi-robot system, built the full robotics software stack beneath the experiments, and conducted 40 physical trials across four task scenarios.

## Paper Summary

The goal of the paper was to have a decentralized framework that combines constraint-aware peer-to-peer task negotiation, symbolic plan grounding, dependency-based parallel scheduling, and plan verification. The system treats two robot manipulators as independent agents with their own LLM planners. Both robots are given a scene observation, a global task goal, task constraints, and individual robot constraints, all represented symbolically in PDDL. The two LLMs negotiate a plan and communicate their robot's individual constraints. Once they agree on a plan, one of them submits a symbolic representation of the actions each robot will take; the other robot either signs off on the plan or they go back to negotiating. A third LLM reviews the final plan and ensures it will lead to completion of the global goal. During execution, the system monitors the shared symbolic state and responds to environmental changes through targeted recovery strategies such as motion-level adjustment or decentralized replanning. 

The paper is under review, and the code and data will be released upon acceptance. More information can be found on the paper's [website](https://vector-planning.github.io/).

## Direct Contributions to the Paper

### Rewrote the Core Multi-Agent Planner for Physical Hardware

The original codebase was tightly coupled to the simulation. I rewrote the main LangGraph planner for physical hardware deployment. The planner orchestrates the interaction between three LLMs to produce a symbolic set of actions for each robot using PDDL and tool calls. The rewrite included tuning the prompts, tool descriptions, and the overall agent graph.

### Designed and Ran the Real-World Experiments

I helped design the scenario for the real-world validation experiments. The scenario consisted of two robotic manipulators (an FR3 and a Panda), two Kinect cameras, a drawer, and two cubes. The cubes were placed on the table with one closer to each robot. The drawer was near the edge of the workspace, equally distanced from both robots. The goal was to place both cubes in the drawer. Only the FR3 was allowed to open the drawer. There were also three interrupt scenarios:

* **Benign interruption:** I moved a cube to a different location on the table.  
* **Supportive interruption:** I moved a cube into the drawer.  
* **Obstructive interruption:** I moved a cube out of the drawer.

I was the only person running the physical trials: 10 trials per scenario, 40 total. I also wrote the rough draft of the experimental section of the paper.

<div style="display: grid; grid-template-columns: 1fr 1fr; gap: 1.25rem; margin: 1.5rem 0;">
  <div style="text-align: center;">
    <div style="min-height: 34px; display: flex; align-items: center; justify-content: center; border: 1px solid #e5e7eb; background: #f5f5f5; border-radius: 4px; font-size: 0.92rem; line-height: 1.2; padding: 6px 10px; box-sizing: border-box;">Without Interruption</div>
    <video autoplay muted loop playsinline controls style="width: 100%; margin-top: 0.55rem; display: block; background: #000; border-radius: 12px; aspect-ratio: 16/9; object-fit: contain;">
      <source src="/video/put_on_the_drawer_without_interrupt_anonymized.mp4" type="video/mp4">
    </video>
  </div>
  <div style="text-align: center;">
    <div style="min-height: 34px; display: flex; align-items: center; justify-content: center; border: 1px solid #e5e7eb; background: #f5f5f5; border-radius: 4px; font-size: 0.92rem; line-height: 1.2; padding: 6px 10px; box-sizing: border-box;">Benign Interruption</div>
    <video autoplay muted loop playsinline controls style="width: 100%; margin-top: 0.55rem; display: block; background: #000; border-radius: 12px; aspect-ratio: 16/9; object-fit: contain;">
      <source src="/video/put_on_drawer_benign_anonymized.mp4" type="video/mp4">
    </video>
  </div>
  <div style="text-align: center;">
    <div style="min-height: 34px; display: flex; align-items: center; justify-content: center; border: 1px solid #e5e7eb; background: #f5f5f5; border-radius: 4px; font-size: 0.92rem; line-height: 1.2; padding: 6px 10px; box-sizing: border-box;">Supportive Interruption</div>
    <video autoplay muted loop playsinline controls style="width: 100%; margin-top: 0.55rem; display: block; background: #000; border-radius: 12px; aspect-ratio: 16/9; object-fit: contain;">
      <source src="/video/put_on_the_drawer_real_supportive_anonymized.mp4" type="video/mp4">
    </video>
  </div>
  <div style="text-align: center;">
    <div style="min-height: 34px; display: flex; align-items: center; justify-content: center; border: 1px solid #e5e7eb; background: #f5f5f5; border-radius: 4px; font-size: 0.92rem; line-height: 1.2; padding: 6px 10px; box-sizing: border-box;">Obstructive Interruption</div>
    <video autoplay muted loop playsinline controls style="width: 100%; margin-top: 0.55rem; display: block; background: #000; border-radius: 12px; aspect-ratio: 16/9; object-fit: contain;">
      <source src="/video/put_on_the_drawer_real_obstructive_anonymized.mp4" type="video/mp4">
    </video>
  </div>
</div>

### Built the Experimental Infrastructure

Beyond the planner itself, I added several systems to support the experiments:

* **Automatic trial logging** that saves data from every experimental run.  
* **A configurable safety system** that allowed the operator to approve observations, approve actions, and insert pause points during trials. This was primarily used during testing to safely validate the pipeline before running unsupervised trials.  
* All actions in the experiments were executed through the robot skills API I created, and all symbolic scene representations were generated by my VLM-based State Observer (both described below).

## Supporting Software

I also designed and built the software stack beneath the experimental planner.

### VLM State Observer

The experimental framework required a state observer that could process the physical environment into a list of PDDL symbolic predicates. This shared symbolic state was used both for initial planning and for detecting environmental changes that trigger replanning.

I implemented the state observer as a LangGraph agent. The core agent received a camera image and a true-or-false question, which were passed to a VLM. The model responded using a custom Pydantic tool call that produced a value (true, false, or unknown) and a brief justification. The framework was compatible with any model that supports the OpenAI API; I tested it with GPT-4o-mini, Qwen3-VL:4B, and GPT-5.2.

The agent was wrapped in a predicate-observer class that mapped PDDL predicates to natural-language scene questions. It iterated over the list of questions and constructed the symbolic predicates from the VLM responses. For example, the predicate `arm_empty(panda)` mapped to the question: *"Is the robot on the left not holding anything? Answer TRUE if the robot is not holding anything, and FALSE if it is holding an object."* In one trial, the VLM responded with value TRUE and reason: *"The left robot's gripper appears empty and open, with no object between the fingers or attached; objects (colored blocks) are on the table away from the gripper."*

This system needed to be reliable enough to make many observations per experiment without failing. Each successful trial required upwards of 5 observations, each consisting of 12 predicates, totaling over 2,400 individual predictions across the 40 trials. Out of those 40 trials, the observer produced an incorrect observation only twice, resulting in a total of 4 incorrect predicates. In a separate offline evaluation, the state observer scored approximately 94% accuracy on predicate prediction.

### CRL Franka Toolkit

The experimental planner produced symbolic actions. I built a multi-layer robotics software stack to execute those actions on the physical robots. The system was tested on the lab's Franka Research 3 and Franka Panda robots with Kinect cameras for perception.

#### Perception Pipeline

At the base of the stack is a vision pipeline that takes a natural-language object description and produces a 6-DOF pose estimate. It works in several stages:

1. **Language-based segmentation:** I built a GroundingDINO \+ SAM 2 inference server with configurable detection thresholds, morphological post-processing, and area filtering. Given a prompt like "red cube," it produces an instance mask on the RGB image.  
2. **Depth-to-pointcloud conversion:** The masked region is projected into 3D using the depth image and camera intrinsics (pinhole model), producing a pointcloud of the object's visible surface.  
3. **Geometric pose estimation:** For cubes, I implemented a RANSAC-based plane fitting algorithm using Open3D that fits a plane to the pointcloud surface, offsets by half the cube size along the plane normal to find the center, and constructs an orthonormal frame. I also built a bounding-box pipeline that computes oriented bounding boxes and can return the nearest face pose relative to the robot, which was used for drawer handle grasping.

The segmentation model, pointcloud utilities, and geometric pipelines all use pluggable strategy patterns, making them easy to extend with new estimation methods.

#### Coordinate Frame Transformation Chain

Getting a pose from the camera into the robot's frame involved a multi-step transformation chain: camera optical frame, an optional calibration transform for fine-tuning, a TF2 lookup to the world frame, projection to the world XY-plane (zeroing roll and pitch while preserving yaw for gravity-aligned grasping), and an optional world-frame adjustment. All calibration offsets were externalized to YAML configuration files so that tuning never required code changes.

#### Motion Planning and Execution

The pick, place, and pull actions were implemented as C++ MoveIt Task Constructor (MTC) motion planning servers exposed as ROS 2 action servers. They included collision-aware planning using a monitored planning scene for obstacle avoidance. I defined custom ROS 2 action interfaces for each skill with goal, result, and feedback messages.

#### Robot Skills

I combined the perception pipeline, frame transforms, and motion planners into high-level robot skills. Each skill server orchestrates a full pipeline: synchronized RGB \+ depth image capture, segmentation, pointcloud extraction, pose estimation, frame transformation, and MTC execution. The skill servers include an error classification system that distinguishes expected failures (e.g., "object not found," which the LLM planner can reason about) from infrastructure failures (e.g., server crash), allowing the planner to respond intelligently to different failure modes.

#### Language-Agnostic HTTP API Layer

To cleanly decouple the LLM planners from ROS 2, I wrapped every major subsystem in a Flask HTTP server: the segmentation pipeline, the camera and TCP pose queries, and the robot skills themselves. I also wrote pure Python client libraries for each service with no ROS 2 dependencies. This meant the LLM-based planners could call robot actions through simple HTTP requests without any robotics middleware installed, and the entire robot skills stack could be developed and tested independently of the planning framework.

Out of the robot skills I experimented with, the final paper used pick-cube-by-color, place-cube-in-drawer, and open-drawer. This enabled both robots to pick cubes with or without perception, open drawers, and place cubes inside.

## Systems Engineering Details

A few technical decisions that mattered in practice:

* **Thread-safe image synchronization:** RGB and depth streams from the Kinect arrive asynchronously. I used ROS 2 `ApproximateTimeSynchronizer` to match frames within 100ms, with locks protecting shared state for concurrent access.  
* **Polling-based action execution:** ROS 2's `spin_until_future_complete` would deadlock with the background spin thread in multi-threaded executors. I implemented polling loops with explicit timeouts instead.  
* **Single-goal concurrency policy:** Each robot's skill server rejects new goals while a goal is active, preventing unintended concurrent commands.  
* **YAML-driven configuration:** Perception thresholds, calibration offsets, frame names, and timeouts are all externalized, making it possible to calibrate the system without touching code.

## Take-Aways

Working on this research paper with the UVA Collaborative Robotics Lab provided an exciting opportunity to experiment with real laboratory-grade robots. Beyond the research itself, I gained hands-on experience building production-quality robotics software: multi-layer system architecture, real-time perception pipelines, concurrent ROS 2 systems, and the practical challenges of making physical hardware behave reliably. My favorite part was watching the idea transform from a simulation into something that could execute on physical hardware in the real world.  
