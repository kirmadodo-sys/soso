# soso
# c52_simulator_gui.py
# Python 3 standard libraries only.
# C-52 style simulator GUI
# 6 fixed wheels, 27 Bars x 6 positions (0 = Lug present, 1 = no Lug)

import tkinter as tk
from tkinter import ttk, messagebox, filedialog
import json
import random
from typing import List
ALPHABET = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"

def idx(c): return ALPHABET.index(c)
def chr_i(i): return ALPHABET[i % 26]

# -----------------------
# Core simulation engine
# -----------------------
class C52Sim:
    def __init__(self, wheel_lengths: List[int]=None, pins: List[List[int]]=None,
                 lug_cage: List[List[int]]=None, positions: List[int]=None, stepping_mode='carry'):
        # default 6 wheels lengths
        if wheel_lengths is None:
            wheel_lengths = [25,26,29,31,34,37]
        # enforce 6 wheels
        if len(wheel_lengths) < 6:
            raise ValueError("wheel_lengths must contain at least 6 values")
        self.nw = 6  # fixed 6 wheels
        self.L = wheel_lengths[:self.nw]
        # pins: list of lists (0/1) for each wheel
        if pins is None:
            random.seed(0)
            pins = [[random.randint(0,1) for _ in range(l)] for l in self.L]
        # validate pins shape
        if len(pins) != self.nw:
            raise ValueError("pins must have exactly 6 lists")
        for i, p in enumerate(pins):
            if len(p) != self.L[i]:
                raise ValueError(f"pins[{i}] length must equal wheel_lengths[{i}]")
        self.pins = [p[:] for p in pins]
        # lug_cage: 27 Bars x 6 positions (0 = Lug present, 1 = no Lug)
        if lug_cage is None:
            lug_cage = []
            for _ in range(27):
                lug_cage.append([random.randint(0,1) for _ in range(6)])
        # validate lug_cage
        if len(lug_cage) != 27:
            raise ValueError("lug_cage must have exactly 27 Bars")
        for bar in lug_cage:
            if len(bar) != 6:
                raise ValueError("each Bar must have exactly 6 positions")
            for v in bar:
                if v not in (0,1):
                    raise ValueError("lug flags must be 0 or 1")
        self.lug_cage = [bar[:] for bar in lug_cage]
        # positions: current index for each wheel
        if positions is None:
            positions = [0]*self.nw
        if len(positions) != self.nw:
            raise ValueError("positions must have exactly 6 values")
        for i, p in enumerate(positions):
            if not (0 <= p < self.L[i]):
                raise ValueError(f"position {i} out of range 0..{self.L[i]-1}")
        self.positions = positions[:]
        self.stepping_mode = stepping_mode  # 'carry' or 'fixed'

    def reset_positions(self, pos_list):
        if len(pos_list) != self.nw:
            raise ValueError("positions length mismatch")
        for i, p in enumerate(pos_list):
            if p < 0 or p >= self.L[i]:
                raise ValueError(f"position {i} out of range")
        self.positions = pos_list[:]

    def gen_keystream_value(self) -> int:
        # Evaluate each bar: activated if ANY position has Lug present (0) and corresponding wheel's pin==1
        lug_count = 0
        for bar in self.lug_cage:  # bar = list of 6 positions
            activated = False
            for wheel_idx in range(self.nw):
                if bar[wheel_idx] == 0:  # 0 = Lug present (per your convention)
                    if self.pins[wheel_idx][self.positions[wheel_idx]] == 1:
                        activated = True
                        break
            if activated:
                lug_count += 1
        return lug_count % 26

    def step_wheels(self, old_positions=None):
        if old_positions is None:
            old_positions = self.positions[:]
        # wheel0 always steps
        self.positions[0] = (self.positions[0] + 1) % self.L[0]
        if self.stepping_mode == 'carry':
            for i in range(1, self.nw):
                prev_pin = self.pins[i-1][old_positions[i-1]]
                if prev_pin == 1:
                    self.positions[i] = (self.positions[i] + 1) % self.L[i]
        elif self.stepping_mode == 'fixed':
            # fixed stepping: do nothing (only wheel0 steps)
            pass
        else:
            # default fallback: carry-like
            for i in range(1, self.nw):
                prev_pin = self.pins[i-1][old_positions[i-1]]
                if prev_pin == 1:
                    self.positions[i] = (self.positions[i] + 1) % self.L[i]

    def process_text(self, text: str, encrypt=True, show_steps=False):
        text = text.upper()
        out = []
        steps_info = []
        for ch in text:
            if ch not in ALPHABET:
                out.append(ch)
                if show_steps:
                    steps_info.append({'char': ch, 'skipped': True})
                continue
            old_pos = self.positions[:]
            K = self.gen_keystream_value()
            p = idx(ch)
            if encrypt:
                c = (p + K) % 26
            else:
                c = (p - K) % 26
            out.append(chr_i(c))
            if show_steps:
                steps_info.append({
                    'char': ch,
                    'P': p,
                    'K': K,
                    'C': c,
                    'Cchar': chr_i(c),
                    'positions_before': old_pos[:],
                    'positions_after': None
                })
            self.step_wheels(old_pos)
            if show_steps:
                steps_info[-1]['positions_after'] = self.positions[:]
        return "".join(out), steps_info

    def export_key(self):
        return {
            'wheel_lengths': self.L,
            'pins': self.pins,
            'lug_cage': self.lug_cage,
            'positions': self.positions,
            'stepping_mode': self.stepping_mode
        }

    def import_key(self, keydict):
        wl = keydict.get('wheel_lengths')
        pins = keydict.get('pins')
        lug_cage = keydict.get('lug_cage')
        positions = keydict.get('positions', [0]*6)
        stepping_mode = keydict.get('stepping_mode', 'carry')
        if not (wl and pins and lug_cage):
            raise ValueError("Key missing required fields")
        if len(wl) < 6:
            raise ValueError("wheel_lengths must have at least 6 values")
        self.L = wl[:6]
        self.nw = 6
        if len(pins) != 6:
            raise ValueError("pins must have exactly 6 lists")
        for i, p in enumerate(pins):
            if len(p) != self.L[i]:
                raise ValueError(f"pins[{i}] length mismatch")
        self.pins = [row[:] for row in pins]
        if len(lug_cage) != 27:
            raise ValueError("lug_cage must have exactly 27 Bars")
        for bar in lug_cage:
            if len(bar) != 6:
                raise ValueError("each Bar must have exactly 6 positions")
        self.lug_cage = [bar[:] for bar in lug_cage]
        if len(positions) != 6:
            raise ValueError("positions must have exactly 6 values")
        for i, p in enumerate(positions):
            if p < 0 or p >= self.L[i]:
                raise ValueError(f"position {i} out of range")
        self.positions = positions[:]
        self.stepping_mode = stepping_mode

