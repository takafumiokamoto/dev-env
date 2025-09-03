# Design Documentation

## 1. Overview

This document outlines the design for a project to create a personal programming environment from scratch. The primary goal is to enable a complete setup with a single command, ensuring a consistent environment across multiple platforms.

## 2. Goals

- **Single-Command Setup:** The entire environment should be configured by executing a single entrypoint script.
- **Platform Agnostic:** The setup must support both Windows (via PowerShell) and Linux (specifically WSL, via Bash/Zsh).
- **Centralized CLI Management:** CLI tool dependencies should be managed primarily through a single tool to ensure consistency and ease of maintenance.
- **Flexibility and Maintainability:** The core setup logic should be written in a way that minimizes code duplication and is easy to extend.
- **Idempotency:** The entire setup process must be safely re-runnable. Subsequent runs should not cause errors or alter an already-configured state. The scripts will install missing components, update existing ones, and create backups of user files where necessary.

## 3. Architecture

The setup process is divided into two distinct phases, creating a clear separation of concerns:

1.  **Bootstrapping (Dispatcher Scripts):** Platform-specific entrypoint scripts (`setup.sh`, `setup.ps1`) are responsible for all package and tool installations. They read dependency lists from plain text files to ensure maintainability.
2.  **Configuration (Python Script):** A lightweight, cross-platform Python script (`src/main.py`) is responsible for the final step of linking configuration files into their correct locations. It is invoked by the dispatcher scripts after they have fully prepared the environment.

### 3.1. Core Components

The project will manage the configuration for the following tools:

- **Terminal:** Alacritty (`alacritty.yml`)
- **Terminal Multiplexer:** Zellij (`config.kdl`)
- **Editor (Terminal & GUI):** Neovim (init.lua) and Neovide (GUI Client)
- **Editor (GUI):** VSCode (`settings.json`, keybindings, extensions)
- **Shell (Linux):** Zsh (`.zshrc` overrides)
- **Shell (Windows):** PowerShell (`Microsoft.PowerShell_profile.ps1`)
- **Fonts:** A designated Nerd Font (e.g., FiraCode).

### 3.2. Dependency Management

Dependency management is handled entirely within the bootstrapping phase by the dispatcher scripts.

- **System-Level Tools:** The dispatcher scripts read package lists from `dependencies/` (e.g., `dependencies/choco.txt`, `dependencies/apt.txt`) and use native package managers (Chocolatey, apt) to install them.
- **CLI Tools (Cross-Platform):** The dispatcher scripts install and run `aqua` using the central `aqua.yaml` file. This `aqua.yaml` file **must** include `uv` to provide a Python runtime.
- **Nerd Fonts:** The dispatcher scripts read `dependencies/nerdfont.txt` and use platform-specific methods to install the font.

### 3.3. Execution Flow

1.  **User Invocation:** The user runs the appropriate entrypoint script (`setup.sh` or `setup.ps1`).
2.  **Bootstrapping:** The script performs the following steps idempotently:
    -   Ensures the native package manager (Chocolatey, apt) is available.
    -   Installs all system packages listed in the `dependencies/*.txt` files.
    -   Installs the Nerd Font.
    -   Installs `aqua`.
    -   Runs `aqua install` to install all tools from `aqua.yaml`, including the `uv` Python runner.
3.  **Configuration:** After all tools are successfully installed, the script makes a final call to the Python script using the `uv` toolchain: `uv run python src/main.py`.
4.  **Python Script Execution:** The Python script performs the idempotent symlinking of configuration files from the `configs/` directory to their system-specific target locations, backing up any pre-existing user files.

### 3.4. Project Structure

The project will be organized with the following directory structure:

```
dev-env/
├── configs/            # Source configuration files for all tools
│   └── ...
├── dependencies/       # Plain text dependency lists
│   ├── apt.txt
│   ├── choco.txt
│   └── nerdfont.txt
├── src/
│   └── main.py         # The core Python application logic for symlinking
├── .gitignore
├── aqua.yaml
├── setup.ps1           # PowerShell bootstrapping script
└── setup.sh            # Bash bootstrapping script
```

## 4. Development and Testing Strategy

The development workflow is straightforward and leverages the immediate feedback of a scripting language.

### 4.1. Core Principle: Shared Filesystem

To ensure the best performance, the project repository **should be cloned onto the Windows filesystem** (e.g., `C:\Users\YourUser\git\dev-env`).

### 4.2. Recommended Workflow

1.  **Clone & Edit:** Clone the repository to a location on your Windows system.
2.  **Modify Dependencies:** Add or remove packages by editing the `aqua.yaml` or the files in the `dependencies/` directory.
3.  **Modify Logic:** Edit the bootstrapping logic in `setup.sh`/`setup.ps1` or the symlinking logic in `src/main.py`.
4.  **Test:** Simply re-run the appropriate setup script (`./setup.sh` or `.\setup.ps1`) to test the changes. The idempotent design ensures that the script will only apply the necessary changes.
