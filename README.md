import wx
import re
import random
import datetime
import os
from pathlib import Path
import wx.adv

APP_DIR = Path.home() / ".niha_app"
APP_DIR.mkdir(parents=True, exist_ok=True)
USERS_FILE = APP_DIR / "niha.txt"
APPTS_FILE = APP_DIR / "names.txt"

PASSWORD_PATTERN = re.compile(r"[A-Za-z0-9]+[!@#$%^&*]+")

def save_user(username, password, age, bloodgroup):
    with open(USERS_FILE, "a", encoding="utf-8") as f:
        f.write(f"{username},{password},{age},{bloodgroup}\n")

def check_credentials(username, password):
    if not USERS_FILE.exists():
        return False
    with open(USERS_FILE, "r", encoding="utf-8") as f:
        for line in f:
            parts = line.strip().split(',')
            if len(parts) >= 2 and parts[0] == username and parts[1] == password:
                return True
    return False

def append_appointment(name, token, dt, doc):
    with open(APPTS_FILE, 'a', encoding='utf-8') as f:
        f.write(name + "\n")
        f.write(str(token) + "\n")
        f.write(str(dt) + "\n")
        f.write(doc + "\n")

class SignupDialog(wx.Dialog):
    def __init__(self, parent):
        super().__init__(parent, title="Sign Up", size=(500,500))
        panel = wx.Panel(self)
        sizer = wx.BoxSizer(wx.VERTICAL)

        self.user_txt = wx.TextCtrl(panel)
        self.pass_txt = wx.TextCtrl(panel, style=wx.TE_PASSWORD)
        self.age_txt = wx.TextCtrl(panel)
        self.bg_txt = wx.TextCtrl(panel)

        sizer.Add(wx.StaticText(panel, label="Username:"), 0, wx.ALL, 5)
        sizer.Add(self.user_txt, 0, wx.EXPAND|wx.ALL, 5)
        sizer.Add(wx.StaticText(panel, label="Password (use a special char):"), 0, wx.ALL, 5)
        sizer.Add(self.pass_txt, 0, wx.EXPAND|wx.ALL, 5)
        sizer.Add(wx.StaticText(panel, label="Age:"), 0, wx.ALL, 5)
        sizer.Add(self.age_txt, 0, wx.EXPAND|wx.ALL, 5)
        sizer.Add(wx.StaticText(panel, label="Blood group:"), 0, wx.ALL, 5)
        sizer.Add(self.bg_txt, 0, wx.EXPAND|wx.ALL, 5)

        btn = wx.Button(panel, label="Sign Up")
        btn.Bind(wx.EVT_BUTTON, self.on_signup)
        sizer.Add(btn, 0, wx.ALIGN_CENTER|wx.ALL, 10)

        panel.SetSizer(sizer)

    def on_signup(self, event):
        username = self.user_txt.GetValue().strip()
        password = self.pass_txt.GetValue().strip()
        age = self.age_txt.GetValue().strip()
        bg = self.bg_txt.GetValue().strip()
        if not username or not password:
            wx.MessageBox("Please provide username and password.", "Error", wx.OK|wx.ICON_ERROR)
            return
        if not PASSWORD_PATTERN.match(password):
            wx.MessageBox("Password must include at least one special character (!@#$%^&*).", "Weak password", wx.OK|wx.ICON_WARNING)
            return
        try:
            age_int = int(age)
        except ValueError:
            wx.MessageBox("Age must be a number.", "Error", wx.OK|wx.ICON_ERROR)
            return
        save_user(username, password, age_int, bg)
        wx.MessageBox("Signup successful!", "Success", wx.OK|wx.ICON_INFORMATION)
        self.EndModal(wx.ID_OK)

