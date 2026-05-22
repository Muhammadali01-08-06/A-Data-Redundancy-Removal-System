# A-Data-Redundancy-Removal-System
A Data Redundancy Removal System is a smart software tool that cleans up storage space by finding and removing duplicate data.  Think of it as a digital cleaning assistant that keeps your files organized and efficient.Finds Exact Matches It checks the unique digital fingerprint of files.If two fingerprints match perfectly,it deletes the extra copy.
[task_1.py](https://github.com/user-attachments/files/28138825/task_1.py)
import customtkinter as ctk
import sqlite3
import hashlib
from datetime import datetime
from rapidfuzz import process, fuzz
from tkinter import messagebox, ttk

# Set the appearance of the GUI
ctk.set_appearance_mode("dark")
ctk.set_default_color_theme("blue")

class DRRS_GUI(ctk.CTk):
    def __init__(self):
        super().__init__()

        self.title("Cloud Data Guard - Redundancy Removal System")
        self.geometry("900x600")

        # System Logic Variables
        self.db_path = "cloud_data.db"
        self.threshold = 85
        self._initialize_database()

        # --- UI LAYOUT ---
        self.grid_columnconfigure(1, weight=1)
        self.grid_rowconfigure(0, weight=1)

        # Sidebar for Inputs
        self.sidebar = ctk.CTkFrame(self, width=300, corner_radius=0)
        self.sidebar.grid(row=0, column=0, sticky="nsew")
        
        self.logo_label = ctk.CTkLabel(self.sidebar, text="DATA GUARD", font=ctk.CTkFont(size=20, weight="bold"))
        self.logo_label.pack(pady=20)

        self.entry_label = ctk.CTkLabel(self.sidebar, text="New Data Entry:")
        self.entry_label.pack(pady=(10, 0))
        
        self.data_input = ctk.CTkEntry(self.sidebar, placeholder_text="Enter content here...", width=250)
        self.data_input.pack(pady=10)

        self.category_input = ctk.CTkComboBox(self.sidebar, values=["IT", "Security", "HR", "Finance"], width=250)
        self.category_input.pack(pady=10)

        self.add_btn = ctk.CTkButton(self.sidebar, text="Validate & Append", command=self.process_entry)
        self.add_btn.pack(pady=20)

        self.stats_label = ctk.CTkLabel(self.sidebar, text="System Status: Active", text_color="green")
        self.stats_label.pack(side="bottom", pady=20)

        # Main Content Area
        self.main_frame = ctk.CTkFrame(self, fg_color="transparent")
        self.main_frame.grid(row=0, column=1, padx=20, pady=20, sticky="nsew")

        self.tabview = ctk.CTkTabview(self.main_frame, width=550)
        self.tabview.pack(expand=True, fill="both")
        self.tabview.add("Verified Database")
        self.tabview.add("System Logs")

        # Database Table (Treeview)
        self.setup_table()
        
        # Log Box
        self.log_box = ctk.CTkTextbox(self.tabview.tab("System Logs"), width=500)
        self.log_box.pack(expand=True, fill="both")

        self.refresh_table()

    def _initialize_database(self):
        with sqlite3.connect(self.db_path) as conn:
            conn.execute('''CREATE TABLE IF NOT EXISTS unique_entries 
                           (id INTEGER PRIMARY KEY, raw_content TEXT, normalized_hash TEXT UNIQUE, 
                            category TEXT, created_at TIMESTAMP)''')

    def setup_table(self):
        style = ttk.Style()
        style.theme_use("clam")
        style.configure("Treeview", background="#2b2b2b", foreground="white", fieldbackground="#2b2b2b", borderwidth=0)
        style.map("Treeview", background=[('selected', '#1f538d')])

        self.tree = ttk.Treeview(self.tabview.tab("Verified Database"), columns=("ID", "Content", "Category", "Date"), show='headings')
        self.tree.heading("ID", text="ID")
        self.tree.heading("Content", text="Verified Content")
        self.tree.heading("Category", text="Category")
        self.tree.heading("Date", text="Timestamp")
        self.tree.column("ID", width=50)
        self.tree.pack(expand=True, fill="both")

    def log_message(self, message, level="INFO"):
        timestamp = datetime.now().strftime("%H:%M:%S")
        self.log_box.insert("end", f"[{timestamp}] {level}: {message}\n")
        self.log_box.see("end")

    def refresh_table(self):
        for item in self.tree.get_children():
            self.tree.delete(item)
        with sqlite3.connect(self.db_path) as conn:
            cursor = conn.cursor()
            cursor.execute("SELECT id, raw_content, category, created_at FROM unique_entries ORDER BY id DESC")
            for row in cursor.fetchall():
                self.tree.insert("", "end", values=row)

    def process_entry(self):
        raw_data = self.data_input.get().strip()
        category = self.category_input.get()
        
        if not raw_data:
            messagebox.showwarning("Input Error", "Please enter data to validate.")
            return

        normalized = " ".join(raw_data.lower().split())
        content_hash = hashlib.sha256(normalized.encode()).hexdigest()
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

        with sqlite3.connect(self.db_path) as conn:
            cursor = conn.cursor()

            # 1. Exact Hash Check
            cursor.execute("SELECT id FROM unique_entries WHERE normalized_hash = ?", (content_hash,))
            if cursor.fetchone():
                self.log_message(f"Rejected: Exact duplicate '{raw_data}'", "WARN")
                messagebox.showerror("Rejected", "This exact data already exists in the cloud.")
                return

            # 2. Fuzzy Similarity Check
            cursor.execute("SELECT raw_content FROM unique_entries")
            existing = [row[0] for row in cursor.fetchall()]
            if existing:
                match = process.extractOne(normalized, existing, scorer=fuzz.token_sort_ratio)
                if match and match[1] >= self.threshold:
                    self.log_message(f"Rejected: Similarity {match[1]}% for '{raw_data}'", "WARN")
                    messagebox.showwarning("False Positive", f"Data too similar to: '{match[0]}'")
                    return

            # 3. Success - Add to DB
            cursor.execute("INSERT INTO unique_entries (raw_content, normalized_hash, category, created_at) VALUES (?, ?, ?, ?)",
                           (raw_data, content_hash, category, timestamp))
            conn.commit()
            
        self.log_message(f"Success: Verified '{raw_data}' added.", "SUCCESS")
        self.data_input.delete(0, 'end')
        self.refresh_table()

if __name__ == "__main__":
    app = DRRS_GUI()
    app.mainloop()
