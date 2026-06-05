import sys
import os
from PyQt6.QtWidgets import QApplication, QMainWindow, QFileDialog, QSystemTrayIcon, QMenu, QMessageBox
from PyQt6.QtCore import QTimer, Qt
from PyQt6.QtWebEngineCore import QWebEngineSettings
from PyQt6.QtWebEngineWidgets import QWebEngineView
import win32gui

CONFIG_FILE = "wallpaper_config.txt"

class LiveWallpaper(QMainWindow):
    def __init__(self):
        super().__init__()
        self.video_path = self.load_config()
        self.init_ui()
        self.setup_tray()
        self.setup_wallpaper()
        
        # [성능 최적화] 1초마다 전체화면 창을 감지하는 타이머
        self.monitor_timer = QTimer(self)
        self.monitor_timer.timeout.connect(self.check_system_activity)
        self.monitor_timer.start(1000)
        
        if self.video_path and os.path.exists(self.video_path):
            self.play_video(self.video_path)
        else:
            QMessageBox.information(
                None, 
                "라이브 월페이퍼", 
                "프로그램을 처음 실행합니다.\n바탕화면으로 사용할 고화질 영상 파일(.mp4)을 선택해주세요!"
            )
            self.change_wallpaper()

    def init_ui(self):
        """[핵심] 크롬 기반의 웹뷰를 생성하여 하드웨어 가속 영상 재생 준비"""
        self.browser = QWebEngineView(self)
        self.setCentralWidget(self.browser)
        
        # 웹뷰 설정 최적화 (하드웨어 가속 활성화 및 비디오 자동 재생 허용)
        settings = self.browser.settings()
        settings.setAttribute(QWebEngineSettings.WebAttribute.LocalContentCanAccessRemoteUrls, True)
        settings.setAttribute(QWebEngineSettings.WebAttribute.PlaybackRequiresUserGesture, False)
        
        # 창 테두리 제거 및 전체 화면 설정
        self.setWindowFlags(Qt.WindowType.FramelessWindowHint)
        self.showFullScreen()

    def setup_wallpaper(self):
        """Windows API 제어 및 WorkerW 레이어 주입 (Python 3.13 호환)"""
        progman = win32gui.FindWindow("Progman", None)
        win32gui.SendMessageTimeout(progman, 0x052C, 0, 0, 0, 1000)
        
        workerw_list = []
        def enum_windows_proc(hwnd, extra):
            shell_dll = win32gui.FindWindowEx(hwnd, 0, "SHELLDLL_DefView", None)
            if shell_dll:
                workerw = win32gui.FindWindowEx(0, hwnd, "WorkerW", None)
                if workerw:
                    workerw_list.append(workerw)
            return True

        win32gui.EnumWindows(enum_windows_proc, None)
        
        if workerw_list:
            workerw = workerw_list
            my_hwnd = int(self.winId())
            win32gui.SetParent(my_hwnd, workerw)

    def check_system_activity(self):
        """[성능 최적화] 전체화면 앱 감지 시 영상 일시정지 명령어 전송"""
        try:
            foreground_hwnd = win32gui.GetForegroundWindow()
            if not foreground_hwnd:
                return

            class_name = win32gui.GetClassName(foreground_hwnd)
            if class_name in ["Progman", "WorkerW", "Shell_TrayWnd"]:
                # 바탕화면으로 돌아오면 영상 다시 재생
                self.browser.page().runJavaScript("document.getElementById('bg-video').play();")
                return

            rect = win32gui.GetWindowRect(foreground_hwnd)
            screen = QApplication.primaryScreen().geometry()
            
            is_full_screen = (rect <= 0 and rect <= 0 and 
                              rect >= screen.width() and rect >= screen.height())

            if is_full_screen:
                # 게임/전체화면 창이 켜지면 영상 정지 (리소스 소모 0%)
                self.browser.page().runJavaScript("document.getElementById('bg-video').pause();")
            else:
                self.browser.page().runJavaScript("document.getElementById('bg-video').play();")
                    
        except Exception:
            pass

    def setup_tray(self):
        self.tray_icon = QSystemTrayIcon(self)
        self.tray_icon.setIcon(self.style().standardIcon(self.style().StandardPixmap.SP_ComputerIcon))
        
        tray_menu = QMenu()
        change_action = QAction("🖼️ 배경 영상 변경...", self)
        change_action.triggered.connect(self.change_wallpaper)
        tray_menu.addAction(change_action)
        
        tray_menu.addSeparator()
        
        quit_action = QAction("❌ 프로그램 종료", self)
        quit_action.triggered.connect(QApplication.instance().quit)
        tray_menu.addAction(quit_action)
        
        self.tray_icon.setContextMenu(tray_menu)
        self.tray_icon.show()

    def change_wallpaper(self):
        file_filter = "고화질 비디오 파일 (*.mp4)"
        file_path, _ = QFileDialog.getOpenFileName(self, "배경 영상 파일 선택", "", file_filter)
        
        if file_path:
            self.video_path = os.path.abspath(file_path)
            self.save_config(self.video_path)
            self.play_video(self.video_path)
        else:
            if not self.video_path:
                sys.exit(0)

    def play_video(self, path):
        """[핵심 원리] HTML5 비디오 태그를 런타임에 생성하여 60FPS 부드러운 재생 구현"""
        # 윈도우 경로 포맷을 웹 규격(URL)으로 변환
        clean_path = path.replace('\\', '/')
        
        # 화면에 꽉 차고, 소리가 없으며(Muted), 무한 반복(Loop)되는 HTML 구조 작성
        html_content = f"""
        <html>
        <head>
            <style>
                body, html {{
                    margin: 0; padding: 0; overflow: hidden; 
                    background-color: black; width: 100%; height: 100%;
                }}
                video {{
                    position: fixed; right: 0; bottom: 0;
                    min-width: 100%; min-height: 100%;
                    width: auto; height: auto; z-index: -100;
                    background-size: cover;
                }}
            </style>
        </head>
        <body>
            <video id="bg-video" autoplay loop muted playsinline>
                <source src="file:///{clean_path}" type="video/mp4">
            </video>
        </body>
        </html>
        """
        self.browser.setHtml(html_content)

    def load_config(self):
        if os.path.exists(CONFIG_FILE):
            with open(CONFIG_FILE, "r", encoding="utf-8") as f:
                return f.read().strip()
        return ""

    def save_config(self, path):
        with open(CONFIG_FILE, "w", encoding="utf-8") as f:
            f.write(path)

if __name__ == "__main__":
    # 웹 엔진 구동을 위한 속성 추가 설정
    os.environ["QTWEBENGINE_DISABLE_SANDBOX"] = "1"
    
    app = QApplication(sys.argv)
    app.setQuitOnLastWindowClosed(False)
    wallpaper = LiveWallpaper()
    sys.exit(app.exec())