class BMIDialog(wx.Dialog):
    def __init__(self, parent):
        super().__init__(parent, title="BMI Calculator", size=(300,250))
        panel = wx.Panel(self)
        s = wx.BoxSizer(wx.VERTICAL)
        self.h = wx.TextCtrl(panel)
        self.w = wx.TextCtrl(panel)
        s.Add(wx.StaticText(panel, label="Height (meters):"), 0, wx.ALL, 5)
        s.Add(self.h, 0, wx.EXPAND|wx.ALL, 5)
        s.Add(wx.StaticText(panel, label="Weight (kg):"), 0, wx.ALL, 5)
        s.Add(self.w, 0, wx.EXPAND|wx.ALL, 5)
        btn = wx.Button(panel, label="Calculate")
        btn.Bind(wx.EVT_BUTTON, self.calculate)
        s.Add(btn, 0, wx.ALIGN_CENTER|wx.ALL, 10)
        panel.SetSizer(s)

    def calculate(self, event):
        try:
            h = float(self.h.GetValue())
            w = float(self.w.GetValue())
            bmi = w / (h**2)
            msg = f"Your BMI is: {bmi:.2f}\n"
            if bmi < 18.5:
                msg += "You are underweight."
            elif bmi <= 24.9:
                msg += "You are normal weight."
            elif bmi <= 29.9:
                msg += "You are overweight."
            else:
                msg += "Alert! You are obese."
            wx.MessageBox(msg, "BMI Result", wx.OK|wx.ICON_INFORMATION)
        except Exception:
            wx.MessageBox("Please enter valid numbers.", "Error", wx.OK|wx.ICON_ERROR)

class SymptomDialog(wx.Dialog):
    SYM_DICT = {
        1: ("Viral fever", "Note your temperature. If temperature between 99F to 103F - HIGH RISK."),
        2: ("Common cold", "Consult the doctor when the cold exceeds 7 days."),
        3: ("Food poisoning", "Consult doctor immediately!"),
        4: ("Common flu", "Consult the doctor when the cough exceeds 7 days."),
        5: ("Gastric problem", "For mild pain use Milk of magnesia."),
        6: ("Fungal skin infection", "Consult your dermatologist."),
        7: ("Cholera", "Consult doctor immediately!"),
        8: ("Migraine", "Don't use medications without doctor's consult."),
        9: ("Possible heart problem", "Get scans and consult cardiologist."),
        10:("Vertigo", "Get your appointment ready."),
    }
    def __init__(self, parent):
        super().__init__(parent, title="Symptom Checker", size=(400,350))
        panel = wx.Panel(self)
        v = wx.BoxSizer(wx.VERTICAL)
        st = wx.StaticText(panel, label="Select a symptom:")
        v.Add(st, 0, wx.ALL, 5)
        self.choice = wx.ListBox(panel, choices=[f"{k}. {v[0]}" for k,v in self.SYM_DICT.items()], style=wx.LB_SINGLE)
        v.Add(self.choice, 1, wx.EXPAND|wx.ALL, 5)
        btn = wx.Button(panel, label="Check")
        btn.Bind(wx.EVT_BUTTON, self.on_check)
        v.Add(btn, 0, wx.ALIGN_CENTER|wx.ALL, 10)
        panel.SetSizer(v)

    def on_check(self, event):
        sel = self.choice.GetSelection()
        if sel == wx.NOT_FOUND:
            wx.MessageBox("Please select a symptom.", "Error", wx.OK|wx.ICON_ERROR)
            return
        key = sel + 1
        title, advice = self.SYM_DICT.get(key, ("Unknown","Please consult doctor."))
        wx.MessageBox(f"You might have: {title}\nAdvice: {advice}", "Symptom Result", wx.OK|wx.ICON_INFORMATION)

class AgeDialog(wx.Dialog):

    def __init__(self, parent):
        super().__init__(parent, title="Age Group Analyzer", size=(300,200))
        panel = wx.Panel(self)
        s = wx.BoxSizer(wx.VERTICAL)
        self.age = wx.TextCtrl(panel)
        s.Add(wx.StaticText(panel, label="Enter age:"), 0, wx.ALL, 5)
        s.Add(self.age, 0, wx.EXPAND|wx.ALL, 5)
        btn = wx.Button(panel, label="Analyze")
        btn.Bind(wx.EVT_BUTTON, self.on_analyze)
        s.Add(btn, 0, wx.ALIGN_CENTER|wx.ALL, 10)
        panel.SetSizer(s)

    def on_analyze(self, event):
        try:
            age = int(self.age.GetValue())
            if age <= 0:
                res = "You don't exist (invalid age)."
            elif age <= 12:
                res = "You are a CHILD"
            elif age <= 19:
                res = "You are a TEENAGER"
            elif age <= 59:
                res = "You are an ADULT"
            else:
                res = "Senior Citizen"
            wx.MessageBox(res, "Age Group", wx.OK|wx.ICON_INFORMATION)
        except Exception:
            wx.MessageBox("Enter a valid integer age.", "Error", wx.OK|wx.ICON_ERROR)

