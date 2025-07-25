import sys
import os
import re
from PySide6.QtWidgets import (
    QApplication, QMainWindow, QFileDialog, QTextBrowser,
    QListWidget, QListWidgetItem, QProgressBar, QWidget, QVBoxLayout,
    QHBoxLayout, QSplitter, QTabWidget
)
from PySide6.QtGui import QAction, QKeySequence, QFontDatabase, QFont, QIcon, QPainter, QPen
from PySide6.QtCore import Qt

import ebooklib
from ebooklib import epub
from bs4 import BeautifulSoup

class RulerProgressBar(QProgressBar):
    """یک ویجت نوار پیشرفت سفارشی که خطوط نشانگر فصل را رسم می‌کند."""
    def __init__(self, parent=None):
        super().__init__(parent)
        self._ticks = []

    def set_ticks(self, tick_percentages):
        self._ticks = tick_percentages
        self.update() # درخواست رسم مجدد

    def paintEvent(self, event):
        # ابتدا نوار پیشرفت استاندارد را رسم کن
        super().paintEvent(event)
        
        painter = QPainter(self)
        pen = QPen(Qt.darkGray, 1)
        painter.setPen(pen)
        
        width = self.width()
        height = self.height()
        
        # سپس خطوط نشانگر فصل را روی آن رسم کن
        for percent in self._ticks:
            x = int(width * (percent / 100.0))
            painter.drawLine(x, 0, x, height)

