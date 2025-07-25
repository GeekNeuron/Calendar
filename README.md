import sys
import os
import re
import sqlite3
from PySide6.QtWidgets import (
    QApplication, QMainWindow, QFileDialog, QTextBrowser,
    QListWidget, QListWidgetItem, QProgressBar, QWidget, QVBoxLayout,
    QHBoxLayout, QSplitter, QTabWidget
)
from PySide6.QtGui import QAction, QKeySequence, QFontDatabase, QFont, QIcon, QPainter, QPen, QColor
from PySide6.QtCore import Qt, QThread, QObject, Signal, QRect

import ebooklib
from ebooklib import epub
from bs4 import BeautifulSoup

# --- Worker class to process the book in the background ---
class BookLoaderWorker(QObject):
    finished = Signal(dict)

    def __init__(self, file_path):
        super().__init__()
        self.file_path = file_path

    def run(self):
        try:
            book = epub.read_epub(self.file_path)
            
            book_title_meta = book.get_metadata('DC', 'title')
            title = book_title_meta[0][0] if book_title_meta else os.path.basename(self.file_path)
            chapters = [{'title': item.title, 'href': item.href} for item in book.toc if isinstance(item, epub.Link)]

            total_len = 0
            chap_lens = []
            cum_lens = [0]
            spine_items = [book.get_item_with_id(item_id) for item_id, _ in book.spine]
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
            
            result = {
                'book': book, 'title': title, 'chapters': chapters, 'total_len': total_len, 
                'chap_lens': chap_lens, 'cum_lens': cum_lens
            }
            self.finished.emit(result)
        except Exception as e:
            self.finished.emit({'error': str(e)})

# --- Clickable progress bar with reactive text color ---
class ClickableProgressBar(QProgressBar):
    jump_requested = Signal(float)

    def __init__(self, parent=None):
        super().__init__(parent)
        self.setCursor(Qt.PointingHandCursor)
        self.setTextVisible(False)

    def paintEvent(self, event):
        super().paintEvent(event)
        painter = QPainter(self)
        
        progress_ratio = self.value() / self.maximum() if self.maximum() != 0 else 0
        chunk_width = int(self.width() * progress_ratio)
        
        text = f"{self.value()}%"
        font = self.font()
        font.setPointSize(9)
        font.setBold(True)
        painter.setFont(font)
        
        pen_white = QPen(QColor("white"))
        painter.setPen(pen_white)
        clip_white = QRect(0, 0, chunk_width, self.height())
        painter.save()
        painter.setClipRect(clip_white)
        painter.drawText(self.rect(), Qt.AlignCenter, text)
        painter.restore()
        
        pen_blue = QPen(QColor("#345B9A"))
        painter.setPen(pen_blue)
        clip_blue = QRect(chunk_width, 0, self.width() - chunk_width, self.height())
        painter.save()
        painter.setClipRect(clip_blue)
        painter.drawText(self.rect(), Qt.AlignCenter, text)
        painter.restore()

    def mousePressEvent(self, event):
        if event.button() == Qt.LeftButton: self.process_jump(event.pos())

    def mouseMoveEvent(self, event):
        if event.buttons() & Qt.LeftButton: self.process_jump(event.pos())
            
    def process_jump(self, pos):
        percentage = (pos.x() / self.width()) * 100
        self.setValue(int(percentage))
        self.jump_requested.emit(percentage)


