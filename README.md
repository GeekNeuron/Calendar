import sys
import os
import re
from PySide6.QtWidgets import (
    QApplication, QMainWindow, QFileDialog, QTextBrowser,
    QListWidget, QListWidgetItem, QProgressBar, QWidget, QVBoxLayout,
    QHBoxLayout, QSplitter, QTabWidget
)
from PySide6.QtGui import QAction, QKeySequence, QFontDatabase, QFont, QIcon, QPainter, QPen
from PySide6.QtCore import Qt, QThread, QObject, Signal

import ebooklib
from ebooklib import epub
from bs4 import BeautifulSoup

# --- کلاس Worker برای پردازش کتاب در پس‌زمینه ---
class BookLoaderWorker(QObject):
    finished = Signal(dict)

    def __init__(self, book):
        super().__init__()
        self.book = book

    def run(self):
        """محاسبه طول کل کتاب و هر فصل برای نوار پیشرفت جامع."""
        total_len = 0
        chap_lens = []
        cum_lens = [0]
        ticks = []

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
                if total_len > 0:
                    # حذف خط اول خط‌کش (تیک صفر درصد)
                    ticks.append((cumulative / total_len) * 100)
            
            result = {
                'total_len': total_len, 'chap_lens': chap_lens, 
                'cum_lens': cum_lens, 'ticks': ticks
            }
            self.finished.emit(result)
        except Exception as e:
            self.finished.emit({'error': str(e)})


# --- کلاس نوار پیشرفت قابل کلیک ---
class RulerProgressBar(QProgressBar):
    jump_requested = Signal(float)

    def __init__(self, parent=None):
        super().__init__(parent)
        self._ticks = []
        self.setCursor(Qt.PointingHandCursor)

    def set_ticks(self, tick_percentages):
        self._ticks = tick_percentages
        self.update()

    def paintEvent(self, event):
        super().paintEvent(event)
        painter = QPainter(self)
        pen = QPen(Qt.darkGray, 1, Qt.DashLine)
        painter.setPen(pen)
        width, height = self.width(), self.height()
        for percent in self._ticks:
            x = int(width * (percent / 100.0))
            painter.drawLine(x, 0, x, height)
    
    def mousePressEvent(self, event):
        if event.button() == Qt.LeftButton:
            self.process_jump(event.pos())

    def mouseMoveEvent(self, event):
        if event.buttons() & Qt.LeftButton:
            self.process_jump(event.pos())
            
    def process_jump(self, pos):
        percentage = (pos.x() / self.width()) * 100
        self.jump_requested.emit(percentage)


