# OpenVPN Daemon Unable to Start/Enable
- After setting up `server.conf`, attempting to enable or start the systemd service fails- seemingly without an error code.
- This can be fixed simply by prepending the `daemon` parameter to the openvpn `server.conf`.
