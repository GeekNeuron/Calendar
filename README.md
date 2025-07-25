import sys
import os
import re
from PySide6.QtWidgets import (
    QApplication, QMainWindow, QFileDialog, QTextBrowser,
    QListWidget, QListWidgetItem, QProgressBar, QWidget, QVBoxLayout,
    QHBoxLayout, QSplitter, QTabWidget
)
from PySide6.QtGui import QAction, QKeySequence, QFontDatabase, QFont, QIcon
from PySide6.QtCore import Qt, QThread, QObject, Signal

import ebooklib
from ebooklib import epub
from bs4 import BeautifulSoup

# --- کلاس Worker برای پردازش کتاب در پس‌زمینه (بدون محاسبه خط‌کش) ---
class BookLoaderWorker(QObject):
    finished = Signal(dict)

    def __init__(self, book):
        super().__init__()
        self.book = book

    def run(self):
        total_len = 0
        chap_lens = []
        cum_lens = [0]
        try:
            spine_items = [self.book.get_item_with_id(item_id) for item_id, _ in self.book.spine]
            for item in spine_items:
                if item and item.get_type() == ebooklib.ITEM_DOCUMENT:
                    content = item.get_content()
                    text_len = len(BeautifulSoup(content, 'html.parser').get_text(strip=True))
                    chap_lens.append(text_len)
                    total_len += text_len
            
            cumulative = 0
            for length in chap_lens:
                cumulative += length
                cum_lens.append(cumulative)
            
            result = {'total_len': total_len, 'chap_lens': chap_lens, 'cum_lens': cum_lens}
            self.finished.emit(result)
        except Exception as e:
            self.finished.emit({'error': str(e)})


