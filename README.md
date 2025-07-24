
pyinstaller --onefile --windowed --name "Network Reset Tool" --icon="icon.ico" network_reset_tool.py


import customtkinter as ctk
import subprocess
import ctypes
import sys
import webbrowser

# --- App Configuration ---
WINDOW_TITLE = "Network Reset Tool"
WINDOW_GEOMETRY = "480x580"

class App(ctk.CTk):
    """The main application class for the Network Reset Tool."""
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)

        # --- Main Window Setup ---
        self.title(WINDOW_TITLE)
        self.geometry(WINDOW_GEOMETRY)
        self.resizable(False, False)
        ctk.set_appearance_mode("dark")

        # --- Main Frame ---
        self.main_frame = ctk.CTkFrame(self)
        self.main_frame.pack(pady=20, padx=20, fill="both", expand=True)

        # --- Title and Description ---
        self.title_label = ctk.CTkLabel(self.main_frame, text="Network Reset Tool", font=ctk.CTkFont(size=20, weight="bold"))
        self.title_label.pack(pady=(10, 5))
        self.description_label = ctk.CTkLabel(self.main_frame, text="This tool resets your network settings to their default state.", font=ctk.CTkFont(size=12))
        self.description_label.pack(pady=(0, 20))

        # --- Primary Action Button ---
        self.reset_button = ctk.CTkButton(self.main_frame, text="Start Full Network Reset", command=self.run_full_reset, height=40, font=ctk.CTkFont(size=14, weight="bold"))
        self.reset_button.pack(pady=10, padx=10, fill="x")

        # --- Log Display Textbox ---
        self.log_textbox = ctk.CTkTextbox(self.main_frame, height=150, state="disabled", font=("Consolas", 11))
        self.log_textbox.pack(pady=10, padx=10, fill="both", expand=True)

        # --- Suggestions & Shortcuts Frame ---
        self.suggestions_frame = ctk.CTkFrame(self.main_frame)
        self.suggestions_frame.pack(pady=10, padx=10, fill="x")
        
        # --- FIX: Use grid for ALL children of suggestions_frame ---
        # Configure the grid to have two equally weighted columns
        self.suggestions_frame.grid_columnconfigure((0, 1), weight=1)

        self.suggestions_label = ctk.CTkLabel(self.suggestions_frame, text="System Shortcuts", font=ctk.CTkFont(size=12, weight="bold"))
        # Use grid for the label and make it span both columns
        self.suggestions_label.grid(row=0, column=0, columnspan=2, padx=10, pady=(10, 10), sticky="ew")

        shortcuts = {
            "Network & Internet": "ms-settings:network-status",
            "Proxy Settings": "ms-settings:network-proxy",
            "Network Adapters": "control.exe ncpa.cpl",
            "Windows Firewall": "control.exe firewall.cpl"
        }
        
        # Loop through shortcuts and place them in the grid, starting from row 1
        for i, (text, command) in enumerate(shortcuts.items()):
            row = (i // 2) + 1  # Start from row 1, below the label
            col = i % 2
            button = ctk.CTkButton(self.suggestions_frame, text=text, command=lambda c=command: self.open_system_command(c))
            button.grid(row=row, column=col, padx=5, pady=5, sticky="ew")

        # --- Footer with clickable link ---
        # A separate frame for the footer to place it at the bottom of the window
        footer_frame = ctk.CTkFrame(self, fg_color="transparent")
        footer_frame.pack(side="bottom", fill="x", pady=(0, 10))

        footer_label = ctk.CTkLabel(footer_frame, text="Created with ❤️ by GeekNeuron", cursor="hand2", font=ctk.CTkFont(size=12))
        footer_label.pack()
        # Set a specific color to make it look like a link
        footer_label.configure(text_color="#85C1E9")
        # Bind the click event to the open_link function
        footer_label.bind("<Button-1>", lambda e: self.open_link("https://github.com/GeekNeuron"))


    def log(self, message):
        """Appends a message to the log display."""
        self.log_textbox.configure(state="normal")
        self.log_textbox.insert("end", message + "\n")
        self.log_textbox.see("end")
        self.log_textbox.configure(state="disabled")
        self.update_idletasks()

    def run_command(self, command, log_message):
        """Executes a shell command and logs its progress."""
        self.log(f"[*] {log_message}...")
        try:
            creation_flags = subprocess.CREATE_NO_WINDOW
            subprocess.run(command, shell=True, check=True, capture_output=True, creationflags=creation_flags)
            self.log(f"[+] {log_message} completed successfully.")
            return True
        except subprocess.CalledProcessError as e:
            self.log(f"[!] Error executing command: {e}")
            return False

    def run_full_reset(self):
        """Executes the complete network reset sequence."""
        self.reset_button.configure(state="disabled", text="Resetting, please wait...")
        self.log_textbox.configure(state="normal")
        self.log_textbox.delete("1.0", "end")
        self.log_textbox.configure(state="disabled")

        self.log("--- Starting Network Reset Process ---")
        
        # Step 1: Reset Proxy Settings
        self.log("[*] Step 1: Resetting Proxy Settings...")
        try:
            creation_flags = subprocess.CREATE_NO_WINDOW
            subprocess.run('reg add "HKCU\\Software\\Microsoft\\Windows\\CurrentVersion\\Internet Settings" /v ProxyEnable /t REG_DWORD /d 0 /f', shell=True, check=True, creationflags=creation_flags)
            subprocess.run('reg delete "HKCU\\Software\\Microsoft\\Windows\\CurrentVersion\\Internet Settings" /v ProxyServer /f', shell=True, creationflags=creation_flags, stderr=subprocess.DEVNULL)
            self.log("[+] Proxy settings reset successfully.")
        except Exception as e:
            self.log(f"[!] Failed to reset proxy settings: {e}")
            
        # Step 2: Reset Winsock and TCP/IP
        self.run_command("netsh winsock reset", "Resetting Winsock")
        self.run_command("netsh int ip reset", "Resetting TCP/IP stack")

        # Step 3: Release/Renew IP and Flush DNS
        self.run_command("ipconfig /release", "Releasing IP address")
        self.run_command("ipconfig /renew", "Renewing IP address")
        self.run_command("ipconfig /flushdns", "Flushing DNS cache")

        self.log("\n--- All Operations Completed ---")
        self.log("!!! IMPORTANT: Please restart your computer for all changes to take effect.")
        self.reset_button.configure(state="normal", text="Start Full Network Reset")
    
    def open_system_command(self, command):
        """Opens various Windows settings panels."""
        try:
            subprocess.run(command, shell=True)
        except Exception as e:
            self.log(f"[!] Failed to open settings: {e}")

    def open_link(self, url):
        """Opens a URL in the default web browser."""
        webbrowser.open_new_tab(url)


def is_admin():
    """Checks if the script is running with administrative privileges."""
    try:
        return ctypes.windll.shell32.IsUserAnAdmin()
    except:
        return False

if __name__ == "__main__":
    if not is_admin():
        ctypes.windll.user32.MessageBoxW(0, 
            "Administrator privileges are required to run this tool.\nPlease right-click the file and select 'Run as administrator'.", 
            "Access Denied", 0x10)
        sys.exit()

    app = App()
    app.mainloop()
