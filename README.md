
import sys
import os
import re
from PySide6.QtWidgets import (
    QApplication, QMainWindow, QFileDialog, QTextBrowser,
    QListWidget, QListWidgetItem, QProgressBar, QWidget, QVBoxLayout,
    QHBoxLayout, QSplitter, QTabWidget
)
from PySide6.QtGui import QAction, QKeySequence, QFontDatabase, QFont, QIcon
from PySide6.QtCore import Qt

import ebooklib
from ebooklib import epub
from bs4 import BeautifulSoup

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

        # --- بارگذاری منابع و تنظیمات اولیه ---
        self.base_path = os.path.dirname(__file__)
        self.load_assets()
        self.update_window_title() # تنظیم عنوان پیش‌فرض
        
        # --- راه‌اندازی رابط کاربری ---
        self.init_ui()
        self.apply_styles()

        # برنامه به صورت Maximize شروع به کار می‌کند اما کاربر کنترل کامل دارد
        self.setWindowState(Qt.WindowMaximized)

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
        self.left_panel.setMaximumWidth(450)

        # زبانه فصول
        self.toc_list = QListWidget()
        self.toc_list.setAlternatingRowColors(True)
        self.toc_list.currentItemChanged.connect(self.display_chapter)
        self.left_panel.addTab(self.toc_list, "فهرست کتاب")

        # زبانه کتابخانه
        self.library_list = QListWidget()
        self.library_list.setAlternatingRowColors(True) # دو رنگ کردن لیست کتابخانه
        self.library_list.itemClicked.connect(self.load_book_from_library)
        self.left_panel.addTab(self.library_list, "کتابخانه")
        
        right_panel_widget = QWidget()
        right_layout = QVBoxLayout(right_panel_widget)
        right_layout.setContentsMargins(0, 0, 0, 0)
        right_layout.setSpacing(8)

        self.text_display = QTextBrowser()
        self.text_display.setOpenExternalLinks(True)
        self.text_display.verticalScrollBar().valueChanged.connect(self.update_line_progress)

        self.progress_bar = QProgressBar()
        self.progress_bar.setTextVisible(False)

        right_layout.addWidget(self.text_display)
        right_layout.addWidget(self.progress_bar)

        splitter = QSplitter(Qt.Horizontal)
        splitter.addWidget(self.left_panel)
        splitter.addWidget(right_panel_widget)
        
        # تنظیم اندازه اولیه پنل‌ها به صورت بهینه
        splitter.setStretchFactor(0, 0) # پنل چپ در حداقل اندازه خود باقی می‌ماند
        splitter.setStretchFactor(1, 1) # پنل راست بقیه فضا را اشغال می‌کند
        splitter.setHandleWidth(2)

        main_layout.addWidget(splitter)

    def apply_styles(self):
        """اعمال استایل شیت (QSS) برای ظاهر مدرن و چشم‌نواز"""
        # استایل‌ها مشابه قبل باقی می‌مانند، چون لیست کتابخانه هم از استایل عمومی QListWidget ارث‌بری می‌کند
        self.setStyleSheet("""
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
            /* استایل خاص برای آیتم‌های کتابخانه جهت خوانایی بهتر */
            #LibraryList QListWidget::item { padding: 12px 8px; }
            QTextBrowser {
                background-color: #ffffff; border: none; border-radius: 8px;
                padding: 20px; font-size: 16px; color: #34495e;
            }
            QProgressBar {
                border: none; border-radius: 4px; background-color: #e0e0e0; max-height: 8px;
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
        """)
        # یک نام شیء برای استایل‌دهی خاص به لیست کتابخانه
        self.library_list.setObjectName("LibraryList")

    def is_rtl(self, text, threshold=0.4):
        """تشخیص خودکار زبان راست‌چین (با رفع باگ تقسیم بر صفر)"""
        if not text: return False
        clean_text = re.sub('<[^<]+?>', '', text)
        if not clean_text: return False
        
        sample = clean_text[:500]
        # **رفع باگ**: جلوگیری از خطای تقسیم بر صفر اگر متن نمونه خالی باشد
        if not sample: return False
        
        rtl_chars = len(re.findall(r'[\u0600-\u06FF]', sample))
        return (rtl_chars / len(sample)) > threshold

    def update_window_title(self, book_title=None):
        """ به‌روزرسانی عنوان پنجره برنامه """
        if book_title:
            self.setWindowTitle(f"{book_title} - {self.app_name} ({self.author_name})")
        else:
            self.setWindowTitle(f"{self.app_name} ({self.author_name})")

    def open_file_dialog(self):
        file_path, _ = QFileDialog.getOpenFileName(self, "یک فایل EPUB انتخاب کنید", "", "EPUB Files (*.epub)")
        if file_path:
            self.load_book(file_path)

    def load_book(self, file_path):
        try:
            self.book = epub.read_epub(file_path)
            self.current_book_path = file_path
            
            # استخراج نام کتاب و به‌روزرسانی عنوان پنجره
            book_title_meta = self.book.get_metadata('DC', 'title')
            book_title = book_title_meta[0][0] if book_title_meta else os.path.basename(file_path)
            self.update_window_title(book_title)
            
            self.update_library(file_path, book_title)

            self.toc_list.clear()
            self.chapters = []
            
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
        
        body_text = soup.get_text()
        content_is_rtl = self.is_rtl(body_text)
        direction = "rtl" if content_is_rtl else "ltr"
        
        style_tag = soup.new_tag('style')
        style_tag.string = f"""
            body, p, div, span, li, a, h1, h2, h3, h4 {{ 
                font-family: 'Vazirmatn', 'Segoe UI', sans-serif !important; 
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

    def update_line_progress(self, value):
        scrollbar = self.text_display.verticalScrollBar()
        max_val = scrollbar.maximum()
        if max_val > 0:
            progress = int((value / max_val) * 100)
            self.progress_bar.setValue(progress)

    def update_library(self, file_path, book_title):
        if any(b['path'] == file_path for b in self.library): return

        page_count = len(self.book.toc)
        self.library.append({
            'path': file_path, 'title': book_title, 'pages': page_count, 'progress': 0
        })
        self.refresh_library_list()

    def refresh_library_list(self):
        self.library_list.clear()
        for book in self.library:
            item_text = f"📖 {book['title']}\n" \
                        f"📄 فصول: {book['pages']}"
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
