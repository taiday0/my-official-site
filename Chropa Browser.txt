import sys
import sqlite3
import os
from PyQt5.QtCore import QUrl, QSize
from PyQt5.QtWidgets import (QApplication, QMainWindow, QToolBar, QAction, 
                             QLineEdit, QTabWidget, QProgressBar, QStatusBar,
                             QDialog, QVBoxLayout, QListWidget, QListWidgetItem, 
                             QPushButton, QInputDialog, QFileDialog, QMessageBox)
from PyQt5.QtWebEngineWidgets import QWebEngineView, QWebEngineProfile
from PyQt5.QtWebEngineCore import QWebEngineUrlRequestInterceptor
from PyQt5.QtGui import QKeySequence

# --- データベースの設定 (履歴保存用) ---
# 実行しているユーザーのホームフォルダ（例: /Users/taiga）を自動取得し、デスクトップのパスを作ります
DB_NAME = os.path.join(os.path.expanduser("~"), "Desktop", "chropa_history.db")

def init_db():
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS history (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT,
            url TEXT,
            timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    """)
    conn.commit()
    conn.close()

def save_history(title, url):
    if not title or title == "新しいタブ" or url.startswith("data:"):
        return
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("SELECT url FROM history ORDER BY id DESC LIMIT 1")
    last_url = cursor.fetchone()
    if not last_url or last_url[0] != url:
        cursor.execute("INSERT INTO history (title, url) VALUES (?, ?)", (title, url))
        conn.commit()
    conn.close()

# --- 履歴表示用ダイアログ ---
class HistoryDialog(QDialog):
    def __init__(self, parent=None):
        super().__init__(parent)
        self.setWindowTitle("閲覧履歴 - Chropa")
        self.resize(500, 400)
        
        layout = QVBoxLayout()
        self.list_widget = QListWidget()
        layout.addWidget(self.list_widget)
        
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute("SELECT title, url FROM history ORDER BY id DESC LIMIT 100")
        self.history_items = cursor.fetchall()
        conn.close()
        
        for title, url in self.history_items:
            item_text = f"{title} ({url})" if title else url
            item = QListWidgetItem(item_text)
            self.list_widget.addItem(item)
            
        self.list_widget.itemDoubleClicked.connect(self.accept)
        
        clear_btn = QPushButton("履歴をすべて削除")
        clear_btn.clicked.connect(self.clear_history)
        layout.addWidget(clear_btn)
        
        self.setLayout(layout)
        self.selected_url = None

    def accept(self):
        current_row = self.list_widget.currentRow()
        if current_row >= 0:
            self.selected_url = self.history_items[current_row][1]
        super().accept()

    def clear_history(self):
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()
        cursor.execute("DELETE FROM history")
        conn.commit()
        conn.close()
        self.list_widget.clear()
        self.history_items = []

# --- 🛡️ 広告ブロック ---
class AdBlockInterceptor(QWebEngineUrlRequestInterceptor):
    def __init__(self):
        super().__init__()
        self.ad_domains = [
            "googleads", "doubleclick", "adservice", "pagead", "adnow",
            "popads", "analytics", "scorecardresearch", "adnxs", "adskeeper",
            "amazon-adsystem", "criteo", "outbrain", "taboola"
        ]

    def interceptRequest(self, info):
        url_string = info.requestUrl().toString().lower()
        for ad_domain in self.ad_domains:
            if ad_domain in url_string:
                info.block(True)
                return

# --- 🚀 メインブラウザクラス「Chropa」 ---
class ChropaBrowser(QMainWindow):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Chropa Browser")
        self.setGeometry(100, 100, 1200, 800)

        # 自作設定: ダークモードのオン/オフ状態
        self.dark_mode_enabled = False

        self.interceptor = AdBlockInterceptor()
        QWebEngineProfile.defaultProfile().setUrlRequestInterceptor(self.interceptor)

        self.tabs = QTabWidget()
        self.tabs.setDocumentMode(True)
        self.tabs.tabBarDoubleClicked.connect(self.tab_open_doubleclick)
        self.tabs.currentChanged.connect(self.current_tab_changed)
        self.tabs.setTabsClosable(True)
        self.tabs.tabCloseRequested.connect(self.close_current_tab)
        self.setCentralWidget(self.tabs)

        self.status = QStatusBar()
        self.setStatusBar(self.status)
        self.progress_bar = QProgressBar()
        self.progress_bar.setMaximumWidth(150)
        self.progress_bar.setMaximumHeight(15)
        self.status.addPermanentWidget(self.progress_bar)

        nav_bar = QToolBar("Navigation")
        nav_bar.setIconSize(QSize(16, 16))
        self.addToolBar(nav_bar)

        back_btn = QAction("←", self)
        back_btn.triggered.connect(lambda: self.active_browser().back() if self.active_browser() else None)
        nav_bar.addAction(back_btn)

        forward_btn = QAction("→", self)
        forward_btn.triggered.connect(lambda: self.active_browser().forward() if self.active_browser() else None)
        nav_bar.addAction(forward_btn)

        reload_btn = QAction("⟳", self)
        reload_btn.triggered.connect(lambda: self.active_browser().reload() if self.active_browser() else None)
        nav_bar.addAction(reload_btn)

        home_btn = QAction("🏠", self)
        home_btn.triggered.connect(self.navigate_home)
        nav_bar.addAction(home_btn)

        new_tab_btn = QAction("➕", self)
        new_tab_btn.triggered.connect(lambda: self.add_new_tab())
        nav_bar.addAction(new_tab_btn)

        history_btn = QAction("📜 履歴", self)
        history_btn.triggered.connect(self.show_history_dialog)
        nav_bar.addAction(history_btn)

        js_btn = QAction("⚡ 手動JS", self)
        js_btn.triggered.connect(self.inject_javascript)
        nav_bar.addAction(js_btn)

        # 【自作機能】ダークモード切替ボタン (Ctrl+D)
        self.dark_btn = QAction("🌙 ダーク", self)
        self.dark_btn.setShortcut(QKeySequence("Ctrl+D"))
        self.dark_btn.triggered.connect(self.toggle_dark_mode)
        nav_bar.addAction(self.dark_btn)

        # 【自作機能】スクリーンショットボタン (Ctrl+S)
        screenshot_btn = QAction("📸 保存", self)
        screenshot_btn.setShortcut(QKeySequence("Ctrl+S"))
        screenshot_btn.triggered.connect(self.capture_screenshot)
        nav_bar.addAction(screenshot_btn)

        self.url_bar = QLineEdit()
        self.url_bar.returnPressed.connect(self.navigate_to_url)
        nav_bar.addWidget(self.url_bar)

        self.add_new_tab(QUrl("https://www.google.com"), "ホーム")

    def inject_javascript(self):
        if not self.active_browser():
            return
        default_js = "document.body.style.backgroundColor = 'hsl(' + Math.random() * 360 + ', 70%, 70%)';"
        text, ok = QInputDialog.getMultiLineText(self, "JavaScriptの実行", "実行したいJavaScript:", default_js)
        if ok and text:
            self.active_browser().page().runJavaScript(text)

    # 【自作機能】ダークモードのオンオフを切り替える関数
    def toggle_dark_mode(self):
        self.dark_mode_enabled = not self.dark_mode_enabled
        self.dark_btn.setText("☀️ ライト" if self.dark_mode_enabled else "🌙 ダーク")
        
        # 開いているすべてのタブに即座に反映
        for i in range(self.tabs.count()):
            browser = self.tabs.widget(i)
            if browser:
                self.apply_dark_mode_style(browser)

    # 【自作機能】JavaScriptを使ってWebページをダークモード化する関数
    def apply_dark_mode_style(self, browser):
        if self.dark_mode_enabled:
            dark_js = """
            (function() {
                let style = document.getElementById('chropa-dark-mode');
                if (!style) {
                    style = document.createElement('style');
                    style.id = 'chropa-dark-mode';
                    style.innerHTML = 'html { filter: invert(0.95) hue-rotate(180deg) !important; background: #fff; } img, video, canvas { filter: invert(1) hue-rotate(180deg) !important; }';
                    document.head.appendChild(style);
                }
            })();
            """
            browser.page().runJavaScript(dark_js)
        else:
            remove_dark_js = """
            (function() {
                let style = document.getElementById('chropa-dark-mode');
                if (style) { style.remove(); }
            })();
            """
            browser.page().runJavaScript(remove_dark_js)

    # 【自作機能】現在表示しているページを画像として保存する関数
    def capture_screenshot(self):
        browser = self.active_browser()
        if not browser:
            return
        
        options = QFileDialog.Options()
        file_path, _ = QFileDialog.getSaveFileName(
            self, "スクリーンショットを保存", "screenshot.png", 
            "PNG Images (*.png);;All Files (*)", options=options
        )
        
        if file_path:
            # 画面をキャプチャして画像として保存
            pixmap = browser.grab()
            pixmap.save(file_path, "PNG")
            self.status.showMessage(f"保存しました: {os.path.basename(file_path)}", 3000)

    def add_new_tab(self, qurl=None, label="新しいタブ"):
        if qurl is None:
            qurl = QUrl("https://www.google.com")

        browser = QWebEngineView()
        browser.setUrl(qurl)
        
        i = self.tabs.addTab(browser, label)
        self.tabs.setCurrentIndex(i)

        # 【バグ修正】動的インデックス取得に変更
        browser.urlChanged.connect(lambda qurl, b=browser: self.update_urlbar(qurl, b))
        browser.loadFinished.connect(lambda _, b=browser: self.handle_load_finished(b))
        browser.iconChanged.connect(lambda icon, b=browser: self.tabs.setTabIcon(self.tabs.indexOf(b), icon))
        browser.loadProgress.connect(lambda val, b=browser: self.progress_bar.setValue(val) if b == self.active_browser() else None)
        browser.page().linkHovered.connect(lambda url: self.status.showMessage(url))

    def handle_load_finished(self, browser):
        i = self.tabs.indexOf(browser)
        if i == -1:
            return
            
        title = browser.page().title()
        url = browser.url().toString()
        self.tabs.setTabText(i, title[:15] if title else "無題のページ")
        
        if browser == self.active_browser():
            self.setWindowTitle(f"{title} - Chropa")
        save_history(title, url)

        # 【自作機能】ロード完了時にダークモードの状態を適用
        self.apply_dark_mode_style(browser)

        # 自動JavaScript流し込みの土台
        self.auto_execute_js(browser, url)

    def auto_execute_js(self, browser, url):
        """特定のサイトを開いたときに自動実行したいJSがあれば、ここに記述します"""
        pass

    def show_history_dialog(self):
        if not self.active_browser():
            return
        dialog = HistoryDialog(self)
        if dialog.exec_():
            if dialog.selected_url:
                self.active_browser().setUrl(QUrl(dialog.selected_url))

    def tab_open_doubleclick(self, i):
        if i == -1: 
            self.add_new_tab()

    def close_current_tab(self, i):
        if self.tabs.count() < 2: 
            return
        self.tabs.removeTab(i)

    def current_tab_changed(self, i):
        browser = self.active_browser()
        if browser:
            qurl = browser.url()
            self.update_urlbar(qurl, browser)
            self.setWindowTitle(f"{browser.page().title()} - Chropa")
            # タブが切り替わったときも現在の状態に合わせてダークモードを再適用
            self.apply_dark_mode_style(browser)
        else:
            self.setWindowTitle("Chropa Browser")

    def active_browser(self):
        return self.tabs.currentWidget()

    def navigate_home(self):
        if self.active_browser():
            self.active_browser().setUrl(QUrl("https://www.google.com"))

    def navigate_to_url(self):
        if not self.active_browser():
            return
        text = self.url_bar.text().strip()
        if not text:
            return

        if "." not in text or " " in text:
            qurl = QUrl(f"https://www.google.com/search?q={text}")
        elif not text.startswith("http://") and not text.startswith("https://"):
            qurl = QUrl("https://" + text)
        else:
            qurl = QUrl(text)
        self.active_browser().setUrl(qurl)

    def update_urlbar(self, q, browser=None):
        if browser == self.active_browser():
            self.url_bar.setText(q.toString())

if __name__ == "__main__":
    init_db()
    app = QApplication(sys.argv)
    window = ChropaBrowser()
    window.show()
    sys.exit(app.exec_())