# -----------------------
# GUI
# -----------------------
class C52GUI:
    def __init__(self, root):
        self.root = root
        root.title("C-52 Simulator (Python)")
        self.sim = C52Sim()

        # frames
        left = ttk.Frame(root, padding=8)
        left.grid(row=0, column=0, sticky="nswe")
        right = ttk.Frame(root, padding=8)
        right.grid(row=0, column=1, sticky="nswe")

        # Wheel lengths
        ttk.Label(left, text="Wheel lengths (comma separated, exactly 6):").grid(row=0, column=0, sticky="w")
        self.wheel_lengths_var = tk.StringVar(value=",".join(map(str, self.sim.L)))
        self.wl_entry = ttk.Entry(left, textvariable=self.wheel_lengths_var, width=30)
        self.wl_entry.grid(row=1, column=0, sticky="w")
        ttk.Button(left, text="Apply lengths", command=self.apply_wheel_lengths).grid(row=1, column=1, padx=4)

        # Pins editor
        ttk.Label(left, text="Pins for each wheel (0/1 strings). One wheel per line: wheel1:0101...").grid(row=2, column=0, sticky="w")
        self.pins_text = tk.Text(left, width=50, height=8)
        self.pins_text.grid(row=3, column=0, columnspan=2)
        ttk.Button(left, text="Load pins from sim", command=self.load_pins_to_text).grid(row=4, column=0, sticky="w", pady=4)
        ttk.Button(left, text="Apply pins", command=self.apply_pins_from_text).grid(row=4, column=1, sticky="e")

        # Lug editor (27 lines)
        ttk.Label(left, text="Lug cage (27 Bars, one Bar per line, 6 positions: 0=Lug present, 1=no Lug):").grid(row=5, column=0, sticky="w")
        self.lugs_text = tk.Text(left, width=50, height=14)
        self.lugs_text.grid(row=6, column=0, columnspan=2)
        ttk.Button(left, text="Load lugs from sim", command=self.load_lugs_to_text).grid(row=7, column=0, sticky="w", pady=4)
        ttk.Button(left, text="Apply lugs", command=self.apply_lugs_from_text).grid(row=7, column=1, sticky="e")

        # Positions
        ttk.Label(left, text="Initial positions (comma separated, exactly 6):").grid(row=8, column=0, sticky="w")
        self.pos_var = tk.StringVar(value=",".join(map(str, self.sim.positions)))
        self.pos_entry = ttk.Entry(left, textvariable=self.pos_var, width=30)
        self.pos_entry.grid(row=9, column=0, sticky="w")
        ttk.Button(left, text="Apply positions", command=self.apply_positions).grid(row=9, column=1, padx=4)

        # Stepping mode
        ttk.Label(left, text="Stepping mode:").grid(row=10, column=0, sticky="w")
        self.step_mode_var = tk.StringVar(value=self.sim.stepping_mode)
        ttk.OptionMenu(left, self.step_mode_var, self.sim.stepping_mode, 'carry', 'fixed').grid(row=10, column=1, sticky="w")

        # Key import/export and utilities
        ttk.Button(left, text="Save key to file", command=self.save_key_to_file).grid(row=11, column=0, pady=8)
        ttk.Button(left, text="Load key from file", command=self.load_key_from_file).grid(row=11, column=1, pady=8)
        ttk.Button(left, text="Randomize pins", command=self.randomize_pins).grid(row=12, column=0, pady=4)
        ttk.Button(left, text="Reset default key", command=self.reset_default_key).grid(row=12, column=1, pady=4)

        # Right: message/input and output
        ttk.Label(right, text="Message (input):").grid(row=0, column=0, sticky="w")
        self.msg_text = tk.Text(right, width=60, height=8)
        self.msg_text.grid(row=1, column=0, columnspan=3)

        ttk.Label(right, text="Output:").grid(row=2, column=0, sticky="w")
        self.out_text = tk.Text(right, width=60, height=8)
        self.out_text.grid(row=3, column=0, columnspan=3)

        # Counter label (shows how many letters were processed in last operation)
        self.count_var = tk.StringVar(value="Last processed letters: 0")
        ttk.Label(right, textvariable=self.count_var).grid(row=6, column=0, columnspan=3, sticky="w", pady=4)

        # Controls
        ttk.Button(right, text="Encrypt", command=self.encrypt_action).grid(row=4, column=0, pady=6)
        ttk.Button(right, text="Decrypt", command=self.decrypt_action).grid(row=4, column=1, pady=6)
        ttk.Button(right, text="Encrypt (show steps)", command=lambda: self.encrypt_action(show_steps=True)).grid(row=4, column=2, pady=6)

        ttk.Button(right, text="Show internal state", command=self.show_internal_state).grid(row=5, column=0, pady=6)
        ttk.Button(right, text="Clear output", command=lambda: self.out_text.delete("1.0", tk.END)).grid(row=5, column=1, pady=6)
        ttk.Button(right, text="Copy output", command=self.copy_output).grid(row=5, column=2, pady=6)

        # Load initial text fields
        self.load_pins_to_text()
        self.load_lugs_to_text()
        self.update_pos_entry()

    # ---------- GUI helper methods ----------
    def apply_wheel_lengths(self):
        s = self.wheel_lengths_var.get().strip()
        try:
            parts = [int(x.strip()) for x in s.split(",") if x.strip()!='']
            if len(parts) != 6:
                raise ValueError("Must have exactly 6 wheels")
            old = self.sim.export_key()
            new_pins = []
            for i, L in enumerate(parts):
                if i < len(old['pins']) and len(old['pins'][i]) == L:
                    new_pins.append(old['pins'][i][:])
                else:
                    new_pins.append([random.randint(0,1) for _ in range(L)])
            self.sim = C52Sim(wheel_lengths=parts, pins=new_pins, lug_cage=old['lug_cage'], positions=[0]*6, stepping_mode=self.step_mode_var.get())
            self.load_pins_to_text()
            self.load_lugs_to_text()
            self.update_pos_entry()
            messagebox.showinfo("OK", "Applied new wheel lengths.")
        except Exception as e:
            messagebox.showerror("Error", str(e))

    def load_pins_to_text(self):
        self.pins_text.delete("1.0", tk.END)
        for i, p in enumerate(self.sim.pins):
            s = "".join(str(bit) for bit in p)
            self.pins_text.insert(tk.END, f"wheel{i+1}:{s}\n")

    def apply_pins_from_text(self):
        txt = self.pins_text.get("1.0", tk.END).strip().splitlines()
        new_pins = []
        try:
            for line in txt:
                if not line.strip(): continue
                if ':' in line:
                    _, bits = line.split(":",1)
                else:
                    bits = line.strip()
                arr = [1 if ch=='1' else 0 for ch in bits if ch in '01']
                new_pins.append(arr)
            if len(new_pins) != 6:
                raise ValueError("Must have exactly 6 wheels")
            for i, arr in enumerate(new_pins):
                if len(arr) != self.sim.L[i]:
                    raise ValueError(f"Wheel {i+1} length mismatch")
            self.sim.pins = new_pins
            messagebox.showinfo("OK", "Pins applied.")
        except Exception as e:
            messagebox.showerror("Error", str(e))

    def load_lugs_to_text(self):
        self.lugs_text.delete("1.0", tk.END)
        for bar in self.sim.lug_cage:
            self.lugs_text.insert(tk.END, ",".join(str(x) for x in bar) + "\n")

    def apply_lugs_from_text(self):
        txt = self.lugs_text.get("1.0", tk.END).strip().splitlines()
        new_lugs = []
        try:
            for line in txt:
                if not line.strip(): continue
                parts = [int(p.strip()) for p in line.split(",") if p.strip()!='']
                if len(parts) != 6:
                    raise ValueError("Each Bar must have exactly 6 positions (0 or 1)")
                for v in parts:
                    if v not in (0,1):
                        raise ValueError("Lug flags must be 0 or 1")
                new_lugs.append(parts)
            if len(new_lugs) != 27:
                raise ValueError("Must have exactly 27 Bars")
            self.sim.lug_cage = new_lugs
            messagebox.showinfo("OK", "Lugs applied.")
        except Exception as e:
            messagebox.showerror("Error applying lugs", str(e))

    def apply_positions(self):
        s = self.pos_var.get().strip()
        try:
            parts = [int(x.strip()) for x in s.split(",") if x.strip()!='']
            if len(parts) != 6:
                raise ValueError("Must have exactly 6 positions")
            for i, p in enumerate(parts):
                if p < 0 or p >= self.sim.L[i]:
                    raise ValueError(f"Wheel {i+1} position out of range")
            self.sim.positions = parts[:]
            messagebox.showinfo("OK", "Positions applied.")
        except Exception as e:
            messagebox.showerror("Error", str(e))

    def update_pos_entry(self):
        self.pos_var.set(",".join(map(str,self.sim.positions)))
        self.wheel_lengths_var.set(",".join(map(str,self.sim.L)))
        self.step_mode_var.set(self.sim.stepping_mode)

    def save_key_to_file(self):
        key = self.sim.export_key()
        path = filedialog.asksaveasfilename(defaultextension=".json", filetypes=[("JSON files","*.json")])
        if not path: return
        with open(path,"w") as f:
            json.dump(key,f,indent=2)
        messagebox.showinfo("Saved", f"Key saved to {path}")

    def load_key_from_file(self):
        path = filedialog.askopenfilename(filetypes=[("JSON files","*.json")])
        if not path: return
        try:
            with open(path,"r") as f:
                key = json.load(f)
            self.sim.import_key(key)
            self.load_pins_to_text()
            self.load_lugs_to_text()
            self.update_pos_entry()
            messagebox.showinfo("Loaded", f"Key loaded from {path}")
        except Exception as e:
            messagebox.showerror("Error loading key", str(e))

    def randomize_pins(self):
        for i in range(6):
            self.sim.pins[i] = [random.randint(0,1) for _ in range(self.sim.L[i])]
        self.load_pins_to_text()
        messagebox.showinfo("OK", "Pins randomized.")

    def reset_default_key(self):
        self.sim = C52Sim()
        self.load_pins_to_text()
        self.load_lugs_to_text()
        self.update_pos_entry()
        messagebox.showinfo("OK", "Default key restored.")

    # ---------- Actions ----------
    def encrypt_action(self, show_steps=False):
        text = self.msg_text.get("1.0", tk.END).strip()
        if not text:
            messagebox.showwarning("Empty", "Please enter a message.")
            return
        # count letters to be processed (A-Z)
        count_letters = sum(1 for ch in text if ch.upper() in ALPHABET)
        self.sim.stepping_mode = self.step_mode_var.get()
        try:
            sim_copy = C52Sim(wheel_lengths=self.sim.L, pins=self.sim.pins, lug_cage=self.sim.lug_cage,
                               positions=self.sim.positions[:], stepping_mode=self.sim.stepping_mode)
            cipher, steps = sim_copy.process_text(text, encrypt=True, show_steps=show_steps)
            self.out_text.delete("1.0", tk.END)
            self.out_text.insert(tk.END, cipher + "\n")
            if show_steps:
                self.out_text.insert(tk.END, "\n--- Steps ---\n")
                for s in steps:
                    if s.get('skipped'):
                        self.out_text.insert(tk.END, f"(skipped) {s['char']}\n")
                    else:
                        self.out_text.insert(tk.END, f"{s['char']} -> P={s['P']} K={s['K']} -> C={s['C']}({s['Cchar']}) | pos_before={s['positions_before']} pos_after={s['positions_after']}\n")
            # update counter label for encryption
            self.count_var.set(f"Encrypted letters: {count_letters}")
            if messagebox.askyesno("Update positions?", "Update wheel positions after encryption?"):
                self.sim.positions = sim_copy.positions[:]
                self.update_pos_entry()
        except Exception as e:
            messagebox.showerror("Error", str(e))

    def decrypt_action(self):
        text = self.msg_text.get("1.0", tk.END).strip()
        if not text:
            messagebox.showwarning("Empty", "Enter cipher text.")
            return
        # count letters to be processed (A-Z)
        count_letters = sum(1 for ch in text if ch.upper() in ALPHABET)
        self.sim.stepping_mode = self.step_mode_var.get()
        try:
            sim_copy = C52Sim(wheel_lengths=self.sim.L, pins=self.sim.pins, lug_cage=self.sim.lug_cage,
                               positions=self.sim.positions[:], stepping_mode=self.sim.stepping_mode)
            plain, steps = sim_copy.process_text(text, encrypt=False, show_steps=True)
            self.out_text.delete("1.0", tk.END)
            self.out_text.insert(tk.END, plain + "\n\n--- Steps ---\n")
            for s in steps:
                if s.get('skipped'):
                    self.out_text.insert(tk.END, f"(skipped) {s['char']}\n")
                else:
                    self.out_text.insert(tk.END, f"{s['char']} -> C={s['P']} K={s['K']} -> P={s['C']}({s['Cchar']}) | pos_before={s['positions_before']} pos_after={s['positions_after']}\n")
            # update counter label for decryption
            self.count_var.set(f"Decrypted letters: {count_letters}")
            if messagebox.askyesno("Update positions?", "Update wheel positions after decryption?"):
                self.sim.positions = sim_copy.positions[:]
                self.update_pos_entry()
        except Exception as e:
            messagebox.showerror("Error", str(e))

    def show_internal_state(self):
        s = f"Wheel lengths: {self.sim.L}\nPositions: {self.sim.positions}\nStepping mode: {self.sim.stepping_mode}\n\nPins (first 200 chars each):\n"
        for i, p in enumerate(self.sim.pins):
            s += f"wheel{i+1} len={len(p)}: {''.join(str(x) for x in p[:200])}\n"
        s += "\nLugs (27 Bars × 6 Positions):\n"
        for i, bar in enumerate(self.sim.lug_cage):
            s += f"Bar{i+1}: {bar}\n"
        messagebox.showinfo("Internal state", s[:10000])

    def copy_output(self):
        txt = self.out_text.get("1.0", tk.END)
        self.root.clipboard_clear()
        self.root.clipboard_append(txt)
        messagebox.showinfo("Copied", "Output copied to clipboard.")

# -----------------------
# Run the GUI
# -----------------------
if __name__ == "__main__":
    root = tk.Tk()
    app = C52GUI(root)
    root.mainloop()
