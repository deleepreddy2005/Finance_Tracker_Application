
# Finance_Tracker_Application
import tkinter as tk
from tkinter import ttk, messagebox, filedialog
import matplotlib.pyplot as plt
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
import datetime
import json
import os
import csv

DATA_FILE = "transactions.json"

class FinanceTracker:
    def __init__(self, root):
        self.root = root
        self.root.title("Personal Finance Tracker")
        self.root.geometry("1100x700")

        # state
        self.income = 0.0
        self.transactions = []   # holds both incomes and expenses
        self.savings_goal = 0.0
        self.monthly_limit = 0.0

        self.load_data()
        self.setup_ui()
        self.refresh_transaction_table()

    def setup_ui(self):
        notebook = ttk.Notebook(self.root)
        notebook.pack(pady=10, expand=True, fill='both')

        self.tab1 = ttk.Frame(notebook)
        self.tab2 = ttk.Frame(notebook)
        self.tab3 = ttk.Frame(notebook)
        self.tab4 = ttk.Frame(notebook)
        self.tab5 = ttk.Frame(notebook)

        notebook.add(self.tab1, text='Add Entry')
        notebook.add(self.tab2, text='Transaction History')
        notebook.add(self.tab3, text='Set Goals')
        notebook.add(self.tab4, text='Spending Chart')
        notebook.add(self.tab5, text='Budget Summary')

        self.setup_add_entry_tab()
        self.setup_transaction_history_tab()
        self.setup_goals_tab()
        self.setup_chart_tab()
        self.setup_budget_tab()

    def setup_add_entry_tab(self):
        ttk.Label(self.tab1, text="Type (Income/Expense): ").grid(row=0, column=0, padx=10, pady=10, sticky='w')
        self.entry_type = ttk.Combobox(self.tab1, values=["Income", "Expense"], state="readonly", width=17)
        self.entry_type.set("Expense")
        self.entry_type.grid(row=0, column=1, sticky='w')

        ttk.Label(self.tab1, text="Amount: ").grid(row=1, column=0, padx=10, pady=10, sticky='w')
        self.entry_amount = ttk.Entry(self.tab1)
        self.entry_amount.grid(row=1, column=1, sticky='w')

        ttk.Label(self.tab1, text="Category: ").grid(row=2, column=0, padx=10, pady=10, sticky='w')
        self.entry_category = ttk.Combobox(self.tab1,
                                           values=["Food", "Transport", "Bills", "Shopping", "Entertainment", "Other"],
                                           width=17)
        self.entry_category.set("Other")
        self.entry_category.grid(row=2, column=1, sticky='w')

        ttk.Label(self.tab1, text="Note: ").grid(row=3, column=0, padx=10, pady=10, sticky='w')
        self.entry_note = ttk.Entry(self.tab1, width=30)
        self.entry_note.grid(row=3, column=1, sticky='w')

        ttk.Label(self.tab1, text="Date (YYYY-MM-DD): ").grid(row=4, column=0, padx=10, pady=10, sticky='w')
        self.entry_date = ttk.Entry(self.tab1)
        self.entry_date.insert(0, str(datetime.date.today()))
        self.entry_date.grid(row=4, column=1, sticky='w')

        ttk.Button(self.tab1, text="Add Transaction", command=self.add_entry).grid(row=5, column=1, pady=10, sticky='w')

    def setup_transaction_history_tab(self):
        filter_frame = ttk.Frame(self.tab2)
        filter_frame.pack(pady=5, fill='x')

        ttk.Label(filter_frame, text="Category:").pack(side='left', padx=(5,0))
        self.filter_category = ttk.Entry(filter_frame, width=15)
        self.filter_category.pack(side='left', padx=5)

        ttk.Label(filter_frame, text="Date (YYYY-MM-DD):").pack(side='left', padx=(10,0))
        self.filter_date = ttk.Entry(filter_frame, width=15)
        self.filter_date.pack(side='left', padx=5)

        ttk.Button(filter_frame, text="View Summary", command=self.refresh_transaction_table).pack(side='left', padx=5)
        ttk.Button(filter_frame, text="Export", command=self.export_csv).pack(side='left', padx=5)

        self.tree = ttk.Treeview(self.tab2, columns=("Date", "Type", "Category", "Amount", "Note"), show='headings')
        for col in self.tree["columns"]:
            self.tree.heading(col, text=col)
            self.tree.column(col, anchor='center')
        self.tree.pack(fill='both', expand=True, padx=5, pady=5)

    def setup_goals_tab(self):
        ttk.Label(self.tab3, text="Monthly Income: ").grid(row=0, column=0, padx=10, pady=10, sticky='w')
        self.income_entry = ttk.Entry(self.tab3)
        self.income_entry.grid(row=0, column=1, sticky='w')

        ttk.Label(self.tab3, text="Savings Goal: ").grid(row=1, column=0, padx=10, pady=10, sticky='w')
        self.savings_entry = ttk.Entry(self.tab3)
        self.savings_entry.grid(row=1, column=1, sticky='w')

        ttk.Label(self.tab3, text="Monthly Limit: ").grid(row=2, column=0, padx=10, pady=10, sticky='w')
        self.limit_entry = ttk.Entry(self.tab3)
        self.limit_entry.grid(row=2, column=1, sticky='w')

        ttk.Button(self.tab3, text="Save Goals", command=self.save_goals).grid(row=3, column=1, pady=10, sticky='w')
        self.goal_label = ttk.Label(self.tab3, text="")
        self.goal_label.grid(row=4, column=0, columnspan=3, pady=10, sticky='w')

    def setup_chart_tab(self):
        self.chart_frame = ttk.Frame(self.tab4)
        self.chart_frame.pack(fill='both', expand=True)
        ttk.Button(self.tab4, text="Show Pie Chart", command=self.show_chart_popup).pack(pady=10)

    def setup_budget_tab(self):
        self.budget_text = tk.Text(self.tab5, height=15)
        self.budget_text.pack(pady=10, padx=10, fill='both', expand=True)
        ttk.Button(self.tab5, text="Show Budget Summary", command=self.show_budget_summary).pack(pady=10)

    def add_entry(self):
        try:
            entry_type = self.entry_type.get().strip()
            amount_str = self.entry_amount.get().strip()
            category = self.entry_category.get().strip()
            note = self.entry_note.get().strip()
            date = self.entry_date.get().strip()

            if entry_type not in ("Income", "Expense"):
                raise ValueError("Please select Income or Expense")

            if not amount_str or not amount_str.replace('.', '', 1).isdigit():
                raise ValueError("Amount must be numeric")

            amount = float(amount_str)
            if amount <= 0:
                raise ValueError("Amount must be positive")

            if not category:
                category = "Other"

            # basic date validation
            try:
                datetime.datetime.strptime(date, "%Y-%m-%d")
            except Exception:
                raise ValueError("Date must be in YYYY-MM-DD format")

            entry = {
                'type': entry_type,
                'amount': amount,
                'category': category,
                'note': note,
                'date': date
            }
            self.transactions.append(entry)

            if entry_type == "Income":
                self.income += amount

            self.save_data()
            self.refresh_transaction_table()
            messagebox.showinfo("Success", f"{entry_type} of Rs.{amount:.2f} added to {category}")

            if entry_type == "Expense" and self.monthly_limit and self.get_total_expenses() > self.monthly_limit:
                messagebox.showwarning("Budget Exceeded", "You have exceeded your monthly budget limit!")
        except ValueError as ve:
            messagebox.showerror("Invalid Input", str(ve))
        except Exception as e:
            messagebox.showerror("Error", f"An unexpected error occurred: {e}")

    def refresh_transaction_table(self):
        for item in self.tree.get_children():
            self.tree.delete(item)

        filter_cat = self.filter_category.get().strip().lower()
        filter_date = self.filter_date.get().strip()

        for entry in self.transactions:
            if (not filter_cat or filter_cat in entry['category'].lower()) and \
               (not filter_date or filter_date == entry['date']):
                self.tree.insert('', 'end', values=(entry['date'], entry['type'], entry['category'],
                                                    f"Rs.{entry['amount']:.2f}", entry['note']))

    def save_goals(self):
        try:
            inc_text = self.income_entry.get().strip()
            self.income = float(inc_text) if inc_text else self.income
            self.savings_goal = float(self.savings_entry.get().strip() or self.savings_goal)
            self.monthly_limit = float(self.limit_entry.get().strip() or self.monthly_limit)
            self.goal_label.config(text=f"Income: Rs.{self.income:.2f}, Goal: Rs.{self.savings_goal:.2f}, Limit: Rs.{self.monthly_limit:.2f}")
            self.save_data()
        except ValueError:
            messagebox.showerror("Invalid Input", "Please enter valid numbers for goals")

    def show_chart_popup(self):
        expenses_only = [e for e in self.transactions if e['type'] == 'Expense']
        if not expenses_only:
            messagebox.showinfo("No Data", "No expenses to visualize")
            return

        categories = {}
        for item in expenses_only:
            categories[item['category']] = categories.get(item['category'], 0) + item['amount']

        fig, ax = plt.subplots(figsize=(6, 4))
        ax.pie(list(categories.values()), labels=list(categories.keys()), autopct='%1.1f%%', startangle=90)
        ax.set_title("Expense Distribution by Category")

        chart_window = tk.Toplevel(self.root)
        chart_window.title("Spending Chart")
        chart = FigureCanvasTkAgg(fig, chart_window)
        chart.get_tk_widget().pack(fill='both', expand=True)
        chart.draw()

    def show_budget_summary(self):
        total_expenses = self.get_total_expenses()
        available_budget = self.income - self.savings_goal
        self.budget_text.delete('1.0', tk.END)
        self.budget_text.insert(tk.END, f"Income: Rs.{self.income:.2f}\n")
        self.budget_text.insert(tk.END, f"Savings Goal: Rs.{self.savings_goal:.2f}\n")
        self.budget_text.insert(tk.END, f"Available Budget: Rs.{available_budget:.2f}\n")
        self.budget_text.insert(tk.END, f"Total Spent: Rs.{total_expenses:.2f}\n")
        self.budget_text.insert(tk.END, f"Monthly Limit: Rs.{self.monthly_limit:.2f}\n")
        status = "Under Budget" if total_expenses <= self.monthly_limit else "Over Budget"
        self.budget_text.insert(tk.END, f"Status: {status}\n")

    def get_total_expenses(self):
        return sum(e['amount'] for e in self.transactions if e['type'] == 'Expense')

    def save_data(self):
        try:
            with open(DATA_FILE, 'w') as f:
                json.dump({
                    'income': self.income,
                    'savings_goal': self.savings_goal,
                    'monthly_limit': self.monthly_limit,
                    'transactions': self.transactions
                }, f, indent=2)
        except Exception as e:
            messagebox.showerror("Save Error", f"Failed to save data: {e}")

    def load_data(self):
        try:
            if os.path.exists(DATA_FILE):
                with open(DATA_FILE, 'r') as f:
                    data = json.load(f)
                    self.income = float(data.get('income', 0.0))
                    self.savings_goal = float(data.get('savings_goal', 0.0))
                    self.monthly_limit = float(data.get('monthly_limit', 0.0))
                    self.transactions = data.get('transactions', [])
        except Exception as e:
            messagebox.showerror("Load Error", f"Failed to load data: {e}")

    def export_csv(self):
        file_path = filedialog.asksaveasfilename(defaultextension=".csv", filetypes=[("CSV files", "*.csv")])
        if file_path:
            try:
                with open(file_path, 'w', newline='') as f:
                    writer = csv.writer(f)
                    writer.writerow(["Date", "Type", "Category", "Amount", "Note"])
                    for entry in self.transactions:
                        writer.writerow([entry['date'], entry['type'], entry['category'], f"{entry['amount']:.2f}", entry['note']])
                messagebox.showinfo("Export Successful", f"Data exported to {file_path}")
            except Exception as e:
                messagebox.showerror("Export Failed", str(e))

if __name__ == "__main__":
    root = tk.Tk()
    app = FinanceTracker(root)
    root.mainloop()
