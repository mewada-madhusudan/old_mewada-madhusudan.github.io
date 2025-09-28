# 🚀 Guide: Bundle React + FastAPI + PyQt6 into One Desktop App

This guide shows how to:  
1. Build a React frontend  
2. Serve it with a FastAPI backend  
3. Launch both inside a PyQt6 desktop app (with backend in a **thread**)  
4. Package everything into a single executable with PyInstaller  

---

## 📂 Project Structure

```
myapp/
│── app.py              # PyQt launcher (runs backend in thread)
│── guide.md            # This guide
│── frontend_build/     # React build (from `npm run build`)
│     ├── index.html
│     ├── assets/
│     └── ...
│── app.spec            # PyInstaller spec file
```

---

## 🔹 Step 1: Build React Frontend

Inside your React project:

```bash
npm run build
```

Copy the generated `dist/` (or `build/`) folder into your Python project as `frontend_build/`.

---

## 🔹 Step 2: Backend (FastAPI)

Instead of keeping a separate `backend.py`, we’ll embed the backend code inside `app.py`.  
This allows running it in a background **thread**.

```python
# backend.py (optional standalone run)
from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles
from fastapi.responses import FileResponse
import os

def create_app():
    app = FastAPI()
    build_path = os.path.join(os.path.dirname(__file__), "frontend_build")

    # Serve React static assets
    app.mount("/assets", StaticFiles(directory=os.path.join(build_path, "assets")), name="assets")

    # Example API
    @app.get("/api/hello")
    async def hello():
        return {"message": "Hello from FastAPI backend (threaded)!"}

    # Serve index.html for frontend routes
    @app.get("/{full_path:path}")
    async def serve_react(full_path: str):
        return FileResponse(os.path.join(build_path, "index.html"))

    return app
```

---

## 🔹 Step 3: PyQt6 App with Threaded Backend

`app.py`

```python
import sys
import threading
import time
from fastapi import FastAPI
import uvicorn
from backend import create_app  # Import from backend.py
from PyQt6.QtWidgets import QApplication, QMainWindow
from PyQt6.QtWebEngineWidgets import QWebEngineView
from PyQt6.QtCore import QUrl

class MainWindow(QMainWindow):
    def __init__(self):
        super().__init__()
        self.browser = QWebEngineView()
        self.browser.setUrl(QUrl("http://127.0.0.1:5000"))
        self.setCentralWidget(self.browser)
        self.setWindowTitle("React + FastAPI + PyQt6 (Threaded)")
        self.resize(1200, 800)

def run_backend():
    app = create_app()
    uvicorn.run(app, host="127.0.0.1", port=5000, log_level="info")

if __name__ == "__main__":
    # Start backend in a thread
    backend_thread = threading.Thread(target=run_backend, daemon=True)
    backend_thread.start()

    # Wait for backend to start
    time.sleep(1.5)

    # Start PyQt app
    app = QApplication(sys.argv)
    window = MainWindow()
    window.show()
    sys.exit(app.exec())
```

---

## 🔹 Step 4: Run It

From the `myapp/` folder:

```bash
python app.py
```

✅ A PyQt window opens and loads your React app.  
✅ Backend runs inside the same process in a thread.  

---

## 🔹 Step 5: Package with PyInstaller

Install PyInstaller:

```bash
pip install pyinstaller
```

Create a `app.spec` file:

```python
# app.spec
# PyInstaller spec file for bundling PyQt + FastAPI + React

block_cipher = None

a = Analysis(
    ['app.py'],
    pathex=[],
    binaries=[],
    datas=[
        ('frontend_build/**/*', 'frontend_build'),  # Include React build
        ('backend.py', '.'),                        # Include backend
    ],
    hiddenimports=[
        'PyQt6.QtWebEngineWidgets',
        'PyQt6.QtWebEngineCore',
        'PyQt6.QtWebEngine',
    ],
    hookspath=[],
    hooksconfig={},
    runtime_hooks=[],
    excludes=[],
    win_no_prefer_redirects=False,
    win_private_assemblies=False,
    cipher=block_cipher,
    noarchive=False,
)

pyz = PYZ(a.pure, a.zipped_data, cipher=block_cipher)

exe = EXE(
    pyz,
    a.scripts,
    [],
    exclude_binaries=True,
    name='myapp',
    debug=False,
    bootloader_ignore_signals=False,
    strip=False,
    upx=True,
    console=False,  # set True if you want backend logs visible
)

coll = COLLECT(
    exe,
    a.binaries,
    a.zipfiles,
    a.datas,
    strip=False,
    upx=True,
    upx_exclude=[],
    name='myapp',
)
```

Build:

```bash
pyinstaller app.spec
```

---

## 🎉 Done!

You now have a **single desktop app** with:  
- React frontend  
- FastAPI backend running in a **thread**  
- PyQt6 embedding everything in one process  

---

👉 Next Steps:  
- Add more API endpoints.  
- Add app icons and splash screen.  
- Optimize PyInstaller packaging.  