# --- کلاس اصلی برنامه ---
class EpubReader(QMainWindow):
    def __init__(self):
        super().__init__()
        # ... (متغیرها مشابه قبل) ...
        self.book = None
        self.chapters = []
        self.library = []
        self.current_book_path = None
        self.app_name = "ePub Swift"
        self.author_name = "GeekNeuron"
        
        self.total_book_len = 0
        self.chapter_lens = []
        self.cumulative_lens = []
        self.spine_ids = []

        self.base_path = os.path.dirname(__file__)
        self.load_assets()
        self.update_window_title()
        self.setWindowState(Qt.WindowMaximized)
        
        self.init_ui()
        self.apply_styles()
        self.worker_thread = None

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
        
        # ... (ساختار UI مشابه قبل) ...
        self.progress_bar = RulerProgressBar() # استفاده از کلاس جدید
        self.progress_bar.jump_requested.connect(self.jump_to_position) # اتصال سیگنال جدید
        # ... بقیه کد init_ui که مشابه قبل است ...
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
            QMenu { 
                background-color: #ffffff; 
                border: 1px solid #dcdde1;
                border-radius: 8px; /* دورگرد کردن کامل منوی بازشونده */
                padding: 5px;
            }
            QMenu::item {
                padding: 8px 25px 8px 20px;
                border-radius: 6px;
            }
            QMenu::separator { height: 1px; background: #e0e0e0; margin: 5px 0; }
            QMenu::item:selected { background-color: #345B9A; color: white; }
            
            /* ... (بقیه استایل‌ها مشابه قبل) ... */
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
        if not file_path: return
        QApplication.setOverrideCursor(Qt.WaitCursor)
        self.toc_list.clear()
        self.text_display.clear()
        self.progress_bar.setValue(0)
        self.progress_bar.set_ticks([])

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

            # ... (بقیه کد load_book برای پر کردن UI که سریع است) ...
            book_title_meta = self.book.get_metadata('DC', 'title')
            book_title = book_title_meta[0][0] if book_title_meta else os.path.basename(file_path)
            self.update_window_title(book_title)
            self.update_library(file_path, book_title)

            self.chapters = []
            self.spine_ids = [item_id for item_id, _ in self.book.spine]
            
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
        """اسلات برای دریافت نتایج از ترد پردازشگر."""
        if 'error' in result:
            print(f"Error in worker thread: {result['error']}")
        else:
            self.total_book_len = result['total_len']
            self.chapter_lens = result['chap_lens']
            self.cumulative_lens = result['cum_lens']
            self.progress_bar.set_ticks(result['ticks'])
            self.update_global_progress()
        QApplication.restoreOverrideCursor()

    def display_chapter(self, current_item):
        # ... (کد display_chapter مشابه قبل با یک تغییر مهم) ...
        style_tag = soup.new_tag('style')
        style_tag.string = f"""
            /* اصلاح خط‌کشی برای اعمال به تگ‌های بیشتر */
            p, div, blockquote {{
                border-bottom: 1px solid #e0e6f0;
                padding-bottom: 8px; 
                margin-bottom: 8px; /* ایجاد فاصله بیشتر */
            }}
            /* ... بقیه استایل فونت و جهت‌گیری ... */
        """
        # ... بقیه کد display_chapter که مشابه قبل است ...
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
        style_tag = soup.new_tag('style')
        style_tag.string = f"""
            p, div, blockquote {{
                border-bottom: 1px solid #e0e6f0;
                padding-bottom: 8px;
                margin-bottom: 8px;
            }}
            body, p, div, span, li, a, h1, h2, h3, h4 {{ 
                font-family: 'Vazirmatn', sans-serif !important; 
                direction: {direction};
                text-align: {"right" if direction == "rtl" else "left"};
            }}
        """
        head = soup.find('head') or soup.new_tag('head')
        if not head.parent: soup.insert(0, head)
        head.append(style_tag)
        self.text_display.setHtml(soup.prettify())
        self.text_display.verticalScrollBar().setValue(0)

    def jump_to_position(self, percentage):
        """پرش به بخش مورد نظر از کتاب بر اساس درصد کلیک شده."""
        if not self.book or self.total_book_len == 0: return

        target_char_pos = self.total_book_len * (percentage / 100)
        
        # پیدا کردن فصل مقصد
        target_chapter_index = -1
        for i, cum_len in enumerate(self.cumulative_lens):
            if target_char_pos < cum_len:
                target_chapter_index = i - 1
                break
        
        if target_chapter_index == -1: return

        # جلوگیری از فراخوانی‌های تودرتو و اضافی
        if self.toc_list.currentRow() != target_chapter_index:
            self.toc_list.blockSignals(True)
            self.toc_list.setCurrentRow(target_chapter_index)
            self.toc_list.blockSignals(False)
            self.display_chapter(self.toc_list.currentItem())
        
        # محاسبه و تنظیم اسکرول در فصل مقصد
        preceding_len = self.cumulative_lens[target_chapter_index]
        current_chapter_len = self.chapter_lens[target_chapter_index]
        
        if current_chapter_len > 0:
            progress_in_chapter = (target_char_pos - preceding_len) / current_chapter_len
            scrollbar = self.text_display.verticalScrollBar()
            new_scroll_value = int(progress_in_chapter * scrollbar.maximum())
            scrollbar.setValue(new_scroll_value)
    
    # ... سایر متدها بدون تغییر باقی می‌مانند ...
    # (load_assets, is_rtl, update_window_title, open_file_dialog, update_library, etc.)
EpubReader.load_assets = load_assets_full
EpubReader.is_rtl = is_rtl_full
EpubReader.update_window_title = update_window_title_full
EpubReader.open_file_dialog = open_file_dialog_full
EpubReader.update_library = update_library_full
EpubReader.refresh_library_list = refresh_library_list_full
EpubReader.load_book_from_library = load_book_from_library_full
EpubReader.update_global_progress = update_global_progress_full # Assuming this is defined elsewhere
# Need to define update_global_progress_full
def update_global_progress_full(self):
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
EpubReader.update_global_progress = update_global_progress_full


if __name__ == "__main__":
    app = QApplication(sys.argv)
    reader = EpubReader()
    reader.show()
    sys.exit(app.exec())