class EpubReader(QMainWindow):
    def __init__(self):
        super().__init__()
        # --- متغیرهای برنامه ---
        self.book = None
        self.chapters = []
        self.library = []
        self.current_book_path = None
        self.app_name = "ePub Swift"
        self.author_name = "GeekNeuron"
        
        # متغیرهای مربوط به نوار پیشرفت جامع
        self.total_book_len = 0
        self.chapter_lens = []
        self.cumulative_lens = []

        # --- بارگذاری و تنظیمات اولیه ---
        self.base_path = os.path.dirname(__file__)
        self.load_assets()
        self.update_window_title()
        self.setWindowState(Qt.WindowMaximized)
        
        # --- راه‌اندازی رابط کاربری ---
        self.init_ui()
        self.apply_styles()

    def init_ui(self):
        # ... (کد منوبار مشابه قبل) ...
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
        # ... (بقیه کد init_ui مشابه قبل با یک تغییر مهم) ...
        self.left_panel = QTabWidget()
        self.left_panel.setMinimumWidth(250)
        self.left_panel.setMaximumWidth(450)
        
        self.toc_list = QListWidget()
        self.toc_list.setAlternatingRowColors(True)
        # اتصال سیگنال تغییر فصل به تابع محاسبه پیشرفت جامع
        self.toc_list.currentItemChanged.connect(self.display_chapter)

        self.library_list = QListWidget()
        self.library_list.setAlternatingRowColors(True)
        self.library_list.itemClicked.connect(self.load_book_from_library)
        
        self.left_panel.addTab(self.toc_list, "فهرست کتاب")
        self.left_panel.addTab(self.library_list, "کتابخانه")
        
        right_panel_widget = QWidget()
        right_layout = QVBoxLayout(right_panel_widget)
        right_layout.setContentsMargins(0, 0, 0, 0)
        right_layout.setSpacing(8)

        self.text_display = QTextBrowser()
        self.text_display.setOpenExternalLinks(True)
        # اتصال سیگنال اسکرول به تابع محاسبه پیشرفت جامع
        self.text_display.verticalScrollBar().valueChanged.connect(self.update_global_progress)

        # استفاده از ویجت سفارشی نوار پیشرفت
        self.progress_bar = RulerProgressBar()
        self.progress_bar.setTextVisible(False)

        right_layout.addWidget(self.text_display)
        right_layout.addWidget(self.progress_bar)

        splitter = QSplitter(Qt.Horizontal)
        splitter.addWidget(self.left_panel)
        splitter.addWidget(right_panel_widget)
        splitter.setStretchFactor(0, 0)
        splitter.setStretchFactor(1, 1)
        splitter.setHandleWidth(2)
        main_layout.addWidget(splitter)
    
    def apply_styles(self):
        self.setStyleSheet("""
            /* ... (استایل‌های قبلی) ... */
            QMenuBar { 
                background-color: #f2f3f7; 
                border-bottom: 1px solid #dcdde1;
                padding: 2px;
            }
            QMenuBar::item {
                background-color: transparent;
                padding: 6px 12px;
                border-radius: 6px; /* گرد کردن گوشه منوهای اصلی */
            }
            QMenuBar::item:selected { /* when hovered */
                background-color: #dcdde1;
            }
            QMenu { background-color: #ffffff; border: 1px solid #dcdde1; }
            QMenu::item:selected { background-color: #345B9A; color: white; }
            /* ... (بقیه استایل‌ها مشابه قبل) ... */
            QMainWindow, QWidget { background-color: #f2f3f7; }
            QTabWidget::pane { border: none; }
            QTabWidget::tab-bar { alignment: center; }
            QTabBar::tab { 
                background: #e1e5ea; color: #555; padding: 8px 20px; 
                border-top-left-radius: 6px; border-top-right-radius: 6px; margin: 0 2px;
            }
            QTabBar::tab:selected { background: #ffffff; color: #000; }
            QListWidget {
                background-color: #ffffff; color: #2c3e50; border: none;
                border-radius: 8px; padding: 5px;
            }
            QListWidget::item { padding: 8px; border-radius: 4px; }
            QListWidget::item:alternate { background-color: #f8f9fa; }
            QListWidget::item:selected { background-color: #345B9A; color: white; }
            #LibraryList QListWidget::item { padding: 12px 8px; }
            QTextBrowser {
                background-color: #ffffff; border: none; border-radius: 8px;
                padding: 20px; font-size: 16px; color: #34495e;
            }
            QProgressBar {
                border: none; border-radius: 4px; background-color: #e0e0e0; max-height: 12px;
            }
            QProgressBar::chunk { background-color: #345B9A; border-radius: 4px; }
            QScrollBar:vertical {
                border: none; background: #f8f9fa; width: 10px; margin: 0; border-radius: 5px;
            }
            QScrollBar::handle:vertical { background: #bdc3c7; min-height: 20px; border-radius: 5px; }
            QScrollBar::handle:vertical:hover { background: #95a5a6; }
        """)
        self.library_list.setObjectName("LibraryList")
    
    def load_book(self, file_path):
        try:
            self.book = epub.read_epub(file_path)
            self.current_book_path = file_path

            book_title_meta = self.book.get_metadata('DC', 'title')
            book_title = book_title_meta[0][0] if book_title_meta else os.path.basename(file_path)
            self.update_window_title(book_title)
            
            self.update_library(file_path, book_title)

            # محاسبه اطلاعات برای نوار پیشرفت جامع
            self.calculate_book_progress_data()

            self.toc_list.clear()
            self.chapters = []
            
            # ... (بقیه کد load_book مشابه قبل) ...
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

    def calculate_book_progress_data(self):
        """محاسبه طول کل کتاب و هر فصل برای نوار پیشرفت جامع (یک بار در هر بارگذاری)."""
        self.total_book_len = 0
        self.chapter_lens = []
        self.cumulative_lens = [0]
        ticks = [0]

        spine_items = [self.book.get_item_with_id(item_id) for item_id, _ in self.book.spine]

        for item in spine_items:
            if item and item.get_type() == ebooklib.ITEM_DOCUMENT:
                content = item.get_content()
                text_len = len(BeautifulSoup(content, 'html.parser').get_text(strip=True))
                self.chapter_lens.append(text_len)
                self.total_book_len += text_len
        
        # محاسبه مجموع تجمعی طول فصل‌ها
        cumulative = 0
        for length in self.chapter_lens:
            cumulative += length
            self.cumulative_lens.append(cumulative)
            if self.total_book_len > 0:
                ticks.append((cumulative / self.total_book_len) * 100)
        
        self.progress_bar.set_ticks(ticks)


    def display_chapter(self, current_item):
        if not current_item or not self.book: return
        self.update_global_progress() # به‌روزرسانی پیشرفت هنگام تغییر فصل

        selected_index = self.toc_list.row(current_item)
        if not (0 <= selected_index < len(self.chapters)): return

        chapter_info = self.chapters[selected_index]
        item = self.book.get_item_with_href(chapter_info['href'])
        content_bytes = item.get_content()
        soup = BeautifulSoup(content_bytes, 'html.parser')
        
        # حذف هرگونه Tooltip پیش‌فرض
        self.text_display.setToolTip("")

        body_text = soup.get_text()
        content_is_rtl = self.is_rtl(body_text)
        direction = "rtl" if content_is_rtl else "ltr"
        
        style_tag = soup.new_tag('style')
        style_tag.string = f"""
            /* اعمال خط جداکننده زیر پاراگراف‌ها */
            p {{
                border-bottom: 1px solid #e0e6f0; /* رنگ آبی بسیار کمرنگ */
                padding-bottom: 8px; /* ایجاد فاصله از خط */
            }}
            /* تنظیم فونت و جهت‌گیری */
            body, p, div, span, li, a, h1, h2, h3, h4 {{ 
                /* فونت وزیرمتن برای فارسی و فونت پیش‌فرض سیستم برای بقیه زبان‌ها */
                font-family: 'Vazirmatn', sans-serif !important; 
                direction: {direction};
                text-align: {"right" if direction == "rtl" else "left"};
            }}
        """
        head = soup.find('head') or soup.new_tag('head')
        if not head.parent:
            soup.insert(0, head)
        head.append(style_tag)

        self.text_display.setHtml(soup.prettify())
        self.text_display.verticalScrollBar().setValue(0)
    
    def update_global_progress(self):
        """محاسبه و به‌روزرسانی نوار پیشرفت جامع بر اساس فصل و اسکرول."""
        if not self.book or self.total_book_len == 0:
            return

        current_chapter_index = self.toc_list.currentRow()
        if current_chapter_index < 0:
            return

        # پیشرفت اسکرول در فصل فعلی
        scrollbar = self.text_display.verticalScrollBar()
        max_val = scrollbar.maximum()
        scroll_progress = (scrollbar.value() / max_val) if max_val > 0 else 0
        
        # محاسبه پیشرفت کلی
        preceding_len = self.cumulative_lens[current_chapter_index]
        current_chapter_len = self.chapter_lens[current_chapter_index]
        
        current_pos_in_book = preceding_len + (scroll_progress * current_chapter_len)
        global_percentage = (current_pos_in_book / self.total_book_len) * 100
        
        self.progress_bar.setValue(int(global_percentage))

    # --- سایر متدها (بدون تغییر زیاد) ---
    def open_file_dialog(self): super().open_file_dialog() if hasattr(super(), 'open_file_dialog') else self.load_book(QFileDialog.getOpenFileName(self, "یک فایل EPUB انتخاب کنید", "", "EPUB Files (*.epub)")[0]) # Dummy implementation, replace with real one
    def load_assets(self): super().load_assets() if hasattr(super(), 'load_assets') else None # Dummy implementation, replace with real one
    def is_rtl(self, text, threshold=0.4): return super().is_rtl(text, threshold) if hasattr(super(), 'is_rtl') else False # Dummy implementation, replace with real one
    def update_window_title(self, book_title=None): super().update_window_title(book_title) if hasattr(super(), 'update_window_title') else None # Dummy implementation, replace with real one
    def update_library(self, file_path, book_title): super().update_library(file_path, book_title) if hasattr(super(), 'update_library') else None # Dummy implementation, replace with real one
    def refresh_library_list(self): super().refresh_library_list() if hasattr(super(), 'refresh_library_list') else None # Dummy implementation, replace with real one
    def load_book_from_library(self, item): super().load_book_from_library(item) if hasattr(super(), 'load_book_from_library') else None # Dummy implementation, replace with real one
