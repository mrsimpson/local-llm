# Systemd User Service

Parameterized service template for auto-starting models on boot.

## Install

```bash
cp llama-server@.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now llama-server@qwen3.6-35b-a3b
```

## Usage

```bash
systemctl --user start llama-server@<config-name>
systemctl --user stop llama-server@<config-name>
systemctl --user status llama-server@<config-name>
```

The instance name matches a config filename in `configs/` (without `.conf`).
