
pyinstaller --onefile --windowed --name "Network Reset Tool" --icon="icon.ico" network_reset_tool.py


Traceback (most recent call last):
  File "Network_reset_tool.py", line 156, in <module>
  File "Network_reset_tool.py", line 63, in __init__
  File "customtkinter\windows\widgets\core_widget_classes\ctk_base_class.py", line 321, in grid
  File "tkinter\__init__.py", line 2693, in grid_configure
_tkinter.TclError: cannot use geometry manager grid inside .!ctkframe.!ctkframe which already has slaves managed by pack