class AppointmentDialog(wx.Dialog):
    DOCTORS = {
        1: ("General Physician", ["Dr.Anil","Dr.Jaya"]),
        2: ("Cardiologist", ["Dr.Ram","Dr.Yajas"]),
        3: ("Pediatrician", ["Dr.Arvind","Dr.Sudhanva"]),
        4: ("Dermatologist", ["Dr.Vartika","Dr.Aditi"]),
        5: ("Psychiatrist", ["Dr.Krishna","Dr.Yamini"]),
    }

    def __init__(self, parent):
        super().__init__(parent, title="Appointment Generator", size=(400,420))
        panel = wx.Panel(self)
        v = wx.BoxSizer(wx.VERTICAL)

        self.name = wx.TextCtrl(panel)
        v.Add(wx.StaticText(panel, label="Enter your name:"), 0, wx.ALL, 5)
        v.Add(self.name, 0, wx.EXPAND|wx.ALL, 5)

        
        v.Add(wx.StaticText(panel, label="Select requirement:"), 0, wx.ALL, 5)
        self.req_choice = wx.Choice(panel, choices=[f"{k}. {v[0]}" for k,v in self.DOCTORS.items()])
        v.Add(self.req_choice, 0, wx.EXPAND|wx.ALL, 5)

       
        v.Add(wx.StaticText(panel, label="Select doctor:"), 0, wx.ALL, 5)
        self.doc_choice = wx.Choice(panel, choices=["(choose requirement first)"])
        v.Add(self.doc_choice, 0, wx.EXPAND|wx.ALL, 5)

        self.req_choice.Bind(wx.EVT_CHOICE, self.on_req)

        
        v.Add(wx.StaticText(panel, label="Select appointment date:"), 0, wx.ALL, 5)
        self.date_picker = wx.adv.DatePickerCtrl(panel, style=wx.adv.DP_DROPDOWN)
        v.Add(self.date_picker, 0, wx.EXPAND|wx.ALL, 5)

        
        v.Add(wx.StaticText(panel, label="Enter time (HH:MM, 24-hour format):"), 0, wx.ALL, 5)
        self.time_txt = wx.TextCtrl(panel, value="10:00")
        v.Add(self.time_txt, 0, wx.EXPAND|wx.ALL, 5)

        
        btn = wx.Button(panel, label="Generate Appointment")
        btn.Bind(wx.EVT_BUTTON, self.on_generate)
        v.Add(btn, 0, wx.ALIGN_CENTER|wx.ALL, 10)

        panel.SetSizer(v)

    def on_req(self, event):
        sel = self.req_choice.GetSelection()
        if sel == wx.NOT_FOUND:
            return
        key = sel + 1
        docs = self.DOCTORS[key][1]
        self.doc_choice.Clear()
        self.doc_choice.AppendItems(docs)
        self.doc_choice.SetSelection(0)

    def on_generate(self, event):
        name = self.name.GetValue().strip()
        if not name:
            wx.MessageBox("Enter name.", "Error", wx.OK|wx.ICON_ERROR)
            return

        req_sel = self.req_choice.GetSelection()
        doc_sel = self.doc_choice.GetSelection()
        if req_sel == wx.NOT_FOUND or doc_sel == wx.NOT_FOUND:
            wx.MessageBox("Select requirement and doctor.", "Error", wx.OK|wx.ICON_ERROR)
            return

        
        doc = self.doc_choice.GetString(doc_sel)

        
        token = random.randint(1000,10000)

        
        date_obj = self.date_picker.GetValue()
        date_py = datetime.date(date_obj.GetYear(), date_obj.GetMonth()+1, date_obj.GetDay())

        
        time_str = self.time_txt.GetValue().strip()
        try:
            t = datetime.datetime.strptime(time_str, "%H:%M").time()
        except:
            wx.MessageBox("Invalid time format. Use HH:MM.", "Error", wx.OK|wx.ICON_ERROR)
            return

        
        appointment_dt = datetime.datetime.combine(date_py, t)

        
        append_appointment(name, token, appointment_dt, doc)

        wx.MessageBox(
            f"Appointment Booked!\n\nName: {name}\nDoctor: {doc}\nToken: {token}\n"
            f"Date & Time: {appointment_dt}",
            "Success",
            wx.OK | wx.ICON_INFORMATION
        )

        self.EndModal(wx.ID_OK)