# --- کلاس اصلی برنامه ---
class EpubReader(QMainWindow):
    def __init__(self):
        super().__init__()
        self.book = None
        self.chapters = []
        self.library = []
        self.current_book_path = None
        self.app_name = "ePub Swift"
        self.author_name = "GeekNeuron"
        
        self.total_book_len = 0
        self.chapter_lens = []
        self.cumulative_lens = []

        self.base_path = os.path.dirname(__file__)
        self.load_assets()
        self.update_window_title()
        self.setWindowState(Qt.WindowMaximized)
        
        self.init_ui()
        self.apply_styles()
        self.worker_thread = None

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
        
        self.left_panel = QTabWidget()
        self.left_panel.setMinimumWidth(250)
        self.left_panel.setMaximumWidth(450)
        
        self.toc_list = QListWidget()
        self.toc_list.setAlternatingRowColors(True)
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
        self.text_display.verticalScrollBar().valueChanged.connect(self.update_global_progress)
        
        # استفاده از نوار پیشرفت استاندارد
        self.progress_bar = QProgressBar()
        self.progress_bar.setTextVisible(True) # نمایش متن درصد
        
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
            QProgressBar {
                border: none;
                border-radius: 4px;
                background-color: #e0e0e0;
                max-height: 14px; /* کمی ارتفاع بیشتر برای نمایش بهتر متن */
                text-align: center; /* مرکزچین کردن درصد */
                color: white; /* رنگ متن درصد */
                font-size: 9px; /* کوچک کردن فونت درصد */
                font-weight: bold;
            }
            QProgressBar::chunk {
                background-color: #345B9A;
                border-radius: 4px;
            }
            QMenu { 
                background-color: #ffffff; 
                border: 1px solid #dcdde1;
                border-radius: 8px;
                padding: 5px;
            }
            QMenu::item {
                padding: 8px 25px 8px 20px;
                border-radius: 6px;
            }
            QMenu::item:selected { background-color: #345B9A; color: white; }
            
            QMenuBar { 
                background-color: #f2f3f7; border-bottom: 1px solid #dcdde1; padding: 2px;
            }
            QMenuBar::item {
                background-color: transparent; padding: 6px 12px; border-radius: 6px;
            }
            QMenuBar::item:selected { background-color: #dcdde1; }
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
            QTextBrowser {
                background-color: #ffffff; border: none; border-radius: 8px;
                padding: 20px; font-size: 16px; color: #34495e;
            }
            QScrollBar:vertical {
                border: none; background: #f8f9fa; width: 10px; margin: 0; border-radius: 5px;
            }
            QScrollBar::handle:vertical { background: #bdc3c7; min-height: 20px; border-radius: 5px; }
            QScrollBar::handle:vertical:hover { background: #95a5a6; }
        """)

    def load_book(self, file_path):
        if not file_path: return
        QApplication.setOverrideCursor(Qt.WaitCursor)
        self.toc_list.clear()
        self.text_display.clear()
        self.progress_bar.setValue(0)

        try:
            self.book = epub.read_epub(file_path)
            self.current_book_path = file_path
            
            self.worker_thread = QThread()
            self.worker = BookLoaderWorker(self.book)
            self.worker.moveToThread(self.worker_thread)
            
            self.worker_thread.started.connect(self.worker.run)
            self.worker.finished.connect(self.on_book_data_loaded)
            self.worker.finished.connect(self.worker_thread.quit)
            self.worker.finished.connect(self.worker.deleteLater)
            self.worker_thread.finished.connect(self.worker_thread.deleteLater)
            
            self.worker_thread.start()

            book_title_meta = self.book.get_metadata('DC', 'title')
            book_title = book_title_meta[0][0] if book_title_meta else os.path.basename(file_path)
            self.update_window_title(book_title)
            self.update_library(file_path, book_title)

            self.chapters = []
            
            toc_is_rtl = self.is_rtl(self.book.toc[0].title if self.book.toc else "")
            self.toc_list.setLayoutDirection(Qt.RightToLeft if toc_is_rtl else Qt.LeftToRight)
            
            for item in self.book.toc:
                if isinstance(item, epub.Link):
                    self.chapters.append({'title': item.title, 'href': item.href})
            self.toc_list.addItems([f"{i+1}. {c['title']}" for i, c in enumerate(self.chapters)])

            if self.toc_list.count() > 0:
                self.toc_list.setCurrentRow(0)
                self.left_panel.setCurrentWidget(self.toc_list)

        except Exception as e:
            self.text_display.setHtml(f"<h1>خطا در باز کردن کتاب</h1><p>{e}</p>")
            QApplication.restoreOverrideCursor()

    def on_book_data_loaded(self, result):
        if 'error' not in result:
            self.total_book_len = result['total_len']
            self.chapter_lens = result['chap_lens']
            self.cumulative_lens = result['cum_lens']
            self.update_global_progress()
        QApplication.restoreOverrideCursor()

    def display_chapter(self, current_item):
        if not current_item or not self.book: return
        self.update_global_progress()
        
        selected_index = self.toc_list.row(current_item)
        if not (0 <= selected_index < len(self.chapters)): return
        
        chapter_info = self.chapters[selected_index]
        item = self.book.get_item_with_href(chapter_info['href'])
        content_bytes = item.get_content()
        soup = BeautifulSoup(content_bytes, 'html.parser')
        
        self.text_display.setToolTip("")
        
        body_text = soup.get_text()
        content_is_rtl = self.is_rtl(body_text)
        direction = "rtl" if content_is_rtl else "ltr"
        
        # --- اصلاح اصلی برای جهت‌گیری و خط‌کشی ---
        # 1. افزودن ویژگی 'dir' به تگ body برای جهت‌گیری قوی‌تر
        body_tag = soup.find('body')
        if body_tag:
            body_tag['dir'] = direction

        # 2. تزریق استایل با سلکتور عمومی‌تر برای خط‌کشی
        style_tag = soup.new_tag('style')
        style_tag.string = f"""
            /* اعمال خط جداکننده به تمام فرزندان مستقیم body */
            body > * {{
                border-bottom: 1px solid #e8eaf6; /* رنگ آبی بسیار کمرنگ و ملایم */
                padding-bottom: 0.5em; 
                margin-bottom: 0.5em;
            }}
            body {{ 
                font-family: 'Vazirmatn', sans-serif !important; 
            }}
        """
        head = soup.find('head') or soup.new_tag('head')
        if not head.parent: soup.insert(0, head)
        head.append(style_tag)
        
        self.text_display.setHtml(soup.prettify())
        self.text_display.verticalScrollBar().setValue(0)

    def update_global_progress(self):
        if not self.book or self.total_book_len == 0 or self.toc_list.currentRow() < 0:
            return
        
        current_chapter_index = self.toc_list.currentRow()
        
        scrollbar = self.text_display.verticalScrollBar()
        max_val = scrollbar.maximum()
        scroll_progress = (scrollbar.value() / max_val) if max_val > 0 else 0
        
        preceding_len = self.cumulative_lens[current_chapter_index]
        current_chapter_len = self.chapter_lens[current_chapter_index]
        
        current_pos_in_book = preceding_len + (scroll_progress * current_chapter_len)
        global_percentage = (current_pos_in_book / self.total_book_len) * 100
        
        self.progress_bar.setValue(int(global_percentage))

    # --- سایر متدها بدون تغییر ---
    def load_assets(self):
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

    def is_rtl(self, text, threshold=0.4):
        if not text: return False
        clean_text = re.sub('<[^<]+?>', '', text)
        if not clean_text: return False
        sample = clean_text[:500]
        if not sample: return False
        rtl_chars = len(re.findall(r'[\u0600-\u06FF]', sample))
        return (rtl_chars / len(sample)) > threshold

    def update_window_title(self, book_title=None):
        if book_title:
            self.setWindowTitle(f"{book_title} - {self.app_name} ({self.author_name})")
        else:
            self.setWindowTitle(f"{self.app_name} ({self.author_name})")

    def open_file_dialog(self):
        file_path, _ = QFileDialog.getOpenFileName(self, "یک فایل EPUB انتخاب کنید", "", "EPUB Files (*.epub)")
        self.load_book(file_path)

    def update_library(self, file_path, book_title):
        if any(b['path'] == file_path for b in self.library): return
        page_count = len(self.book.toc)
        self.library.append({'path': file_path, 'title': book_title, 'pages': page_count})
        self.refresh_library_list()

    def refresh_library_list(self):
        self.library_list.clear()
        for book in self.library:
            item_text = f"📖 {book['title']}\n📄 فصول: {book['pages']}"
            list_item = QListWidgetItem(item_text)
            list_item.setData(Qt.UserRole, book['path'])
            self.library_list.addItem(list_item)
    
    def load_book_from_library(self, item):
        file_path = item.data(Qt.UserRole)
        if file_path and file_path != self.current_book_path:
            self.load_book(file_path)


if __name__ == "__main__":
    app = QApplication(sys.argv)
    reader = EpubReader()
    reader.show()
    sys.exit(app.exec())