# --- Main Application Class ---
class EpubReader(QMainWindow):
    def __init__(self):
        super().__init__()
        self.book = None
        self.chapters = []
        self.library = [] # This will be a list of dicts
        self.current_book_path = None
        self.app_name = "ePub Swift"
        self.author_name = "GeekNeuron"
        
        self.total_book_len = 0
        self.chapter_lens = []
        self.cumulative_lens = []

        self.base_path = os.path.dirname(__file__)
        self.db_path = os.path.join(self.base_path, "epub_swift.db")
        
        self.setup_database()
        self.load_assets()
        self.update_window_title()
        self.setWindowState(Qt.WindowMaximized)
        
        self.init_ui()
        self.apply_styles()
        self.worker_thread = None
        self.load_library_from_db()
        
    def setup_database(self):
        """Creates the database and tables if they don't exist."""
        con = sqlite3.connect(self.db_path)
        cur = con.cursor()
        cur.execute("""
            CREATE TABLE IF NOT EXISTS books (
                path TEXT PRIMARY KEY,
                title TEXT NOT NULL,
                chapter_count INTEGER,
                last_read_pos INTEGER DEFAULT 0
            )
        """)
        con.commit()
        con.close()
    
    def load_library_from_db(self):
        """Loads the book library from the SQLite database on startup."""
        con = sqlite3.connect(self.db_path)
        cur = con.cursor()
        cur.execute("SELECT path, title, chapter_count, last_read_pos FROM books")
        for row in cur.fetchall():
            self.library.append({
                'path': row[0], 'title': row[1], 'pages': row[2], 'last_read_pos': row[3]
            })
        con.close()
        self.refresh_library_list()

    def add_or_update_book_in_db(self, book_data):
        """Adds a new book or updates an existing one in the database."""
        con = sqlite3.connect(self.db_path)
        cur = con.cursor()
        cur.execute("INSERT OR REPLACE INTO books (path, title, chapter_count) VALUES (?, ?, ?)",
                    (book_data['path'], book_data['title'], book_data['pages']))
        con.commit()
        con.close()
        
    def save_current_progress_to_db(self):
        """Calculates and saves the current reading position to the database."""
        if not self.current_book_path or self.total_book_len == 0:
            return

        current_char_pos = self.get_current_char_position()
        
        con = sqlite3.connect(self.db_path)
        cur = con.cursor()
        cur.execute("UPDATE books SET last_read_pos = ? WHERE path = ?", (current_char_pos, self.current_book_path))
        con.commit()
        con.close()

    def get_current_char_position(self):
        """Helper function to get the absolute character position in the book."""
        if not self.book or self.total_book_len == 0 or self.toc_list.currentRow() < 0:
            return 0
            
        current_chapter_index = self.toc_list.currentRow()
        if not (0 <= current_chapter_index < len(self.cumulative_lens) - 1): return 0
        
        scrollbar = self.text_display.verticalScrollBar()
        max_val = scrollbar.maximum()
        scroll_progress = (scrollbar.value() / max_val) if max_val > 0 else 0
        
        preceding_len = self.cumulative_lens[current_chapter_index]
        current_chapter_len = self.chapter_lens[current_chapter_index]
        
        return int(preceding_len + (scroll_progress * current_chapter_len))

    def closeEvent(self, event):
        """Saves progress when the application is about to close."""
        self.save_current_progress_to_db()
        event.accept()

    def init_ui(self):
        menu_bar = self.menuBar()
        file_menu = menu_bar.addMenu("File")
        open_action = QAction("Load EPUB (Ctrl+O)", self)
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
        self.left_panel.addTab(self.toc_list, "Contents")
        self.left_panel.addTab(self.library_list, "Library")
        right_panel_widget = QWidget()
        right_layout = QVBoxLayout(right_panel_widget)
        right_layout.setContentsMargins(0, 0, 0, 0)
        right_layout.setSpacing(8)
        self.text_display = QTextBrowser()
        self.text_display.setOpenExternalLinks(True)
        self.text_display.verticalScrollBar().valueChanged.connect(self.update_global_progress)
        self.progress_bar = ClickableProgressBar()
        self.progress_bar.jump_requested.connect(self.jump_to_position)
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
                border: none; border-radius: 4px; background-color: #e0e0e0; max-height: 14px;
            }
            QProgressBar::chunk { background-color: #345B9A; border-radius: 4px; }
            QMenu { background-color: #ffffff; border: 1px solid #dcdde1; border-radius: 8px; padding: 5px; }
            QMenu::item { padding: 8px 25px 8px 20px; border-radius: 6px; }
            QMenu::item:selected { background-color: #345B9A; color: white; }
            QMenuBar { background-color: #f2f3f7; border-bottom: 1px solid #dcdde1; padding: 2px; }
            QMenuBar::item { background-color: transparent; padding: 6px 12px; border-radius: 6px; }
            QMenuBar::item:selected { background-color: #dcdde1; }
            QMainWindow, QWidget { background-color: #f2f3f7; }
            QTabWidget::pane { border: none; }
            QTabWidget::tab-bar { alignment: center; }
            QTabBar::tab { background: #e1e5ea; color: #555; padding: 8px 20px; border-top-left-radius: 6px; border-top-right-radius: 6px; margin: 0 2px; }
            QTabBar::tab:selected { background: #ffffff; color: #000; }
            QListWidget { background-color: #ffffff; color: #2c3e50; border: none; border-radius: 8px; padding: 5px; }
            QListWidget::item { padding: 8px; border-radius: 4px; }
            QListWidget::item:alternate { background-color: #f8f9fa; }
            QListWidget::item:selected { background-color: #345B9A; color: white; }
            QTextBrowser { background-color: #ffffff; border: none; border-radius: 8px; padding: 20px; font-size: 16px; color: #34495e; }
            QScrollBar:vertical { border: none; background: #f8f9fa; width: 10px; margin: 0; border-radius: 5px; }
            QScrollBar::handle:vertical { background: #bdc3c7; min-height: 20px; border-radius: 5px; }
            QScrollBar::handle:vertical:hover { background: #95a5a6; }
        """)

    def load_book(self, file_path):
        if not file_path: return
        
        # Corrected logic to allow loading new books
        if self.worker_thread and self.worker_thread.isRunning():
            # Optionally, you could warn the user that a book is already loading
            return 
            
        QApplication.setOverrideCursor(Qt.WaitCursor)
        # Save progress of the PREVIOUS book before loading a new one
        self.save_current_progress_to_db()
        
        self.toc_list.clear()
        self.text_display.clear()
        self.progress_bar.setValue(0)
        self.current_book_path = file_path
        
        self.worker_thread = QThread()
        self.worker = BookLoaderWorker(file_path)
        self.worker.moveToThread(self.worker_thread)
        self.worker_thread.started.connect(self.worker.run)
        self.worker.finished.connect(self.on_book_data_loaded)
        self.worker.finished.connect(self.worker_thread.quit)
        self.worker.finished.connect(self.worker.deleteLater)
        self.worker_thread.finished.connect(self.worker_thread.deleteLater)
        self.worker_thread.start()

    def on_book_data_loaded(self, result):
        QApplication.restoreOverrideCursor()
        if 'error' in result:
            self.text_display.setHtml(f"<h1>Error Opening Book</h1><p>{result['error']}</p>")
            return
        
        self.book = result['book']
        self.chapters = result['chapters']
        self.total_book_len = result['total_len']
        self.chapter_lens = result['chap_lens']
        self.cumulative_lens = result['cum_lens']

        self.update_window_title(result['title'])
        self.update_library(self.current_book_path, result['title'])
        
        toc_is_rtl = self.is_rtl(self.chapters[0]['title'] if self.chapters else "")
        self.toc_list.setLayoutDirection(Qt.RightToLeft if toc_is_rtl else Qt.LeftToRight)

        self.toc_list.addItems([f"{i+1}. {c['title']}" for i, c in enumerate(self.chapters)])

        if self.toc_list.count() > 0:
            self.toc_list.setCurrentRow(0)
            self.left_panel.setCurrentWidget(self.toc_list)
            
            # Restore last reading position
            book_in_lib = next((b for b in self.library if b['path'] == self.current_book_path), None)
            if book_in_lib and book_in_lib['last_read_pos'] > 0:
                percentage = (book_in_lib['last_read_pos'] / self.total_book_len) * 100
                self.jump_to_position(percentage)


    def display_chapter(self, current_item):
        if not current_item or not self.book: return
        self.update_global_progress()
        
        selected_index = self.toc_list.row(current_item)
        if not (0 <= selected_index < len(self.chapters)): return
        
        chapter_info = self.chapters[selected_index]
        item = self.book.get_item_with_href(chapter_info['href'])
        content_bytes = item.get_content()
        soup = BeautifulSoup(content_bytes, 'html.parser')
        
        body_tag = soup.find('body')
        if body_tag:
            direction = "rtl" if self.is_rtl(body_tag.get_text()) else "ltr"
            body_tag['dir'] = direction
        
        # Removed the <hr> injection
        style_tag = soup.new_tag('style')
        style_tag.string = "body { font-family: 'Vazirmatn', sans-serif !important; }"
        head = soup.find('head') or soup.new_tag('head')
        if not head.parent: soup.insert(0, head)
        head.append(style_tag)
        
        self.text_display.setHtml(soup.prettify())
        self.text_display.verticalScrollBar().setValue(0)

    def update_global_progress(self):
        current_char_pos = self.get_current_char_position()
        if current_char_pos == 0 and self.total_book_len == 0:
            global_percentage = 0
        else:
            global_percentage = (current_char_pos / self.total_book_len) * 100
        self.progress_bar.setValue(int(global_percentage))

    def jump_to_position(self, percentage):
        if not self.book or self.total_book_len == 0: return

        target_char_pos = self.total_book_len * (percentage / 100)
        
        target_chapter_index = -1
        for i in range(len(self.cumulative_lens) - 1):
            if self.cumulative_lens[i] <= target_char_pos < self.cumulative_lens[i+1]:
                target_chapter_index = i
                break
        
        if target_chapter_index == -1 and target_char_pos >= self.cumulative_lens[-1]:
             target_chapter_index = len(self.cumulative_lens) - 2

        if target_chapter_index < 0: return

        if self.toc_list.currentRow() != target_chapter_index:
            self.toc_list.blockSignals(True)
            self.toc_list.setCurrentRow(target_chapter_index)
            self.toc_list.blockSignals(False)
            self.display_chapter(self.toc_list.currentItem())
        
        # This needs a small delay to allow the text browser to render the new chapter
        # before we can accurately set the scroll position. A direct call might fail.
        # However, for simplicity, we try a direct call first.
        preceding_len = self.cumulative_lens[target_chapter_index]
        current_chapter_len = self.chapter_lens[target_chapter_index]
        
        if current_chapter_len > 0:
            progress_in_chapter = (target_char_pos - preceding_len) / current_chapter_len
            scrollbar = self.text_display.verticalScrollBar()
            new_scroll_value = int(progress_in_chapter * scrollbar.maximum())
            scrollbar.setValue(new_scroll_value)
    
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
        file_path, _ = QFileDialog.getOpenFileName(self, "Select an EPUB File", "", "EPUB Files (*.epub)")
        self.load_book(file_path)

    def update_library(self, file_path, book_title):
        # Check if book is already in the self.library list
        if any(b['path'] == file_path for b in self.library):
            return

        new_book_data = {'path': file_path, 'title': book_title, 'pages': len(self.chapters), 'last_read_pos': 0}
        self.library.append(new_book_data)
        self.add_or_update_book_in_db(new_book_data) # Save to DB
        self.refresh_library_list()

    def refresh_library_list(self):
        self.library_list.clear()
        for book in self.library:
            item_text = f"📖 {book['title']}\n📄 Chapters: {book['pages']}"
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
