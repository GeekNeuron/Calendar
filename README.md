import sys
from PySide6.QtWidgets import (
    QApplication, QMainWindow, QFileDialog, QTextBrowser,
    QListWidget, QProgressBar, QWidget, QVBoxLayout,
    QHBoxLayout, QSplitter
)
from PySide6.QtGui import QAction, QKeySequence
from PySide6.QtCore import Qt
import ebooklib
from ebooklib import epub
from bs4 import BeautifulSoup

class EpubReader(QMainWindow):
    """
    کلاس اصلی نرم‌افزار کتاب‌خوان EPUB
    """
    def __init__(self):
        super().__init__()

        # --- متغیرهای مربوط به کتاب ---
        self.book = None
        self.chapters = []  # لیستی برای نگهداری اطلاعات فصول (عنوان، لینک)
        self.spine = []     # لیستی برای نگهداری ترتیب خواندن فصول

        # --- تنظیمات اولیه پنجره ---
        self.setWindowTitle("کتاب‌خوان EPUB")
        self.setGeometry(100, 100, 1200, 800)
        
        # اجرای برنامه به صورت تمام صفحه
        self.showFullScreen()
        
        # --- راه‌اندازی رابط کاربری ---
        self.init_ui()
        
        # --- اعمال تم روشن ---
        self.apply_light_theme()

    def init_ui(self):
        """
        متد ساخت و چیدمان تمام اجزای رابط کاربری
        """
        # --- ساخت منو بار (Menu Bar) ---
        menu_bar = self.menuBar()
        
        # منوی File
        file_menu = menu_bar.addMenu("File")
        open_action = QAction("بارگذاری کتاب (Load EPUB)...", self)
        open_action.setShortcut(QKeySequence("Ctrl+O"))
        open_action.triggered.connect(self.open_file_dialog)
        file_menu.addAction(open_action)

        # منوهای Edit و Settings (فعلا خالی)
        menu_bar.addMenu("Edit")
        menu_bar.addMenu("Settings")
        
        # --- ساخت ویجت‌های اصلی ---
        
        # پنل چپ: لیست فصول کتاب
        self.toc_list = QListWidget()
        self.toc_list.currentItemChanged.connect(self.display_chapter)

        # پنل راست: نمایش محتوای کتاب
        self.text_display = QTextBrowser()
        self.text_display.setOpenExternalLinks(True) # برای باز کردن لینک‌های خارجی در مرورگر

        # پنل پایین: نوار پیشرفت
        self.progress_bar = QProgressBar()
        self.progress_bar.setTextVisible(True)
        self.progress_bar.setFormat("%p%") # نمایش به صورت درصد

        # --- چیدمان (Layout) ---
        
        # ویجت مرکزی برای پنجره اصلی
        central_widget = QWidget()
        self.setCentralWidget(central_widget)

        # چیدمان عمودی اصلی (برای قرار دادن بخش کتاب و نوار پیشرفت)
        main_layout = QVBoxLayout(central_widget)

        # استفاده از QSplitter برای تقسیم‌بندی قابل تغییر بین لیست فصول و محتوا
        splitter = QSplitter(Qt.Horizontal)
        splitter.addWidget(self.toc_list)
        splitter.addWidget(self.text_display)
        
        # تنظیم اندازه اولیه پنل‌ها (پنل محتوا 3 برابر پنل فصول)
        splitter.setSizes([200, 600])

        # اضافه کردن اجزا به چیدمان اصلی
        main_layout.addWidget(splitter)
        main_layout.addWidget(self.progress_bar)

    def apply_light_theme(self):
        """اعمال استایل برای تم روشن"""
        self.setStyleSheet("""
            QMainWindow, QWidget {
                background-color: #f0f0f0;
                color: #000000;
            }
            QListWidget {
                background-color: #ffffff;
                border: 1px solid #cccccc;
                font-size: 14px;
            }
            QTextBrowser {
                background-color: #ffffff;
                border: 1px solid #cccccc;
                font-size: 16px;
            }
            QMenuBar {
                background-color: #e8e8e8;
            }
            QMenu {
                background-color: #f8f8f8;
            }
            QProgressBar {
                text-align: center;
            }
        """)

    def open_file_dialog(self):
        """باز کردن دیالوگ انتخاب فایل EPUB"""
        file_path, _ = QFileDialog.getOpenFileName(
            self,
            "یک فایل EPUB انتخاب کنید",
            "",
            "EPUB Files (*.epub)"
        )
        if file_path:
            self.load_book(file_path)

    def load_book(self, file_path):
        """بارگذاری و پردازش کتاب از مسیر داده شده"""
        try:
            self.book = epub.read_epub(file_path)
            
            # پاک کردن اطلاعات کتاب قبلی
            self.toc_list.clear()
            self.text_display.clear()
            self.chapters = []
            
            # استخراج ترتیب فصول (spine)
            self.spine = list(self.book.spine)

            # استخراج و نمایش فهرست مطالب (Table of Contents)
            for item in self.book.toc:
                if isinstance(item, epub.Link):
                    self.chapters.append({'title': item.title, 'href': item.href})
                    self.toc_list.addItem(item.title)
                elif isinstance(item, tuple): # برای فهرست‌های تو در تو
                    link, _ = item
                    self.chapters.append({'title': link.title, 'href': link.href})
                    self.toc_list.addItem(link.title)
            
            # نمایش فصل اول به صورت پیش‌فرض
            if self.toc_list.count() > 0:
                self.toc_list.setCurrentRow(0)

        except Exception as e:
            self.text_display.setText(f"خطا در باز کردن کتاب: \n{e}")
            self.progress_bar.setValue(0)

    def display_chapter(self, current_item):
        """نمایش محتوای فصل انتخاب شده"""
        if not current_item or not self.book:
            return

        # پیدا کردن اطلاعات فصل بر اساس آیتم انتخاب شده
        selected_index = self.toc_list.row(current_item)
        if selected_index < 0 or selected_index >= len(self.chapters):
            return
            
        chapter_info = self.chapters[selected_index]
        href = chapter_info['href']
        
        # دریافت محتوای فصل از کتاب
        item = self.book.get_item_with_href(href)
        content_bytes = item.get_content()
        
        # استفاده از BeautifulSoup برای پاک‌سازی و استخراج بدنه HTML
        soup = BeautifulSoup(content_bytes, 'html.parser')
        
        # حذف تگ‌های اسکریپت و استایل برای نمایش بهتر
        for tag in soup(['script', 'style']):
            tag.decompose()
            
        body = soup.find('body')
        if body:
            self.text_display.setHtml(body.prettify())
        else:
            # اگر تگ body نبود، کل محتوا را نمایش بده
            self.text_display.setHtml(soup.prettify())
            
        self.update_progress(item.id)
        
    def update_progress(self, current_item_id):
        """به‌روزرسانی نوار پیشرفت بر اساس فصل فعلی"""
        try:
            # پیدا کردن ایندکس فصل فعلی در لیست spine
            total_items = len(self.spine)
            current_index = 0
            for i, (item_id, _) in enumerate(self.spine):
                if item_id == current_item_id:
                    current_index = i
                    break
            
            if total_items > 0:
                progress_percentage = int(((current_index + 1) / total_items) * 100)
                self.progress_bar.setValue(progress_percentage)
        except Exception:
            self.progress_bar.setValue(0)


if __name__ == "__main__":
    app = QApplication(sys.argv)
    reader = EpubReader()
    reader.show()
    sys.exit(app.exec())
