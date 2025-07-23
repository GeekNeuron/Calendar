import customtkinter as ctk
import subprocess
import ctypes
import sys

# --- App Configuration ---
WINDOW_TITLE = "Network Reset Tool"
WINDOW_GEOMETRY = "450x620" # Increased height for the new button

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
        self.suggestions_frame.pack(pady=(10, 5), padx=10, fill="x")
        
        self.suggestions_label = ctk.CTkLabel(self.suggestions_frame, text="System Shortcuts", font=ctk.CTkFont(size=12, weight="bold"))
        self.suggestions_label.pack(pady=(10, 10))

        # Buttons for opening Windows settings
        shortcuts = {
            "Network & Internet Settings": "ms-settings:network-status",
            "Proxy Settings": "ms-settings:network-proxy",
            "Network Adapters (ncpa.cpl)": "control.exe ncpa.cpl",
            "Windows Defender Firewall": "control.exe firewall.cpl", # New Suggestion
            "Device Manager": "devmgmt.msc" # New Suggestion
        }
        for text, command in shortcuts.items():
            button = ctk.CTkButton(self.suggestions_frame, text=text, command=lambda c=command: self.open_system_command(c))
            button.pack(pady=4, padx=20, fill="x")
        
        # Add some padding at the bottom of the frame
        ctk.CTkLabel(self.suggestions_frame, text="").pack()

        # --- Restart Button ---
        self.restart_button = ctk.CTkButton(self.main_frame, text="Restart Computer", command=self.restart_computer, state="disabled", fg_color="#C0392B", hover_color="#A93226")
        self.restart_button.pack(pady=(10, 10), padx=10, fill="x")

    def log(self, message):
        """Appends a message to the log display."""
        self.log_textbox.configure(state="normal")
        self.log_textbox.insert("end", message + "\n")
        self.log_textbox.see("end") # Scroll to the bottom
        self.log_textbox.configure(state="disabled")
        self.update_idletasks() # Force UI update

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
        # Disable buttons during operation
        self.reset_button.configure(state="disabled", text="Resetting, please wait...")
        self.restart_button.configure(state="disabled") # Disable restart button at the start
        
        # Clear previous logs
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
        
        # Re-enable buttons and enable the restart button
        self.reset_button.configure(state="normal", text="Start Full Network Reset")
        self.restart_button.configure(state="normal") # Enable restart button on completion

    def open_system_command(self, command):
        """Opens various Windows settings panels."""
        try:
            subprocess.run(command, shell=True, creationflags=subprocess.CREATE_NO_WINDOW)
        except Exception as e:
            self.log(f"[!] Failed to open settings: {e}")

    def restart_computer(self):
        """Initiates a system restart."""
        self.log("[*] System will restart in a few seconds...")
        try:
            # Shutdown command: /r for restart, /t 1 for 1 second delay
            subprocess.run("shutdown /r /t 1", shell=True, check=True, creationflags=subprocess.CREATE_NO_WINDOW)
        except Exception as e:
            self.log(f"[!] Failed to initiate restart: {e}")


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
