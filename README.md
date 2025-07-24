
pyinstaller --onefile --windowed --name "Network Reset Tool" --icon="icon.ico" network_reset_tool.py

import customtkinter as ctk
import subprocess
import ctypes
import sys
import webbrowser
from tkinter import messagebox

# --- App Configuration ---
WINDOW_TITLE = "Network Reset Tool"
WINDOW_GEOMETRY = "500x640" # افزایش جزئی ارتفاع برای اطمینان از فضای کافی

# --- Colors for better visual feedback ---
SUCCESS_COLOR = "#2ECC71"
ERROR_COLOR = "#E74C3C"
PRIMARY_BUTTON_COLOR = "#3498DB"
RESTART_BUTTON_COLOR = "#C0392B"


class App(ctk.CTk):
    """The main application class for the Network Reset Tool."""
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)

        # --- Main Window Setup ---
        self.title(WINDOW_TITLE)
        self.geometry(WINDOW_GEOMETRY)
        self.resizable(False, False)
        ctk.set_appearance_mode("dark")

        # --- FIX 1: Footer is created and packed first to ensure visibility ---
        footer_frame = ctk.CTkFrame(self, fg_color="transparent")
        footer_frame.pack(side="bottom", fill="x", pady=(0, 10))
        footer_label = ctk.CTkLabel(footer_frame, text="Created with ❤️ by GeekNeuron", cursor="hand2", font=ctk.CTkFont(size=12))
        footer_label.pack()
        footer_label.configure(text_color="#85C1E9")
        footer_label.bind("<Button-1>", lambda e: self.open_link("https://github.com/GeekNeuron"))

        # --- Main Frame ---
        # The main frame now expands to fill the remaining space above the footer
        self.main_frame = ctk.CTkFrame(self)
        self.main_frame.pack(pady=10, padx=20, fill="both", expand=True)
        
        # --- FIX 3: Increased top padding for the logo ---
        self.logo_label = ctk.CTkLabel(self.main_frame, text="⚙️", font=ctk.CTkFont(size=40))
        self.logo_label.pack(pady=(15, 5)) 

        # --- Title and Description ---
        self.title_label = ctk.CTkLabel(self.main_frame, text="Network Reset Tool", font=ctk.CTkFont(size=22, weight="bold"))
        self.title_label.pack()
        
        # --- FIX 2: Reduced bottom padding for the description label ---
        self.description_label = ctk.CTkLabel(self.main_frame, text="A professional tool to fix your network connectivity issues.", font=ctk.CTkFont(size=13), text_color="gray60")
        self.description_label.pack(pady=(0, 10))

        # --- Primary Action Button ---
        self.reset_button = ctk.CTkButton(self.main_frame, text="Start Full Network Reset", command=self.confirm_and_run_reset, height=40, font=ctk.CTkFont(size=14, weight="bold"), fg_color=PRIMARY_BUTTON_COLOR, hover_color="#2980B9")
        self.reset_button.pack(pady=10, padx=10, fill="x")
        
        # --- Progress Bar ---
        self.progress_bar = ctk.CTkProgressBar(self.main_frame, orientation="horizontal")
        self.progress_bar.set(0)
        self.progress_bar.pack(pady=(5, 10), padx=10, fill="x")

        # --- Log Display Textbox ---
        self.log_textbox = ctk.CTkTextbox(self.main_frame, height=150, state="disabled", font=("Consolas", 11), border_width=1)
        self.log_textbox.pack(pady=10, padx=10, fill="both", expand=True)

        # --- System Shortcuts Frame ---
        self.shortcuts_frame = ctk.CTkFrame(self.main_frame)
        self.shortcuts_frame.pack(pady=10, padx=10, fill="x")
        self.shortcuts_frame.grid_columnconfigure((0, 1), weight=1)

        self.shortcuts_label = ctk.CTkLabel(self.shortcuts_frame, text="Troubleshooting Shortcuts", font=ctk.CTkFont(size=12, weight="bold"))
        self.shortcuts_label.grid(row=0, column=0, columnspan=2, padx=10, pady=(10, 10), sticky="ew")

        shortcuts = {"Network & Internet": "ms-settings:network-status", "Proxy Settings": "ms-settings:network-proxy", "Network Adapters": "control.exe ncpa.cpl", "Windows Firewall": "control.exe firewall.cpl"}
        for i, (text, command) in enumerate(shortcuts.items()):
            button = ctk.CTkButton(self.shortcuts_frame, text=text, command=lambda c=command: self.open_system_command(c), fg_color="gray50", hover_color="gray40")
            button.grid(row=(i // 2) + 1, column=i % 2, padx=5, pady=5, sticky="ew")
            
        # --- Smart Restart Button ---
        # --- FIX 3: Increased bottom padding for the restart button ---
        self.restart_button = ctk.CTkButton(self.main_frame, text="Restart Computer to Apply Changes", command=self.confirm_and_restart, state="disabled", fg_color=RESTART_BUTTON_COLOR, hover_color="#A93226")
        self.restart_button.pack(pady=(10, 15), padx=10, fill="x")

    def log(self, message, status="info"):
        """Appends a styled message to the log display."""
        icon_map = {"info": "[*]", "success": "[✅]", "error": "[❌]"}
        icon = icon_map.get(status, "[*]")
        self.log_textbox.configure(state="normal")
        self.log_textbox.insert("end", f"{icon} {message}\n")
        self.log_textbox.see("end")
        self.log_textbox.configure(state="disabled")
        self.update_idletasks()

    def run_command(self, command, log_message):
        """Executes a shell command and logs its progress and status."""
        self.log(f"{log_message}...", status="info")
        try:
            creation_flags = subprocess.CREATE_NO_WINDOW
            subprocess.run(command, shell=True, check=True, capture_output=True, creationflags=creation_flags)
            self.log(f"{log_message} completed successfully.", status="success")
            return True
        except subprocess.CalledProcessError as e:
            self.log(f"Error during '{log_message}': {e}", status="error")
            return False

    def confirm_and_run_reset(self):
        """Show a confirmation dialog before running the reset."""
        answer = messagebox.askyesno(
            title="Confirmation",
            message="Are you sure you want to reset all network settings?\nThis action cannot be undone and may temporarily disconnect you from the internet."
        )
        if answer:
            self.run_full_reset()

    def run_full_reset(self):
        """Executes the complete network reset sequence with visual feedback."""
        self.reset_button.configure(state="disabled", text="Resetting, please wait...")
        self.restart_button.configure(state="disabled")
        self.progress_bar.set(0)
        
        self.log_textbox.configure(state="normal")
        self.log_textbox.delete("1.0", "end")
        self.log_textbox.configure(state="disabled")

        self.log("--- Starting Network Reset Process ---", status="info")
        
        # Step 1: Reset Proxy
        self.log("Resetting Proxy Settings...", status="info")
        try:
            subprocess.run('reg add "HKCU\\Software\\Microsoft\\Windows\\CurrentVersion\\Internet Settings" /v ProxyEnable /t REG_DWORD /d 0 /f', shell=True, check=True, creationflags=subprocess.CREATE_NO_WINDOW)
            subprocess.run('reg delete "HKCU\\Software\\Microsoft\\Windows\\CurrentVersion\\Internet Settings" /v ProxyServer /f', shell=True, creationflags=subprocess.CREATE_NO_WINDOW, stderr=subprocess.DEVNULL)
            self.log("Proxy settings reset successfully.", status="success")
        except Exception as e:
            self.log(f"Failed to reset proxy settings: {e}", status="error")
        self.progress_bar.set(0.2)

        # Step 2: Reset Winsock and TCP/IP
        self.run_command("netsh winsock reset", "Resetting Winsock")
        self.progress_bar.set(0.4)
        self.run_command("netsh int ip reset", "Resetting TCP/IP stack")
        self.progress_bar.set(0.6)

        # Step 3: Flush DNS and Renew IP
        self.run_command("ipconfig /flushdns", "Flushing DNS cache")
        self.progress_bar.set(0.8)
        self.run_command("ipconfig /release", "Releasing IP address")
        self.run_command("ipconfig /renew", "Renewing IP address")
        self.progress_bar.set(1.0)

        self.log("\n--- All Operations Completed ---", status="success")
        self.log("IMPORTANT: Please restart your computer to apply all changes.", status="info")
        
        self.reset_button.configure(state="normal", text="Start Full Network Reset")
        self.restart_button.configure(state="normal")

    def confirm_and_restart(self):
        """Show a confirmation dialog before restarting."""
        answer = messagebox.askyesno(
            title="Restart Confirmation",
            message="Are you sure you want to restart your computer now?\nPlease save all your work before proceeding."
        )
        if answer:
            self.log("System will restart in a few seconds...", status="info")
            try:
                subprocess.run("shutdown /r /t 5", shell=True, check=True, creationflags=subprocess.CREATE_NO_WINDOW)
            except Exception as e:
                self.log(f"Failed to initiate restart: {e}", status="error")

    def open_system_command(self, command):
        try:
            subprocess.run(command, shell=True)
        except Exception as e:
            self.log(f"Failed to open '{command}': {e}", status="error")

    def open_link(self, url):
        webbrowser.open_new_tab(url)

def is_admin():
    try:
        return ctypes.windll.shell32.IsUserAnAdmin()
    except:
        return False

if __name__ == "__main__":
    if not is_admin():
        ctypes.windll.user32.MessageBoxW(0, "Administrator privileges are required to run this tool.\nPlease right-click the file and select 'Run as administrator'.", "Access Denied", 0x10 | 0x1000)
        sys.exit()

    app = App()
    app.mainloop()
