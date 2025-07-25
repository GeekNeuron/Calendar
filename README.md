import sys
import os
from PySide6.QtWidgets import (
    QApplication, QMainWindow, QFileDialog, QTextBrowser,
    QListWidget, QProgressBar, QWidget, QVBoxLayout,
    QSplitter
)
from PySide6.QtGui import QAction, QKeySequence, QFontDatabase, QFont, QIcon
from PySide6.QtCore import Qt
import ebooklib
from ebooklib import epub
from bs4 import BeautifulSoup

class EpubReader(QMainWindow):
    """
    کلاس اصلی نرم‌افزار کتاب‌خوان EPUB با ظاهر مدرن
    """
    def __init__(self):
        super().__init__()

        # --- متغیرهای مربوط به کتاب ---
        self.book = None
        self.chapters = []
        self.spine = []
        
        # --- بارگذاری فونت و آیکون ---
        self.base_path = os.path.dirname(__file__)
        self.load_assets()

        # --- تنظیمات اولیه پنجره ---
        self.setWindowTitle("کتاب‌خوان EPUB")
        # اجرای برنامه به صورت ماکسیمایز (تمام صفحه نرمال)
        self.showMaximized()
        
        # --- راه‌اندازی رابط کاربری ---
        self.init_ui()
        
        # --- اعمال استایل مدرن ---
        self.apply_styles()

    def load_assets(self):
        """
        بارگذاری فونت سفارشی و آیکون برنامه از پوشه assets
        """
        # بارگذاری فونت وزیرمتن
        font_path = os.path.join(self.base_path, 'assets', 'fonts', 'Vazirmatn-Medium.ttf')
        if os.path.exists(font_path):
            font_id = QFontDatabase.addApplicationFont(font_path)
            if font_id != -1:
                font_families = QFontDatabase.applicationFontFamilies(font_id)
                app_font = QFont(font_families[0], 11)
                QApplication.instance().setFont(app_font)
        
        # تنظیم آیکون برنامه
        icon_path = os.path.join(self.base_path, 'assets', 'icons', 'app_icon.png')
        if os.path.exists(icon_path):
            self.setWindowIcon(QIcon(icon_path))


    def init_ui(self):
        """
        متد ساخت و چیدمان تمام اجزای رابط کاربری
        """
        # --- ساخت منو بار ---
        menu_bar = self.menuBar()
        file_menu = menu_bar.addMenu("File")
        open_action = QAction("بارگذاری کتاب (Ctrl+O)", self)
        open_action.setShortcut(QKeySequence("Ctrl+O"))
        open_action.triggered.connect(self.open_file_dialog)
        file_menu.addAction(open_action)
        menu_bar.addMenu("Edit")
        menu_bar.addMenu("Settings")
        
        # --- ساخت ویجت‌های اصلی ---
        
        # پنل چپ: لیست فصول کتاب
        self.toc_list = QListWidget()
        self.toc_list.setAlternatingRowColors(True) # فعال‌سازی رنگ‌بندی یکی در میان
        self.toc_list.currentItemChanged.connect(self.display_chapter)

        # پنل راست: نمایش محتوای کتاب
        self.text_display = QTextBrowser()
        self.text_display.setOpenExternalLinks(True)

        # پنل پایین: نوار پیشرفت
        self.progress_bar = QProgressBar()
        self.progress_bar.setTextVisible(True)
        self.progress_bar.setFormat("%p%")

        # --- چیدمان (Layout) ---
        central_widget = QWidget()
        self.setCentralWidget(central_widget)
        main_layout = QVBoxLayout(central_widget)
        main_layout.setContentsMargins(10, 10, 10, 10) # ایجاد فاصله از لبه‌ها
        main_layout.setSpacing(10)

        splitter = QSplitter(Qt.Horizontal)
        splitter.addWidget(self.toc_list)
        splitter.addWidget(self.text_display)
        splitter.setSizes([250, 750])
        splitter.setHandleWidth(2) # نازک کردن جداکننده

        main_layout.addWidget(splitter)
        main_layout.addWidget(self.progress_bar)

    def apply_styles(self):
        """اعمال استایل شیت (QSS) برای ظاهر مدرن و چشم‌نواز"""
        self.setStyleSheet("""
            QMainWindow, QWidget {
                background-color: #f2f3f7; /* رنگ پس‌زمینه اصلی کمی خاکستری */
            }
            QListWidget {
                background-color: #ffffff;
                color: #2c3e50;
                border: none;
                border-radius: 8px; /* گوشه‌های گرد */
                padding: 5px;
                font-size: 14px;
            }
            QListWidget::item {
                padding: 8px;
                border-radius: 4px;
            }
            QListWidget::item:alternate {
                background-color: #f8f9fa; /* رنگ ردیف‌های زوج */
            }
            QListWidget::item:selected {
                background-color: #345B9A; /* رنگ آبی تیره برای آیتم انتخاب شده */
                color: white;
            }
            QTextBrowser {
                background-color: #ffffff;
                border: none;
                border-radius: 8px;
                padding: 15px;
                font-size: 16px;
                color: #34495e;
            }
            QProgressBar {
                border: none;
                border-radius: 5px;
                background-color: #e0e0e0;
                max-height: 10px; /* نازک کردن نوار پیشرفت */
                text-align: center;
                color: transparent; /* مخفی کردن درصد پیش‌فرض */
            }
            QProgressBar::chunk {
                background-color: #345B9A; /* رنگ آبی تیره برای بخش پر شده */
                border-radius: 5px;
            }
            QMenuBar {
                background-color: #f2f3f7;
                border-bottom: 1px solid #dcdde1;
            }
            QMenu {
                background-color: #ffffff;
                border: 1px solid #dcdde1;
            }
            QMenu::item:selected {
                background-color: #345B9A;
                color: white;
            }
            /* استایل اسکرول بار مدرن */
            QScrollBar:vertical {
                border: none;
                background: #f8f9fa;
                width: 10px;
                margin: 0px 0px 0px 0px;
                border-radius: 5px;
            }
            QScrollBar::handle:vertical {
                background: #bdc3c7;
                min-height: 20px;
                border-radius: 5px;
            }
            QScrollBar::handle:vertical:hover {
                background: #95a5a6;
            }
            QScrollBar::add-line:vertical, QScrollBar::sub-line:vertical {
                border: none;
                background: none;
                height: 0px;
            }
        """)

    def open_file_dialog(self):
        file_path, _ = QFileDialog.getOpenFileName(self, "یک فایل EPUB انتخاب کنید", "", "EPUB Files (*.epub)")
        if file_path:
            self.load_book(file_path)

    def load_book(self, file_path):
        try:
            self.book = epub.read_epub(file_path)
            self.toc_list.clear()
            self.text_display.clear()
            self.chapters = []
            self.spine = list(self.book.spine)
            
            # افزودن شماره به ابتدای هر فصل
            chapter_counter = 1
            for item in self.book.toc:
                if isinstance(item, epub.Link):
                    self.chapters.append({'title': item.title, 'href': item.href})
                    self.toc_list.addItem(f"{chapter_counter}. {item.title}")
                    chapter_counter += 1
                elif isinstance(item, tuple):
                    link, _ = item
                    self.chapters.append({'title': link.title, 'href': link.href})
                    self.toc_list.addItem(f"{chapter_counter}. {link.title}")
                    chapter_counter += 1

            if self.toc_list.count() > 0:
                self.toc_list.setCurrentRow(0)
        except Exception as e:
            self.text_display.setText(f"خطا در باز کردن کتاب: \n{e}")
            self.progress_bar.setValue(0)

    def display_chapter(self, current_item):
        if not current_item or not self.book:
            return

        selected_index = self.toc_list.row(current_item)
        if not (0 <= selected_index < len(self.chapters)):
            return

        chapter_info = self.chapters[selected_index]
        item = self.book.get_item_with_href(chapter_info['href'])
        content_bytes = item.get_content()
        soup = BeautifulSoup(content_bytes, 'html.parser')
        
        # تزریق فونت وزیرمتن و استایل راست‌چین به محتوای کتاب
        font_style = soup.new_tag('style')
        font_style.string = """
            body, p, div, span, li, a, h1, h2, h3, h4 { 
                font-family: 'Vazirmatn', sans-serif !important; 
                direction: rtl; 
            }
        """
        
        head = soup.find('head')
        if not head:
            head = soup.new_tag('head')
            soup.insert(0, head)
        head.append(font_style)

        self.text_display.setHtml(soup.prettify())
        self.update_progress(item.id)
        
    def update_progress(self, current_item_id):
        try:
            total_items = len(self.spine)
            current_index = next((i for i, (item_id, _) in enumerate(self.spine) if item_id == current_item_id), -1)
            
            if total_items > 0 and current_index != -1:
                progress_percentage = int(((current_index + 1) / total_items) * 100)
                self.progress_bar.setValue(progress_percentage)
        except Exception:
            self.progress_bar.setValue(0)


if __name__ == "__main__":
    app = QApplication(sys.argv)
    reader = EpubReader()
    reader.show()
    sys.exit(app.exec())
