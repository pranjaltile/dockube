# First Docker Project: First App

This is my first Docker project. It shows how to build and run a simple Python app in a container.

## Docker setup (Windows)
1. Install Docker Desktop for Windows.
2. Start Docker Desktop and wait until it shows **Running**.
3. Ensure it is using Linux containers (tray icon menu).
4. Confirm the daemon is reachable:

```powershell
docker info
```

You should see a **Server** section in the output.

## Project setup
### Files
- `app.py`: Python application entry point.
- `requirements.txt`: Python dependencies.
- `Dockerfile`: Build instructions for the image.

### Build the image
From this folder:

```powershell
docker build -t first_app:v1 .
```

### Run the container

```powershell
docker run -p 5000:5000 first_app:v1
```

### Common issues
- If you see `failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine`, start Docker Desktop and check Linux containers mode.
- If you see `pull access denied`, check the image name and tag (e.g., `first_app:v1`).
