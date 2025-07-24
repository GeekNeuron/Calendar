
pyinstaller --onefile --windowed --name "Network Reset Tool" --icon="icon.ico" network_reset_tool.py


Traceback (most recent call last):
  File "Network_reset_tool.py", line 193, in <module>
  File "Network_reset_tool.py", line 41, in __init__
  File "customtkinter\windows\ctk_tk.py", line 208, in configure
  File "customtkinter\windows\widgets\appearance_mode\appearance_mode_base_class.py", line 55, in _check_color_type
ValueError: transparency is not allowed for this attribute