# ... (کد متدهای دیگر که تغییر نکرده‌اند اینجا قرار می‌گیرند) ...
# برای اختصار، متدهای بدون تغییر از نسخه قبل در اینجا تکرار نشده‌اند.
# شما باید کد کامل نسخه قبل را داشته باشید و این تغییرات را در آن اعمال کنید.
# در کد نهایی که ارائه می‌شود، همه چیز کامل خواهد بود.

# کد کامل متدهای بدون تغییر برای اطمینان:
def load_assets_full(self):
    font_path = os.path.join(self.base_path, 'assets', 'fonts', 'Vazirmatn-Medium.ttf')
    if os.path.exists(font_path):
        font_id = QFontDatabase.addApplicationFont(font_path)
        if font_id != -1:
            font_families = QFontDatabase.applicationFontFamilies(font_id)
            app_font = QFont(font_families[0], 10)
            QApplication.instance().setFont(app_font)
    icon_path = os.path.join(self.base_path, 'assets', 'icons', 'app_icon.png')
    if os.path.exists(icon_path):
        self.setWindowIcon(QIcon(icon_path))
EpubReader.load_assets = load_assets_full

def is_rtl_full(self, text, threshold=0.4):
    if not text: return False
    clean_text = re.sub('<[^<]+?>', '', text)
    if not clean_text: return False
    sample = clean_text[:500]
    if not sample: return False
    rtl_chars = len(re.findall(r'[\u0600-\u06FF]', sample))
    return (rtl_chars / len(sample)) > threshold
EpubReader.is_rtl = is_rtl_full

def update_window_title_full(self, book_title=None):
    if book_title:
        self.setWindowTitle(f"{book_title} - {self.app_name} ({self.author_name})")
    else:
        self.setWindowTitle(f"{self.app_name} ({self.author_name})")
EpubReader.update_window_title = update_window_title_full

def open_file_dialog_full(self):
    file_path, _ = QFileDialog.getOpenFileName(self, "یک فایل EPUB انتخاب کنید", "", "EPUB Files (*.epub)")
    if file_path:
        self.load_book(file_path)
EpubReader.open_file_dialog = open_file_dialog_full

def update_library_full(self, file_path, book_title):
    if any(b['path'] == file_path for b in self.library): return
    page_count = len(self.book.toc)
    self.library.append({'path': file_path, 'title': book_title, 'pages': page_count, 'progress': 0})
    self.refresh_library_list()
EpubReader.update_library = update_library_full

def refresh_library_list_full(self):
    self.library_list.clear()
    for book in self.library:
        item_text = f"📖 {book['title']}\n📄 فصول: {book['pages']}"
        list_item = QListWidgetItem(item_text)
        list_item.setData(Qt.UserRole, book['path'])
        self.library_list.addItem(list_item)
EpubReader.refresh_library_list = refresh_library_list_full

def load_book_from_library_full(self, item):
    file_path = item.data(Qt.UserRole)
    if file_path and file_path != self.current_book_path:
        self.load_book(file_path)
EpubReader.load_book_from_library = load_book_from_library_full


if __name__ == "__main__":
    app = QApplication(sys.argv)
    reader = EpubReader()
    reader.show()
    sys.exit(app.exec())
