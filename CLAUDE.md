# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is jvepilot, a fork of openpilot - an open source driver assistance system. Openpilot is an operating system for robotics that upgrades the driver assistance system in 300+ supported cars. The project is built primarily in Python and C++ with real-time control systems, computer vision models, and hardware interfaces.

## Development Commands

### Build System
- `scons -j$(nproc)`: Build the entire project using SCons (primary build system)
- `scons --minimal`: Minimal build without tests, tools, etc.
- `scons --asan`: Build with Address Sanitizer for debugging
- `scons --coverage`: Build with test coverage options

### Python Environment & Dependencies  
- `python -m pip install -e .`: Install openpilot package in development mode
- `python -m pip install -e .[testing]`: Install with testing dependencies
- `python -m pip install -e .[dev]`: Install with development dependencies

### Testing
- `pytest`: Run the test suite (configured in pyproject.toml)
- `pytest -n auto`: Run tests in parallel using available CPU cores
- `pytest -m "not slow"`: Skip slow tests
- `pytest selfdrive/`: Run tests for a specific module
- `pytest --timeout=30`: Set test timeout (default configuration)
- `pytest -x`: Stop on first failure

### Code Quality & Linting
- `ruff check`: Run linting (configured in pyproject.toml)
- `ruff format`: Format code
- `mypy`: Type checking
- `codespell`: Spell checking

### Running Individual Tests
- `python -m pytest selfdrive/test/test_onroad.py`: Run a specific test file
- `python -m pytest selfdrive/controls/tests/test_longcontrol.py::test_pid_controller`: Run a specific test method

### Launch Scripts
- `./launch_openpilot.sh`: Launch the full openpilot system
- `./launch_chffrplus.sh`: Alternative launch script
- `python system/manager/manager.py`: Start the process manager directly

## High-Level Architecture

### Core System Components

**Process Manager (`system/manager/`)**
- `manager.py`: Main process orchestrator that manages all system processes
- `process_config.py`: Defines which processes run under different conditions
- Each process has conditions (e.g., car vs. not car, joystick mode, etc.)

**State Machine (`selfdrive/selfdrived/`)**
- `state.py`: Core state machine managing openpilot states (disabled, enabled, soft disabling, etc.)
- `events.py`: Event system for alerts and state transitions
- Handles user inputs, safety events, and system state transitions

**Control Systems (`selfdrive/controls/`)**
- `controlsd.py`: Main control daemon - the "brain" that computes steering/acceleration commands
- `plannerd.py`: Path planning and trajectory generation
- `radard.py`: Radar processing and object tracking
- Integrates with car-specific interfaces through opendbc

**Perception & ML (`selfdrive/modeld/`, `selfdrive/locationd/`)**
- `modeld.py`: Vision model inference for lane lines, objects, road understanding
- `locationd.py`: Sensor fusion and localization
- `calibrationd.py`: Camera calibration
- Uses neural networks (ONNX models) for computer vision

**Hardware Abstraction (`system/hardware/`, `selfdrive/pandad/`)**
- `pandad/`: Interface with the panda device (CAN bus interface)
- `hardware/`: Platform-specific hardware management (comma device, webcam, etc.)
- `camerad/`: Camera capture and processing

**Car Interface (`opendbc_repo/opendbc/car/`)**
- Car-specific implementations for different manufacturers
- CAN message parsing and control signal generation
- Safety monitoring and enforcement

### Data Flow Architecture

1. **Sensors → Perception**: Camera, GPS, IMU data flows to modeld/locationd
2. **Perception → Planning**: Vision model outputs feed into plannerd for path planning  
3. **Planning → Control**: Desired path goes to controlsd for steering/acceleration commands
4. **Control → Car**: Commands sent via pandad to vehicle CAN bus
5. **State Management**: selfdrived monitors everything and manages system state

### Message Passing System

- Uses **cereal** (Cap'n Proto) for inter-process communication
- **ZeroMQ** messaging infrastructure in `msgq/`
- Processes communicate via publish/subscribe pattern
- Each process typically has a main loop reading from SubMaster and writing to PubMaster

### Build System Details

- **SCons** is the primary build system (configured in `SConstruct`)
- Supports multiple architectures: x86_64, aarch64, Darwin (macOS)
- Different build configurations for different hardware (comma device vs. PC)
- **Python packaging** via `pyproject.toml` with multiple dependency groups

### Testing Infrastructure

- **pytest** with custom fixtures for openpilot environment setup
- Tests organized by module with shared test helpers in `selfdrive/test/`
- Hardware-in-the-loop testing capabilities for comma devices
- Process replay system for testing with real driving data

### Key Configuration Files

- `pyproject.toml`: Python packaging, dependencies, tool configuration (pytest, ruff, mypy)
- `SConstruct`: SCons build configuration with platform-specific settings
- `conftest.py`: pytest configuration and custom fixtures
- `process_config.py`: Defines which processes run under different conditions

### Development Workflow Notes

- The system is designed to run on comma devices (ARM-based) but can be developed on PC
- Use `USE_WEBCAM=1` environment variable for development with webcam instead of car
- Process manager handles automatic restarts and dependency management
- All processes are designed to be real-time with specific timing requirements
- Safety is paramount - multiple layers of safety checks in both software and hardware