class MainFrame(wx.Frame):
    def __init__(self, parent, title, username=None):
        super().__init__(parent, title=title, size=(450,400))
        panel = wx.Panel(self)
        v = wx.BoxSizer(wx.VERTICAL)

        welcome = wx.StaticText(panel, label=f"Welcome, {username if username else 'User'}")
        v.Add(welcome, 0, wx.ALL, 10)

        btn_bmi = wx.Button(panel, label="BMI Calculator")
        btn_sym = wx.Button(panel, label="Symptom Checker")
        btn_age = wx.Button(panel, label="Age Group Analyzer")
        btn_app = wx.Button(panel, label="Appointment Generator")
        btn_logout = wx.Button(panel, label="Logout")

        btn_bmi.Bind(wx.EVT_BUTTON, self.open_bmi)
        btn_sym.Bind(wx.EVT_BUTTON, self.open_symptom)
        btn_age.Bind(wx.EVT_BUTTON, self.open_age)
        btn_app.Bind(wx.EVT_BUTTON, self.open_appointment)
        btn_logout.Bind(wx.EVT_BUTTON, self.on_logout)

        for b in (btn_bmi, btn_sym, btn_age, btn_app, btn_logout):
            v.Add(b, 0, wx.EXPAND|wx.ALL, 8)

        panel.SetSizer(v)
        self.Centre()

    def open_bmi(self, event):
        dlg = BMIDialog(self)
        dlg.ShowModal()
        dlg.Destroy()

    def open_symptom(self, event):
        dlg = SymptomDialog(self)
        dlg.ShowModal()
        dlg.Destroy()

    def open_age(self, event):
        dlg = AgeDialog(self)
        dlg.ShowModal()
        dlg.Destroy()

    def open_appointment(self, event):
        dlg = AppointmentDialog(self)
        dlg.ShowModal()
        dlg.Destroy()

    def on_logout(self, event):
        self.Close()

class LoginFrame(wx.Frame):
    def __init__(self):
        super().__init__(None, title="Health App", size=(350,300))
        panel = wx.Panel(self)
        s = wx.BoxSizer(wx.VERTICAL)

        s.Add(wx.StaticText(panel, label="Username:"), 0, wx.ALL, 5)
        self.user = wx.TextCtrl(panel)
        s.Add(self.user, 0, wx.EXPAND|wx.ALL, 5)

        s.Add(wx.StaticText(panel, label="Password:"), 0, wx.ALL, 5)
        self.pwd = wx.TextCtrl(panel, style=wx.TE_PASSWORD)
        s.Add(self.pwd, 0, wx.EXPAND|wx.ALL, 5)

        btn_login = wx.Button(panel, label="Login")
        btn_signup = wx.Button(panel, label="Sign Up")
        btn_exit = wx.Button(panel, label="Exit")

        h = wx.BoxSizer(wx.HORIZONTAL)
        h.Add(btn_login, 1, wx.EXPAND|wx.ALL, 5)
        h.Add(btn_signup, 1, wx.EXPAND|wx.ALL, 5)++++
        h.Add(btn_exit, 1, wx.EXPAND|wx.ALL, 5)

        s.Add(h, 0, wx.EXPAND)
        panel.SetSizer(s)

        btn_login.Bind(wx.EVT_BUTTON, self.on_login)
        btn_signup.Bind(wx.EVT_BUTTON, self.on_signup)
        btn_exit.Bind(wx.EVT_BUTTON, lambda e: self.Close())

        self.Centre()

    def on_login(self, event):
        username = self.user.GetValue().strip()
        password = self.pwd.GetValue().strip()
        if check_credentials(username, password):
            wx.MessageBox("Login successful", "Welcome", wx.OK|wx.ICON_INFORMATION)
            self.Hide()
            mf = MainFrame(None, "Health App", username=username)
            mf.Show()
        else:
            wx.MessageBox("Invalid credentials", "Error", wx.OK|wx.ICON_ERROR)

    def on_signup(self, event):
        dlg = SignupDialog(self)
        res = dlg.ShowModal()
        dlg.Destroy()


if __name__ == '__main__':
    app = wx.App(False)
    frame = LoginFrame()
    frame.Show()
    app.MainLoop()
