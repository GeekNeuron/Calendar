import sys
import os
import re
from PySide6.QtWidgets import (
    QApplication, QMainWindow, QFileDialog, QTextBrowser,
    QListWidget, QListWidgetItem, QProgressBar, QWidget, QVBoxLayout,
    QHBoxLayout, QSplitter, QTabWidget, QLabel
)
from PySide6.QtGui import QAction, QKeySequence, QFontDatabase, QFont, QIcon, QTextOption
from PySide6.QtCore import Qt, QEvent
import ebooklib
from ebooklib import epub
from bs4 import BeautifulSoup

class EpubReader(QMainWindow):
    def __init__(self):
        super().__init__()

        # --- متغیرهای برنامه ---
        self.book = None
        self.chapters = []
        self.library = [] # لیستی برای نگهداری اطلاعات کتاب‌های باز شده
        self.current_book_path = None

        # --- بارگذاری منابع و تنظیمات اولیه ---
        self.base_path = os.path.dirname(__file__)
        self.load_assets()
        self.setWindowTitle("کتاب‌خوان حرفه‌ای EPUB")
        
        # --- راه‌اندازی رابط کاربری ---
        self.init_ui()
        self.apply_styles()

        # پنجره همیشه در حالت Maximize باقی می‌ماند
        self.setWindowState(Qt.WindowMaximized)

    def event(self, event):
        """ Override event handler to keep the window maximized. """
        if event.type() == QEvent.WindowStateChange:
            if not self.isMaximized():
                self.setWindowState(Qt.WindowMaximized)
        return super().event(event)

    def load_assets(self):
        font_path = os.path.join(self.base_path, 'assets', 'fonts', 'Vazirmatn-Medium.ttf')
        if os.path.exists(font_path):
            font_id = QFontDatabase.addApplicationFont(font_path)
            if font_id != -1:
                # فونت اصلی برنامه کمی کوچکتر برای خوانایی بهتر UI
                font_families = QFontDatabase.applicationFontFamilies(font_id)
                app_font = QFont(font_families[0], 10) # فونت کوچکتر برای UI
                QApplication.instance().setFont(app_font)
        
        icon_path = os.path.join(self.base_path, 'assets', 'icons', 'app_icon.png')
        if os.path.exists(icon_path):
            self.setWindowIcon(QIcon(icon_path))

    def init_ui(self):
        menu_bar = self.menuBar()
        file_menu = menu_bar.addMenu("File")
        open_action = QAction("بارگذاری کتاب (Ctrl+O)", self)
        open_action.setShortcut(QKeySequence("Ctrl+O"))
        open_action.triggered.connect(self.open_file_dialog)
        file_menu.addAction(open_action)
        menu_bar.addMenu("Edit")
        menu_bar.addMenu("Settings")

        central_widget = QWidget()
        self.setCentralWidget(central_widget)
        main_layout = QHBoxLayout(central_widget)
        main_layout.setContentsMargins(10, 10, 10, 10)
        main_layout.setSpacing(10)

        # --- پنل سمت چپ با زبانه‌ها (Tabs) ---
        self.left_panel = QTabWidget()
        self.left_panel.setMinimumWidth(250)
        self.left_panel.setMaximumWidth(400)

        # زبانه فصول
        self.toc_list = QListWidget()
        self.toc_list.setAlternatingRowColors(True)
        self.toc_list.currentItemChanged.connect(self.display_chapter)
        self.left_panel.addTab(self.toc_list, "فهرست کتاب")

        # زبانه کتابخانه
        self.library_list = QListWidget()
        self.library_list.itemClicked.connect(self.load_book_from_library)
        self.left_panel.addTab(self.library_list, "کتابخانه")
        
        # --- پنل سمت راست (محتوای کتاب و نوار پیشرفت) ---
        right_panel_widget = QWidget()
        right_layout = QVBoxLayout(right_panel_widget)
        right_layout.setContentsMargins(0, 0, 0, 0)
        right_layout.setSpacing(8)

        self.text_display = QTextBrowser()
        self.text_display.setOpenExternalLinks(True)
        # اتصال اسکرول به نوار پیشرفت
        self.text_display.verticalScrollBar().valueChanged.connect(self.update_line_progress)

        self.progress_bar = QProgressBar()
        self.progress_bar.setTextVisible(False) # متن درصد را مخفی می‌کنیم

        right_layout.addWidget(self.text_display)
        right_layout.addWidget(self.progress_bar)

        # --- چیدمان اصلی با Splitter ---
        splitter = QSplitter(Qt.Horizontal)
        splitter.addWidget(self.left_panel)
        splitter.addWidget(right_panel_widget)
        splitter.setSizes([300, 700])
        splitter.setHandleWidth(2)

        main_layout.addWidget(splitter)

    def apply_styles(self):
        """اعمال استایل شیت (QSS) برای ظاهر مدرن و چشم‌نواز"""
        self.setStyleSheet("""
            QMainWindow, QWidget { background-color: #f2f3f7; }
            QTabWidget::pane { border: none; border-radius: 8px; }
            QTabWidget::tab-bar { alignment: center; }
            QTabBar::tab { 
                background: #e1e5ea; 
                color: #555;
                padding: 8px 20px; 
                border-top-left-radius: 6px;
                border-top-right-radius: 6px;
                margin: 0 2px;
            }
            QTabBar::tab:selected { 
                background: #ffffff; 
                color: #000;
            }
            QListWidget {
                background-color: #ffffff;
                color: #2c3e50;
                border: none;
                border-radius: 8px;
                padding: 5px;
            }
            QListWidget::item { padding: 8px; border-radius: 4px; }
            QListWidget::item:alternate { background-color: #f8f9fa; }
            QListWidget::item:selected { background-color: #345B9A; color: white; }
            QTextBrowser {
                background-color: #ffffff;
                border: none;
                border-radius: 8px;
                padding: 20px;
                font-size: 16px; /* فونت بزرگتر برای متن کتاب */
                color: #34495e;
            }
            QProgressBar {
                border: none;
                border-radius: 4px;
                background-color: #e0e0e0;
                max-height: 8px; /* نوار پیشرفت نازک‌تر */
            }
            QProgressBar::chunk { background-color: #345B9A; border-radius: 4px; }
            QMenuBar { background-color: #f2f3f7; border-bottom: 1px solid #dcdde1; }
            QMenu { background-color: #ffffff; border: 1px solid #dcdde1; }
            QMenu::item:selected { background-color: #345B9A; color: white; }
            QScrollBar:vertical {
                border: none; background: #f8f9fa; width: 10px; margin: 0; border-radius: 5px;
            }
            QScrollBar::handle:vertical { background: #bdc3c7; min-height: 20px; border-radius: 5px; }
            QScrollBar::handle:vertical:hover { background: #95a5a6; }
            QScrollBar::add-line:vertical, QScrollBar::sub-line:vertical { border: none; background: none; }
        """)

    def is_rtl(self, text, threshold=0.4):
        """تشخیص خودکار زبان راست‌چین بر اساس حروف عربی/فارسی"""
        if not text: return False
        # حذف تگ‌های HTML برای تشخیص دقیق‌تر
        clean_text = re.sub('<[^<]+?>', '', text)
        if not clean_text: return False
        
        # بررسی 500 کاراکتر اول کافی است
        sample = clean_text[:500]
        rtl_chars = len(re.findall(r'[\u0600-\u06FF]', sample))
        
        return (rtl_chars / len(sample)) > threshold

    def open_file_dialog(self):
        file_path, _ = QFileDialog.getOpenFileName(self, "یک فایل EPUB انتخاب کنید", "", "EPUB Files (*.epub)")
        if file_path:
            self.load_book(file_path)

    def load_book(self, file_path):
        try:
            self.book = epub.read_epub(file_path)
            self.current_book_path = file_path
            self.update_library(file_path)

            self.toc_list.clear()
            self.chapters = []
            
            # تشخیص جهت کلی کتاب برای فهرست
            first_title = self.book.toc[0].title if self.book.toc else ""
            toc_is_rtl = self.is_rtl(first_title)
            self.toc_list.setLayoutDirection(Qt.RightToLeft if toc_is_rtl else Qt.LeftToRight)

            chapter_counter = 1
            for item in self.book.toc:
                if isinstance(item, epub.Link):
                    self.chapters.append({'title': item.title, 'href': item.href})
                    list_item = QListWidgetItem(f"{chapter_counter}. {item.title}")
                    if toc_is_rtl:
                        list_item.setTextAlignment(Qt.AlignRight)
                    self.toc_list.addItem(list_item)
                    chapter_counter += 1

            if self.toc_list.count() > 0:
                self.toc_list.setCurrentRow(0)
                self.left_panel.setCurrentWidget(self.toc_list)

        except Exception as e:
            self.text_display.setHtml(f"<h1>خطا در باز کردن کتاب</h1><p>{e}</p>")

    def display_chapter(self, current_item):
        if not current_item or not self.book: return
        selected_index = self.toc_list.row(current_item)
        if not (0 <= selected_index < len(self.chapters)): return

        chapter_info = self.chapters[selected_index]
        item = self.book.get_item_with_href(chapter_info['href'])
        content_bytes = item.get_content()
        soup = BeautifulSoup(content_bytes, 'html.parser')
        
        # تشخیص جهت متن فصل
        body_text = soup.get_text()
        content_is_rtl = self.is_rtl(body_text)
        direction = "rtl" if content_is_rtl else "ltr"
        
        # تزریق فونت و جهت‌گیری به محتوا
        style_tag = soup.new_tag('style')
        style_tag.string = f"""
            body, p, div, span, li, a, h1, h2, h3, h4 {{ 
                font-family: 'Vazirmatn', 'Segoe UI', sans-serif !important; 
                direction: {direction};
                text-align: {"right" if direction == "rtl" else "left"};
            }}
        """
        head = soup.find('head') or soup.new_tag('head')
        soup.insert(0, head)
        head.append(style_tag)

        self.text_display.setHtml(soup.prettify())
        # ریست کردن اسکرول به ابتدای فصل
        self.text_display.verticalScrollBar().setValue(0)

    def update_line_progress(self, value):
        """به‌روزرسانی نوار پیشرفت بر اساس موقعیت اسکرول در فصل فعلی"""
        scrollbar = self.text_display.verticalScrollBar()
        max_val = scrollbar.maximum()
        if max_val > 0:
            progress = int((value / max_val) * 100)
            self.progress_bar.setValue(progress)
            
            # به‌روزرسانی درصد پیشرفت در کتابخانه
            if self.current_book_path:
                for book_info in self.library:
                    if book_info['path'] == self.current_book_path:
                        # این یک تخمین ساده است، برای دقت بیشتر باید پیچیده‌تر شود
                        # فعلا پیشرفت فصل فعلی را به عنوان پیشرفت کل در نظر می‌گیریم
                        book_info['progress'] = progress 
                        break
                self.refresh_library_list()

    def update_library(self, file_path):
        """افزودن یا به‌روزرسانی کتاب در کتابخانه"""
        if any(b['path'] == file_path for b in self.library):
            return # کتاب از قبل وجود دارد

        title = self.book.get_metadata('DC', 'title')
        title = title[0][0] if title else os.path.basename(file_path)
        
        # تعداد صفحات را همان تعداد فصول در نظر می‌گیریم
        page_count = len(self.book.toc)

        self.library.append({
            'path': file_path,
            'title': title,
            'pages': page_count,
            'progress': 0
        })
        self.refresh_library_list()

    def refresh_library_list(self):
        """بازрисов لیست کتابخانه"""
        self.library_list.clear()
        for book in self.library:
            item_text = f"نام: {book['title']}\n" \
                        f"تعداد فصل: {book['pages']}\n" \
                        f"پیشرفت: {book['progress']}%"
            list_item = QListWidgetItem(item_text)
            # ذخیره مسیر فایل در آیتم برای دسترسی بعدی
            list_item.setData(Qt.UserRole, book['path'])
            self.library_list.addItem(list_item)
    
    def load_book_from_library(self, item):
        """بارگذاری کتاب با کلیک روی آن در لیست کتابخانه"""
        file_path = item.data(Qt.UserRole)
        if file_path and file_path != self.current_book_path:
            self.load_book(file_path)

if __name__ == "__main__":
    app = QApplication(sys.argv)
    reader = EpubReader()
    reader.show()
    sys.exit(app.exec())
