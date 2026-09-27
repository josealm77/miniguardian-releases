# Astraxis — The Deterministic Robotics Movement & Mission Engine

## Overview

**Astraxis** is a deterministic, mission-verified robotics movement and sensor engine built natively on top of **MiniGuardian**, **Apodeixis**, **Jayce Kernel**, **TRust Terminal**, and **COSMIC Map**.

Unlike conventional robotics stacks (such as ROS2 or moveit) that rely on unverified application-layer motion planning and third-party drivers, Astraxis operates at the OS kernel level directly on top of the **Alien Bridge** (kernel hardware interface) and **Raw Signal Engine** (signal normalizer).

---

## 1. Core Submodules Matrix

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ ASTRAXIS ROBOTICS & SENSOR ENGINE ARCHITECTURE                              │
└─────────────────────────────────────────────────────────────────────────────┘
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                         Developer Movement & Sensor API                 │
  ├───────────────────┬───────────────────┬─────────────────────────────────┤
  │ Astraxis.Controller│  Astraxis.Servo   │         Astraxis.Axis           │
  │ (Orchestration)   │  (Actuator/Pwm)   │      (Kinematics / Arms)        │
  ├───────────────────┼───────────────────┼─────────────────────────────────┤
  │   SensorForge     │   SpectraSense    │           PathSense             │
  │ (Registration &   │ (Waveform Noise   │ (Sensor-to-Movement Spatial     │
  │  Fusion Pipeline) │  Harmonization)   │  Translation & Clearance)       │
  ├───────────────────┼───────────────────┼─────────────────────────────────┤
  │   Astraxis.Path   │Astraxis.Calibrate │         Astraxis.Safety         │
  │ (Trajectories)    │(Zero-point Offset)│   (Boundary & Speed Limit Gate) │
  └─────────┬─────────┴─────────┬─────────┴────────────────┬────────────────┘
            │                   │                          │
            ▼                   ▼                          ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                   Apodeixis Robotics Mission Library                    │
  │   - Movement Missions     - Sensor & Invariants   - Diagnostics         │
  └─────────────────────────────────────────────────────────────────────────┘
```

1. **`Astraxis.Controller`**: High-level movement orchestration (`move_dir`, `rotate`, `follow`).
2. **`Astraxis.Servo`**: Servo & PWM actuator handle control (`set_position`, `home`, 0.0–1.0 bounds).
3. **`Astraxis.Path`**: Waypoint trajectory planning & clearance collision checks.
4. **`Astraxis.Axis`**: Multi-axis spatial kinematics control (`move_to(x, y, z)`).
5. **`SensorForge`**: Sensor registration, calibration, normalization, fusion, and secrecy taint tracking (`@Secret`/`@Public`).
6. **`SpectraSense`**: Advanced signal interpretation for LIDAR, ultrasonic, IR, radar, depth cameras, and tactile sensors (moving-average noise suppression & waveform harmonization).
7. **`PathSense`**: Sensor-to-movement translation evaluating real-time spatial clearance and issuing automatic speed reduction or emergency brakes.
8. **`Astraxis.Safety`**: MiniGuardian-enforced boundary box limits, max velocity limits, and emergency e-stops.
9. **`Astraxis.Diagnostics`**: Real-time thermal monitoring, bus voltage checks, and health reports.

---

## 2. Sensor Processing Example

```rust
use astraxis::{Astraxis, SensorForge, SensorKind, SpectraSense, PathSense};

fn main() -> Result<(), String> {
    let mut astraxis = Astraxis::new();

    // 1. Register & Ingest Raw Alien Bridge Signal
    let mut forge = SensorForge::new();
    forge.register_sensor(1, SensorKind::Lidar);
    let norm_val = forge.ingest_alien_signal(1, 0.45, false)?;

    // 2. Harmonize Waveform Spectrum (LIDAR / Ultrasonic)
    let samples = vec![0.50, 0.45, 0.48, 0.42, 0.44];
    let spectrum = SpectraSense::harmonize_spectrum(SensorKind::Lidar, &samples);

    // 3. Evaluate PathSense Spatial Clearance
    let decision = PathSense::evaluate_spatial_clearance(&spectrum, 1.0, &astraxis.safety);
    println!("Path Decision: {:?} (Recommended Speed: {:.2} m/s)", decision.action_reason, decision.recommended_speed);

    Ok(())
}
```

---

## 3. Why Native Architecture is Superior to ROS2 / External SDKs

| Dimension | External Robotics Stacks (ROS2 / DDS) | Astraxis + Jayce Kernel Architecture |
| :--- | :--- | :--- |
| **Sensor Hardware Access** | Third-party vendor drivers & ROS topics (high latency) | **Kernel-level Alien Bridge** (Direct hardware access, 0 driver bloat) |
| **Signal Processing** | C++ middleware / Python wrappers (unpredictable jitter) | **Kernel Raw Signal Engine** (Deterministic 1,000 Hz waveform harmonization) |
| **Movement Safety** | Application-level checks (prone to runaway motors) | **MiniGuardian OS-level Taint Lattice & Apodeixis Proof Vetoes** |
| **Determinism** | Non-deterministic preemptive scheduling | **Jayce Microkernel round-robin (Zero heap, zero GC pauses)** |
| **Introspection** | Requires external tools (RViz) | **Native COSMIC Map 3D overlay & TRust Terminal REPL** |
