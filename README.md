    import tkinter as tk
    from tkinter import ttk, filedialog, messagebox
    import pygame
    import threading
    import socket
    import json
    import random
    import time
    from PIL import Image, ImageTk, ImageDraw
    import speech_recognition as sr
    import requests
    import io
    import os
    import pickle
    import re

    class DnDGame:
        def __init__(self):
            self.root = tk.Tk()
            self.root.title("D&D 4e - ИИ Мастер")
            self.root.geometry("1400x900")
            self.root.configure(bg='#1a1a2e')
            
            # API configuration
            self.api_key = "07e1b8ed910f4dfbb4790aaa78b7ee88.xuAOgdGLYgbnNoaw"
            self.api_url = "https://open.bigmodel.cn/api/paas/v4/chat/completions"
            
            # Game state
            self.nickname = ""
            self.avatar_path = ""
            self.character_sheet = {}
            self.is_host = False
            self.socket = None
            self.connected_players = []
            self.game_state = "login"
            self.recognizer = sr.Recognizer()
            self.microphone = None
            self.is_recording = False
            self.chat_history = []
            self.dm_thinking = False
            self.character_approved = False
            self.save_file = "character_save.dat"
            self.open_windows = {}
            self.settings_window = None
            self.secret_menu_open = False
            
            # Audio settings
            self.input_device = None
            self.output_device = None
            
            # D&D 4e skills list
            self.dnd_4e_skills = [
                "Акробатика", "Магия", "Блефф", "Дипломатия", "Подземелья",
                "Выносливость", "Лечение", "История", "Запугивание", "Интуиция",
                "Природа", "Восприятие", "Религия", "Скрытность", "Воровство",
                "Атлетика", "Уличные знания"
            ]
            
            # Load cached master avatar
            self.master_avatar_path = os.path.join(os.path.dirname(__file__), "Master.png")
            self.dm_photo = None
            
            # Bind keyboard events
            self.root.bind('<KeyPress>', self.on_key_press)
            self.root.focus_set()
            
            # Save character on close
            self.root.protocol("WM_DELETE_WINDOW", self.on_closing)
            
            self.load_character()
            
            self.setup_login_screen()
        
        def on_key_press(self, event):
            # Handle Shift+P for secret menu
            if event.state & 0x1 and event.keysym == 'P':  # Shift+P
                self.open_secret_menu()
        
        def on_closing(self):
            # Save character before closing
            self.save_character()
            self.root.destroy()
        
        def open_secret_menu(self):
            if self.secret_menu_open:
                return
                
            self.secret_menu_open = True
            
            # Password dialog
            password_window = tk.Toplevel(self.root)
            password_window.title("Доступ")
            password_window.geometry("300x150")
            password_window.configure(bg='#16537e')
            password_window.transient(self.root)
            password_window.grab_set()
            
            def on_password_close():
                self.secret_menu_open = False
                password_window.destroy()
            
            password_window.protocol("WM_DELETE_WINDOW", on_password_close)
            
            tk.Label(password_window, text="Введите пароль:", 
                    font=('Arial', 12), fg='#ffffff', bg='#16537e').pack(pady=20)
            
            password_entry = tk.Entry(password_window, font=('Arial', 12), show='*')
            password_entry.pack(pady=10)
            password_entry.focus()
            
            def check_password():
                if password_entry.get() == "cxzcxzcxz":
                    password_window.destroy()
                    self.show_master_menu()
                else:
                    self.secret_menu_open = False
                    password_window.destroy()
            
            password_entry.bind('<Return>', lambda e: check_password())
            
            tk.Button(password_window, text="Войти", command=check_password,
                    bg='#27ae60', fg='white', font=('Arial', 12)).pack(pady=10)
        
        def show_master_menu(self):
            master_window = tk.Toplevel(self.root)
            master_window.title("Меню мастера")
            master_window.geometry("400x500")
            master_window.configure(bg='#16537e')
            master_window.transient(self.root)
            
            def on_master_close():
                self.secret_menu_open = False
                master_window.destroy()
            
            master_window.protocol("WM_DELETE_WINDOW", on_master_close)
            
            tk.Label(master_window, text="Меню мастера", 
                    font=('Arial', 16, 'bold'), fg='#ffffff', bg='#16537e').pack(pady=20)
            
            # Approve character button
            approve_btn = tk.Button(master_window, text="Разрешить анкету", 
                                command=self.approve_character, bg='#27ae60', fg='white',
                                font=('Arial', 12), width=25, pady=5)
            approve_btn.pack(pady=10)
            
            # Random event button
            event_btn = tk.Button(master_window, text="Сделать открытую крутку на событие", 
                                command=self.trigger_random_event, bg='#f39c12', fg='white',
                                font=('Arial', 12), width=25, pady=5)
            event_btn.pack(pady=10)
            
            # Write as master button
            write_btn = tk.Button(master_window, text="Написать что-либо от мастера", 
                                command=self.write_as_master, bg='#9b59b6', fg='white',
                                font=('Arial', 12), width=25, pady=5)
            write_btn.pack(pady=10)
            
            # Edit character button
            edit_btn = tk.Button(master_window, text="Редактор любых анкет чужой и своей", 
                                command=self.edit_character, bg='#e74c3c', fg='white',
                                font=('Arial', 12), width=25, pady=5)
            edit_btn.pack(pady=10)
            
            # Start game button
            start_btn = tk.Button(master_window, text="Начать игру", 
                                command=self.start_game_as_master, bg='#3498db', fg='white',
                                font=('Arial', 12), width=25, pady=5)
            start_btn.pack(pady=10)
            
            # End game button
            end_btn = tk.Button(master_window, text="Завершить игру", 
                            command=self.end_game, bg='#95a5a6', fg='white',
                            font=('Arial', 12), width=25, pady=5)
            end_btn.pack(pady=10)
        
        def approve_character(self):
            self.character_approved = True
            self.save_character()
            messagebox.showinfo("Успех", "Анкета одобрена!")
        
        def trigger_random_event(self):
            self.dm_random_event()
        
        def write_as_master(self):
            write_window = tk.Toplevel(self.root)
            write_window.title("Написать от мастера")
            write_window.geometry("500x400")
            write_window.configure(bg='#16537e')
            
            tk.Label(write_window, text="Сообщение от мастера:", 
                    font=('Arial', 14, 'bold'), fg='#ffffff', bg='#16537e').pack(pady=20)
            
            text_widget = tk.Text(write_window, font=('Arial', 12), height=15, width=50)
            text_widget.pack(pady=20, padx=20, fill='both', expand=True)
            
            def send_master_message():
                message = text_widget.get("1.0", tk.END).strip()
                if message:
                    self.chat_history.append({
                        'sender': 'dm',
                        'message': message,
                        'timestamp': time.strftime("%H:%M")
                    })
                    self.refresh_chat_display()
                    write_window.destroy()
            
            tk.Button(write_window, text="Отправить", command=send_master_message,
                    bg='#27ae60', fg='white', font=('Arial', 12)).pack(pady=10)
        
        def edit_character(self):
            # Open character creation screen for editing
            self.setup_character_creation()
        
        def start_game_as_master(self):
            if self.character_approved:
                self.game_state = "playing"
                self.setup_game_screen()
                self.start_adventure()
            else:
                messagebox.showwarning("Предупреждение", "Сначала одобрите анкету персонажа!")
        
        def end_game(self):
            end_message = "Хорошая игра, но пора уже заканчивать. Спасибо за участие!"
            self.chat_history.append({
                'sender': 'dm',
                'message': end_message,
                'timestamp': time.strftime("%H:%M")
            })
            self.refresh_chat_display()
            messagebox.showinfo("Игра завершена", "Игра завершена мастером.")
            
        def setup_login_screen(self):
            self.clear_screen()
            
            bg_frame = tk.Frame(self.root, bg='#0f3460')
            bg_frame.pack(fill='both', expand=True)
            
            login_frame = tk.Frame(bg_frame, bg='#16537e', relief='raised', bd=3)
            login_frame.place(relx=0.5, rely=0.5, anchor='center', width=500, height=600)
            
            title_label = tk.Label(login_frame, text="D&D 4-я редакция - ИИ Мастер Подземелий", 
                                font=('Times New Roman', 24, 'bold'), fg='#ffffff', bg='#16537e')
            title_label.pack(pady=30)
            
            tk.Label(login_frame, text="Никнейм (макс. 7 символов):", 
                    font=('Arial', 14), fg='#ffffff', bg='#16537e').pack(pady=10)
            
            self.nickname_entry = tk.Entry(login_frame, font=('Arial', 14), width=25)
            self.nickname_entry.pack(pady=5)
            self.nickname_entry.bind('<KeyRelease>', self.validate_nickname)
            
            tk.Label(login_frame, text="Аватар (300x300 пикселей):", 
                    font=('Arial', 14), fg='#ffffff', bg='#16537e').pack(pady=(30,10))
            
            self.avatar_label = tk.Label(login_frame, text="Аватар не выбран", 
                                    font=('Arial', 12), fg='#cccccc', bg='#16537e')
            self.avatar_label.pack(pady=5)
            
            avatar_btn = tk.Button(login_frame, text="Выбрать аватар", 
                                command=self.choose_avatar, bg='#3498db', fg='white',
                                font=('Arial', 12), relief='flat', pady=5)
            avatar_btn.pack(pady=10)
            
            self.login_btn = tk.Button(login_frame, text="Войти в игру", 
                                    command=self.enter_character_creation, 
                                    bg='#27ae60', fg='white', font=('Arial', 14, 'bold'),
                                    state='disabled', relief='flat', pady=8)
            self.login_btn.pack(pady=30)
            
            # Check for saved characters and show load buttons
            if os.path.exists(self.save_file):
                try:
                    with open(self.save_file, 'rb') as f:
                        save_data = pickle.load(f)
                        saved_name = save_data.get('character_sheet', {}).get('Имя персонажа', 'Неизвестный')
                        
                    load_btn = tk.Button(login_frame, text=f"Сыграть за {saved_name}", 
                                    command=self.load_and_enter_game, 
                                    bg='#9b59b6', fg='white', font=('Arial', 12),
                                    relief='flat', pady=5)
                    load_btn.pack(pady=10)
                except:
                    pass
            
            # Settings button
            settings_btn = tk.Button(login_frame, text="Настройки", 
                                command=self.show_settings, bg='#34495e', fg='white',
                                font=('Arial', 12), relief='flat', pady=5)
            settings_btn.pack(pady=10)
            
        def show_settings(self):
            if self.settings_window:
                self.settings_window.lift()
                return
                
            self.settings_window = tk.Toplevel(self.root)
            self.settings_window.title("Настройки")
            self.settings_window.geometry("400x300")
            self.settings_window.configure(bg='#16537e')
            
            def on_settings_close():
                self.settings_window = None
            
            self.settings_window.protocol("WM_DELETE_WINDOW", on_settings_close)
            
            tk.Label(self.settings_window, text="Настройки", 
                    font=('Arial', 16, 'bold'), fg='#ffffff', bg='#16537e').pack(pady=20)
            
            # Audio input device
            tk.Label(self.settings_window, text="Устройство ввода (микрофон):", 
                    font=('Arial', 12), fg='#ffffff', bg='#16537e').pack(pady=10)
            
            input_var = tk.StringVar(value="По умолчанию")
            input_combo = ttk.Combobox(self.settings_window, textvariable=input_var, 
                                    values=["По умолчанию", "Микрофон 1", "Микрофон 2"])
            input_combo.pack(pady=5)
            
            # Audio output device
            tk.Label(self.settings_window, text="Устройство вывода (динамики):", 
                    font=('Arial', 12), fg='#ffffff', bg='#16537e').pack(pady=10)
            
            output_var = tk.StringVar(value="По умолчанию")
            output_combo = ttk.Combobox(self.settings_window, textvariable=output_var, 
                                    values=["По умолчанию", "Динамики 1", "Динамики 2"])
            output_combo.pack(pady=5)
            
            def save_settings():
                self.input_device = input_var.get()
                self.output_device = output_var.get()
                messagebox.showinfo("Настройки", "Настройки сохранены!")
                
            tk.Button(self.settings_window, text="Сохранить", command=save_settings,
                    bg='#27ae60', fg='white', font=('Arial', 12)).pack(pady=20)
            
        def validate_nickname(self, event):
            text = self.nickname_entry.get()
            if len(text) > 7:
                self.nickname_entry.delete(7, tk.END)
            
            self.nickname = self.nickname_entry.get()
            self.update_login_button()
            
        def choose_avatar(self):
            file_path = filedialog.askopenfilename(
                title="Выберите аватар",
                filetypes=[("Изображения", "*.png *.jpg *.jpeg *.gif *.bmp")]
            )
            
            if file_path:
                try:
                    # Validate image size
                    img = Image.open(file_path)
                    if img.size != (300, 300):
                        result = messagebox.askyesno("Размер изображения", 
                                                    "Рекомендуемый размер аватара 300x300 пикселей. Продолжить?")
                        if not result:
                            return
                    
                    img = img.resize((300, 300), Image.Resampling.LANCZOS)
                    
                    # Create circular mask
                    mask = Image.new('L', (300, 300), 0)
                    draw = ImageDraw.Draw(mask)
                    draw.ellipse((0, 0, 300, 300), fill=255)
                    
                    output = Image.new('RGBA', (300, 300), (0, 0, 0, 0))
                    output.paste(img, (0, 0))
                    output.putalpha(mask)
                    
                    # Save temporary avatar
                    temp_path = os.path.join(os.path.dirname(__file__), "temp_avatar.png")
                    output.save(temp_path)
                    
                    self.avatar_path = temp_path
                    self.avatar_label.config(text="Аватар выбран")
                    self.update_login_button()
                    
                except Exception as e:
                    messagebox.showerror("Ошибка", f"Не удалось обработать аватар: {str(e)}")
                    
        def update_login_button(self):
            if self.nickname and len(self.nickname) <= 7 and self.avatar_path:
                self.login_btn.config(state='normal')
            else:
                self.login_btn.config(state='disabled')
                
        def enter_character_creation(self):
            self.game_state = "character_creation"
            self.setup_character_creation()
            
        def load_and_enter_game(self):
            try:
                with open(self.save_file, 'rb') as f:
                    save_data = pickle.load(f)
                    self.nickname = save_data.get('nickname', '')
                    self.avatar_path = save_data.get('avatar_path', '')
                    self.character_sheet = save_data.get('character_sheet', {})
                    self.character_approved = save_data.get('character_approved', False)
                    
                if self.character_approved:
                    self.game_state = "lobby"
                    self.setup_lobby_screen()
                else:
                    messagebox.showwarning("Предупреждение", "Персонаж не одобрен мастером")
                    self.game_state = "character_creation"
                    self.setup_character_creation()
                    
            except Exception as e:
                messagebox.showerror("Ошибка", f"Не удалось загрузить персонажа: {str(e)}")
        
        def save_character(self):
            try:
                save_data = {
                    'nickname': self.nickname,
                    'avatar_path': self.avatar_path,
                    'character_sheet': self.character_sheet,
                    'character_approved': self.character_approved,
                    'chat_history': self.chat_history
                }
                with open(self.save_file, 'wb') as f:
                    pickle.dump(save_data, f)
            except Exception as e:
                print(f"Ошибка сохранения: {e}")
        
        def load_character(self):
            try:
                if os.path.exists(self.save_file):
                    with open(self.save_file, 'rb') as f:
                        save_data = pickle.load(f)
                        self.nickname = save_data.get('nickname', '')
                        self.avatar_path = save_data.get('avatar_path', '')
                        self.character_sheet = save_data.get('character_sheet', {})
                        self.character_approved = save_data.get('character_approved', False)
                        self.chat_history = save_data.get('chat_history', [])
            except Exception as e:
                print(f"Ошибка загрузки: {e}")
                
        def setup_character_creation(self):
            self.clear_screen()
            
            main_frame = tk.Frame(self.root, bg='#0f3460')
            main_frame.pack(fill='both', expand=True)
            
            char_frame = tk.Frame(main_frame, bg='#16537e', relief='raised', bd=3)
            char_frame.pack(fill='both', expand=True, padx=30, pady=20)
            
            title_label = tk.Label(char_frame, text="Создание персонажа - D&D 4-я редакция", 
                                font=('Times New Roman', 20, 'bold'), fg='#ffffff', bg='#16537e')
            title_label.pack(pady=20)
            
            warning_label = tk.Label(char_frame, 
                                text="Стандартные характеристики: 20. Максимальные навыки: 4. Кастомные навыки требуют описания.",
                                font=('Arial', 12), fg='#ffcc00', bg='#16537e')
            warning_label.pack(pady=10)
            
            scroll_frame = tk.Frame(char_frame, bg='#16537e')
            scroll_frame.pack(fill='both', expand=True, padx=10, pady=10)
            
            canvas = tk.Canvas(scroll_frame, bg='#16537e', highlightthickness=0)
            scrollbar = ttk.Scrollbar(scroll_frame, orient="vertical", command=canvas.yview)
            scrollable_frame = tk.Frame(canvas, bg='#16537e')
            
            scrollable_frame.bind(
                "<Configure>",
                lambda e: canvas.configure(scrollregion=canvas.bbox("all"))
            )
            
            canvas.create_window((0, 0), window=scrollable_frame, anchor="nw")
            canvas.configure(yscrollcommand=scrollbar.set)
            
            canvas.pack(side="left", fill="both", expand=True)
            scrollbar.pack(side="right", fill="y")
            
            fields = [
                "Имя персонажа", "Раса", "Класс", "Предыстория", "Мировоззрение",
                "Сила", "Телосложение", "Ловкость", "Интеллект", "Мудрость", "Харизма",
                "Класс доспехов", "Очки здоровья", "Скорость", "Бонус мастерства",
                "Навыки", "Языки", "Снаряжение", "Особенности и черты", "Предыстория персонажа"
            ]
            
            self.char_entries = {}
            
            for field in fields:
                row_frame = tk.Frame(scrollable_frame, bg='#16537e')
                row_frame.pack(fill='x', pady=8, padx=20)
                
                label = tk.Label(row_frame, text=f"{field}:", 
                            font=('Arial', 12, 'bold'), fg='#ffffff', bg='#16537e', 
                            width=25, anchor='w')
                label.pack(side='left')
                
                if field in ["Особенности и черты", "Предыстория персонажа", "Снаряжение", "Навыки"]:
                    text_widget = tk.Text(row_frame, font=('Arial', 11), height=5, width=40)
                    text_widget.pack(side='right', fill='x', expand=True)
                    self.char_entries[field] = text_widget
                    text_widget.bind('<KeyRelease>', lambda e, f=field: self.validate_text_length(f, 500))
                    
                    # Load existing data if editing
                    if field in self.character_sheet:
                        text_widget.insert('1.0', self.character_sheet[field])
                else:
                    entry = tk.Entry(row_frame, font=('Arial', 11), width=35)
                    entry.pack(side='right', fill='x', expand=True)
                    self.char_entries[field] = entry
                    entry.bind('<KeyRelease>', lambda e, f=field: self.validate_entry_length(f, 50))
                    
                    # Load existing data if editing
                    if field in self.character_sheet:
                        entry.insert(0, self.character_sheet[field])
                    elif field in ["Сила", "Телосложение", "Ловкость", "Интеллект", "Мудрость", "Харизма"]:
                        entry.insert(0, "20")
            
            button_frame = tk.Frame(char_frame, bg='#16537e')
            button_frame.pack(pady=20)
            
            submit_btn = tk.Button(button_frame, text="Отправить мастеру на проверку", 
                                command=self.submit_character, bg='#e74c3c', fg='white', 
                                font=('Arial', 14, 'bold'), relief='flat', padx=20, pady=10)
            submit_btn.pack()
            
        def validate_text_length(self, field, max_length):
            widget = self.char_entries[field]
            content = widget.get("1.0", tk.END)
            if len(content) > max_length:
                widget.delete(f"1.{max_length}", tk.END)
                
        def validate_entry_length(self, field, max_length):
            entry = self.char_entries[field]
            content = entry.get()
            if len(content) > max_length:
                entry.delete(max_length, tk.END)
                
        def submit_character(self):
            self.character_sheet = {}
            for field, widget in self.char_entries.items():
                if isinstance(widget, tk.Text):
                    self.character_sheet[field] = widget.get("1.0", tk.END).strip()
                else:
                    self.character_sheet[field] = widget.get().strip()
            
            required_fields = ["Имя персонажа", "Раса", "Класс"]
            for field in required_fields:
                if not self.character_sheet.get(field):
                    messagebox.showerror("Ошибка", f"Поле '{field}' обязательно для заполнения!")
                    return
            
            try:
                self.character_sheet["Очки здоровья"] = int(self.character_sheet.get("Очки здоровья", "20"))
                self.character_sheet["Максимальные очки здоровья"] = self.character_sheet["Очки здоровья"]
            except:
                self.character_sheet["Очки здоровья"] = 20
                self.character_sheet["Максимальные очки здоровья"] = 20
            
            self.character_approved = False
            self.check_character_with_dm()
            
        def check_character_with_dm(self):
            if self.dm_thinking:
                return
                
            self.dm_thinking = True
            
            progress_window = tk.Toplevel(self.root)
            progress_window.title("Проверка персонажа")
            progress_window.geometry("400x200")
            progress_window.configure(bg='#16537e')
            progress_window.transient(self.root)
            progress_window.grab_set()
            
            tk.Label(progress_window, text="Мастер проверяет вашего персонажа...", 
                    font=('Arial', 14), fg='#ffffff', bg='#16537e').pack(pady=40)
            
            progress = ttk.Progressbar(progress_window, mode='indeterminate')
            progress.pack(pady=20)
            progress.start()
            
            def check_in_background():
                try:
                    headers = {
                        "Authorization": f"Bearer {self.api_key}",
                        "Content-Type": "application/json"
                    }
                    
                    char_info = ""
                    for field, value in self.character_sheet.items():
                        char_info += f"{field}: {value}\n"
                    
                    system_prompt = f"""Ты строгий мастер D&D 4-й редакции. Проверь анкету персонажа.

    Правила проверки:
    1. Базовые характеристики (Сила, Телосложение, Ловкость, Интеллект, Мудрость, Харизма) должны быть 20 или меньше
    2. Навыки не должны превышать 4 уровень
    3. Любые кастомные навыки должны быть описаны
    4. Класс и раса должны соответствовать D&D 4e
    5. Снаряжение должно быть разумным для начального персонажа

    Стандартные навыки D&D 4e: Акробатика, Магия, Блефф, Дипломатия, Подземелья, Выносливость, Лечение, История, Запугивание, Интуиция, Природа, Восприятие, Религия, Скрытность, Воровство, Атлетика, Уличные знания.

    Если персонаж соответствует правилам - ответь "ОДОБРЕН".
    Если есть нарушения - объясни что нужно исправить.

    Анкета персонажа:
    {char_info}"""
                    
                    data = {
                        "model": "glm-4",
                        "messages": [
                            {"role": "system", "content": system_prompt},
                            {"role": "user", "content": "Проверь анкету персонажа"}
                        ],
                        "temperature": 0.3,
                        "max_tokens": 300
                    }
                    
                    response = requests.post(self.api_url, headers=headers, json=data, timeout=30)
                    if response.status_code == 200:
                        result = response.json()
                        dm_response = result['choices'][0]['message']['content']
                        
                        self.root.after(0, lambda: self.handle_character_check_result(dm_response, progress_window))
                    else:
                        self.root.after(0, lambda: self.handle_dm_error(progress_window, "Ошибка API"))
                except requests.exceptions.Timeout:
                    self.root.after(0, lambda: self.handle_dm_error(progress_window, "Тайм-аут подключения"))
                except requests.exceptions.ConnectionError:
                    self.root.after(0, lambda: self.handle_dm_error(progress_window, "Нет подключения к интернету"))
                except Exception as e:
                    self.root.after(0, lambda: self.handle_dm_error(progress_window, f"Неизвестная ошибка: {str(e)}"))
            
            threading.Thread(target=check_in_background, daemon=True).start()
            
        def handle_character_check_result(self, response, progress_window):
            self.dm_thinking = False
            progress_window.destroy()
            
            if "ОДОБРЕН" in response.upper():
                self.character_approved = True
                self.save_character()
                
                # Show response in the same window/context where character was submitted
                result_window = tk.Toplevel(self.root)
                result_window.title("Результат проверки")
                result_window.geometry("500x300")
                result_window.configure(bg='#16537e')
                result_window.transient(self.root)
                
                tk.Label(result_window, text="Персонаж одобрен!", 
                        font=('Arial', 16, 'bold'), fg='#27ae60', bg='#16537e').pack(pady=20)
                
                tk.Label(result_window, text=response, font=('Arial', 12), 
                        fg='#ffffff', bg='#16537e', wraplength=400).pack(pady=20, padx=20)
                
                def continue_to_game():
                    result_window.destroy()
                    self.game_state = "playing"
                    self.setup_game_screen()
                    self.start_adventure()
                
                tk.Button(result_window, text="Продолжить в игру", command=continue_to_game,
                        bg='#27ae60', fg='white', font=('Arial', 14, 'bold')).pack(pady=20)
            else:
                # Show rejection in the same context
                result_window = tk.Toplevel(self.root)
                result_window.title("Результат проверки")
                result_window.geometry("500x400")
                result_window.configure(bg='#16537e')
                result_window.transient(self.root)
                
                tk.Label(result_window, text="Персонаж отклонен", 
                        font=('Arial', 16, 'bold'), fg='#e74c3c', bg='#16537e').pack(pady=20)
                
                tk.Label(result_window, text="Замечания мастера:", 
                        font=('Arial', 14, 'bold'), fg='#ffffff', bg='#16537e').pack(pady=10)
                
                text_widget = tk.Text(result_window, font=('Arial', 11), height=10, width=50,
                                    bg='#0f3460', fg='#ffffff', wrap='word')
                text_widget.pack(pady=10, padx=20, fill='both', expand=True)
                text_widget.insert('1.0', response)
                text_widget.config(state='disabled')
                
                def continue_editing():
                    result_window.destroy()
                    # Allow player to continue editing in the same character creation window
                    # The character creation window should remain open
                
                tk.Button(result_window, text="Исправить анкету", command=continue_editing,
                        bg='#f39c12', fg='white', font=('Arial', 12, 'bold')).pack(pady=20)
                
        def setup_game_screen(self):
            self.clear_screen()
            
            main_frame = tk.Frame(self.root, bg='#0f0f23')
            main_frame.pack(fill='both', expand=True)
            
            # Create master avatar area (fixed the overlapping avatar issue)
            master_frame = tk.Frame(main_frame, bg='#1e3a5f', relief='raised', bd=2)
            master_frame.place(x=50, y=50, width=200, height=250)
            
            tk.Label(master_frame, text="Мастер", font=('Arial', 14, 'bold'), 
                    fg='#ffffff', bg='#1e3a5f').pack(pady=10)
            
            # Master avatar (removed duplicate/overlapping avatar)
            avatar_frame = tk.Frame(master_frame, bg='#1e3a5f')
            avatar_frame.pack(pady=10)
            
            if os.path.exists(self.master_avatar_path):
                try:
                    master_img = Image.open(self.master_avatar_path)
                    master_img = master_img.resize((120, 120), Image.Resampling.LANCZOS)
                    self.dm_photo = ImageTk.PhotoImage(master_img)
                    avatar_label = tk.Label(avatar_frame, image=self.dm_photo, bg='#1e3a5f')
                    avatar_label.pack()
                except:
                    # Fallback if master image not found
                    tk.Label(avatar_frame, text="DM", font=('Arial', 24, 'bold'), 
                            fg='#ffffff', bg='#8b4513', width=8, height=4).pack()
            else:
                tk.Label(avatar_frame, text="DM", font=('Arial', 24, 'bold'), 
                        fg='#ffffff', bg='#8b4513', width=8, height=4).pack()
            
            # Player frames
            self.player_frames = []
            self.setup_player_frames(main_frame)
            
            # Menu buttons
            self.setup_menu_buttons(main_frame)
            
        def setup_player_frames(self, parent):
            positions = [(50, 350), (50, 450), (50, 550), (50, 650)]
            
            for i, (x, y) in enumerate(positions):
                player_frame = tk.Frame(parent, bg='#1e3a5f', relief='raised', bd=2)
                player_frame.place(x=x, y=y, width=200, height=80)
                
                if i == 0 and self.nickname:  # Current player
                    tk.Label(player_frame, text="Вы", font=('Arial', 12, 'bold'), 
                            fg='#ffffff', bg='#1e3a5f').pack()
                    
                    # Player avatar
                    if self.avatar_path and os.path.exists(self.avatar_path):
                        try:
                            player_img = Image.open(self.avatar_path)
                            player_img = player_img.resize((40, 40), Image.Resampling.LANCZOS)
                            player_photo = ImageTk.PhotoImage(player_img)
                            player_label = tk.Label(player_frame, image=player_photo, bg='#1e3a5f')
                            player_label.image = player_photo  # Keep reference
                            player_label.pack()
                        except:
                            player_label = tk.Label(player_frame, text=self.nickname[:2].upper(), 
                                                font=('Arial', 18, 'bold'), fg='#ffffff', bg='#3498db')
                            player_label.pack(expand=True)
                            
                    tk.Label(player_frame, text=self.nickname, font=('Arial', 10, 'bold'), 
                            fg='#ffffff', bg='#1e3a5f').pack()
                else:
                    empty_label = tk.Label(player_frame, text="Пусто", font=('Arial', 12), 
                                        fg='#888888', bg='#1e3a5f')
                    empty_label.pack(expand=True)
                    
                self.player_frames.append(player_frame)
                
        def setup_menu_buttons(self, parent):
            button_frame = tk.Frame(parent, bg='#0f0f23')
            button_frame.place(x=20, y=20)
            
            chat_btn = tk.Button(button_frame, text="Чат", font=('Arial', 14, 'bold'), 
                            command=self.show_chat_window, bg='#27ae60', fg='white',
                            width=8, height=2, relief='flat')
            chat_btn.grid(row=0, column=0, padx=5, pady=5)
            
            stats_btn = tk.Button(button_frame, text="Лист", font=('Arial', 14, 'bold'), 
                                command=self.show_character_stats, bg='#9b59b6', fg='white',
                                width=8, height=2, relief='flat')
            stats_btn.grid(row=0, column=1, padx=5, pady=5)
            
            question_btn = tk.Button(button_frame, text="Вопрос", font=('Arial', 14, 'bold'), 
                                command=self.show_dm_question, bg='#f39c12', fg='white',
                                width=8, height=2, relief='flat')
            question_btn.grid(row=0, column=2, padx=5, pady=5)
            
            # Settings button in game
            settings_btn = tk.Button(button_frame, text="Настройки", font=('Arial', 14, 'bold'), 
                                command=self.show_settings, bg='#34495e', fg='white',
                                width=8, height=2, relief='flat')
            settings_btn.grid(row=0, column=3, padx=5, pady=5)
            
        def show_chat_window(self):
            if 'chat' in self.open_windows:
                self.open_windows['chat'].lift()
                return
                
            self.chat_window = tk.Toplevel(self.root)
            self.chat_window.title("Чат с мастером")
            
            # Center the window on screen
            window_width = 700
            window_height = 800
            screen_width = self.root.winfo_screenwidth()
            screen_height = self.root.winfo_screenheight()
            x = (screen_width - window_width) // 2
            y = (screen_height - window_height) // 2
            
            self.chat_window.geometry(f"{window_width}x{window_height}+{x}+{y}")
            
            # Make window semi-transparent
            self.chat_window.attributes('-alpha', 0.9)
            
            # Configure background with border
            self.chat_window.configure(bg='#16537e')
            self.chat_window.resizable(True, True)
            
            # Add border frame
            border_frame = tk.Frame(self.chat_window, bg='#ffffff', relief='raised', bd=3)
            border_frame.pack(fill='both', expand=True, padx=5, pady=5)
            
            self.open_windows['chat'] = self.chat_window
            
            def on_chat_close():
                if 'chat' in self.open_windows:
                    del self.open_windows['chat']
                self.chat_window.destroy()
            
            self.chat_window.protocol("WM_DELETE_WINDOW", on_chat_close)
            
            chat_frame = tk.Frame(border_frame, bg='#16537e')
            chat_frame.pack(fill='both', expand=True, padx=10, pady=10)
            
            history_frame = tk.Frame(chat_frame, bg='#16537e')
            history_frame.pack(fill='both', expand=True, pady=(0, 10))
            
            self.chat_display = tk.Text(history_frame, font=('Arial', 11), bg='#0f3460', 
                                    fg='#ffffff', wrap='word', state='disabled', height=25)
            chat_scrollbar = ttk.Scrollbar(history_frame, orient="vertical", command=self.chat_display.yview)
            self.chat_display.configure(yscrollcommand=chat_scrollbar.set)
            
            self.chat_display.pack(side="left", fill="both", expand=True)
            chat_scrollbar.pack(side="right", fill="y")
            
            self.refresh_chat_display()
            
            input_frame = tk.Frame(chat_frame, bg='#16537e')
            input_frame.pack(fill='x', pady=5)
            
            # Voice chat button (more prominent)
            voice_btn = tk.Button(input_frame, text="🎤 Голосовой чат", font=('Arial', 12, 'bold'), 
                            command=self.toggle_recording, bg='#e74c3c', fg='white',
                            relief='flat', padx=15, pady=8)
            voice_btn.pack(side='left', padx=(0, 10))
            
            self.recording_status = tk.Label(input_frame, text="", font=('Arial', 10), 
                                        fg='#ffffff', bg='#16537e')
            self.recording_status.pack(side='left')
            
            message_frame = tk.Frame(chat_frame, bg='#16537e')
            message_frame.pack(fill='x', pady=5)
            
            tk.Label(message_frame, text="Сообщение мастеру:", font=('Arial', 12, 'bold'), 
                    fg='#ffffff', bg='#16537e').pack(anchor='w')
            
            self.chat_text = tk.Text(message_frame, font=('Arial', 11), height=4, bg='#ffffff', 
                                fg='#000000', wrap='word')
            self.chat_text.pack(fill='x', pady=5)
            self.chat_text.bind('<KeyRelease>', self.validate_chat_length)
            
            self.char_count_label = tk.Label(message_frame, text="0/400", font=('Arial', 10), 
                                        fg='#cccccc', bg='#16537e')
            self.char_count_label.pack(anchor='e')
            
            send_btn = tk.Button(message_frame, text="Отправить", 
                            command=self.send_message, bg='#27ae60', fg='white',
                            font=('Arial', 12, 'bold'), relief='flat', pady=5)
            send_btn.pack(pady=10)
            
        def refresh_chat_display(self):
            if hasattr(self, 'chat_display'):
                self.chat_display.config(state='normal')
                self.chat_display.delete('1.0', tk.END)
                
                # Configure tags for different message types
                self.chat_display.tag_config('player', foreground='#aaffaa')
                self.chat_display.tag_config('dm', foreground='#ffaa44')
                self.chat_display.tag_config('timestamp', foreground='#888888', font=('Arial', 9))
                
                for entry in self.chat_history:
                    sender = entry['sender']
                    message = entry['message']
                    timestamp = entry.get('timestamp', '')
                    
                    # Add timestamp
                    self.chat_display.insert(tk.END, f"[{timestamp}] ", 'timestamp')
                    
                    if sender == 'player':
                        self.chat_display.insert(tk.END, f"Вы: {message}\n\n", 'player')
                    else:
                        self.chat_display.insert(tk.END, f"Мастер: {message}\n\n", 'dm')
                
                self.chat_display.config(state='disabled')
                self.chat_display.see(tk.END)
        
        def send_message(self):
            if hasattr(self, 'chat_text'):
                message = self.chat_text.get("1.0", tk.END).strip()
                if message:
                    self.chat_history.append({
                        'sender': 'player',
                        'message': message,
                        'timestamp': time.strftime("%H:%M")
                    })
                    
                    self.chat_text.delete("1.0", tk.END)
                    self.char_count_label.config(text="0/400")
                    self.refresh_chat_display()
                    
                    # Get DM response
                    self.get_dm_response_streaming(message)
            
        def validate_chat_length(self, event=None):
            if hasattr(self, 'chat_text') and hasattr(self, 'char_count_label'):
                content = self.chat_text.get("1.0", tk.END)
                length = len(content) - 1
                
                if length > 400:
                    self.chat_text.delete("1.400", tk.END)
                    length = 400
                    
                self.char_count_label.config(text=f"{length}/400")
            
        def show_character_stats(self):
            if 'stats' in self.open_windows:
                self.open_windows['stats'].lift()
                return
                
            stats_window = tk.Toplevel(self.root)
            stats_window.title("Лист персонажа")
            stats_window.geometry("600x700")
            stats_window.configure(bg='#16537e')
            
            self.open_windows['stats'] = stats_window
            
            def on_stats_close():
                if 'stats' in self.open_windows:
                    del self.open_windows['stats']
                stats_window.destroy()
            
            stats_window.protocol("WM_DELETE_WINDOW", on_stats_close)
            
            canvas = tk.Canvas(stats_window, bg='#16537e', highlightthickness=0)
            scrollbar = ttk.Scrollbar(stats_window, orient="vertical", command=canvas.yview)
            scrollable_frame = tk.Frame(canvas, bg='#16537e')
            
            scrollable_frame.bind(
                "<Configure>",
                lambda e: canvas.configure(scrollregion=canvas.bbox("all"))
            )
            
            canvas.create_window((0, 0), window=scrollable_frame, anchor="nw")
            canvas.configure(yscrollcommand=scrollbar.set)
            
            canvas.pack(side="left", fill="both", expand=True)
            scrollbar.pack(side="right", fill="y")
            
            title_label = tk.Label(scrollable_frame, text="Лист персонажа", 
                                font=('Times New Roman', 18, 'bold'), fg='#ffffff', bg='#16537e')
            title_label.pack(pady=20)
            
            # Character image/avatar
            if self.avatar_path and os.path.exists(self.avatar_path):
                try:
                    avatar_img = Image.open(self.avatar_path)
                    avatar_img = avatar_img.resize((150, 150), Image.Resampling.LANCZOS)
                    avatar_photo = ImageTk.PhotoImage(avatar_img)
                    avatar_label = tk.Label(scrollable_frame, image=avatar_photo, bg='#16537e')
                    avatar_label.image = avatar_photo  # Keep a reference
                    avatar_label.pack(pady=10)
                except:
                    pass
            
            # Character name and basic info
            name_frame = tk.Frame(scrollable_frame, bg='#0f3460', relief='raised', bd=2)
            name_frame.pack(pady=10, padx=20, fill='x')
            
            char_name = self.character_sheet.get('Имя персонажа', 'Неизвестно')
            char_class = self.character_sheet.get('Класс', 'Неизвестно')
            char_race = self.character_sheet.get('Раса', 'Неизвестно')
            
            tk.Label(name_frame, text=f"{char_name}", font=('Arial', 16, 'bold'), 
                    fg='#ffffff', bg='#0f3460').pack(pady=5)
            tk.Label(name_frame, text=f"{char_race} {char_class}", font=('Arial', 12), 
                    fg='#cccccc', bg='#0f3460').pack(pady=5)
            
            # Health points
            hp_frame = tk.Frame(scrollable_frame, bg='#0f3460', relief='raised', bd=2)
            hp_frame.pack(pady=10, padx=20, fill='x')
            
            tk.Label(hp_frame, text="Очки здоровья:", font=('Arial', 14, 'bold'), 
                    fg='#ffffff', bg='#0f3460').pack(side='left', padx=10, pady=10)
            
            current_hp = self.character_sheet.get('Очки здоровья', 20)
            max_hp = self.character_sheet.get('Максимальные очки здоровья', 20)
            hp_text = f"{current_hp}/{max_hp}"
            tk.Label(hp_frame, text=hp_text, font=('Arial', 14, 'bold'), 
                    fg='#ff4444', bg='#0f3460').pack(side='right', padx=10, pady=10)
            
            # Character attributes
            attributes = ["Сила", "Телосложение", "Ловкость", "Интеллект", "Мудрость", "Харизма"]
            attr_frame = tk.Frame(scrollable_frame, bg='#0f3460', relief='raised', bd=2)
            attr_frame.pack(pady=10, padx=20, fill='x')
            
            tk.Label(attr_frame, text="Характеристики", font=('Arial', 14, 'bold'), 
                    fg='#ffffff', bg='#0f3460').pack(pady=10)
            
            attr_grid = tk.Frame(attr_frame, bg='#0f3460')
            attr_grid.pack(pady=10, padx=10, fill='x')
            
            for i, attr in enumerate(attributes):
                value = self.character_sheet.get(attr, "20")
                row = i % 3
                col = i // 3
                
                attr_label = tk.Label(attr_grid, text=f"{attr}: {value}", 
                                    font=('Arial', 11), fg='#ffffff', bg='#0f3460',
                                    width=15, anchor='w')
                attr_label.grid(row=row, column=col, padx=10, pady=5, sticky='w')
            
            # Other character details
            details = [
                "Класс доспехов", "Скорость", "Бонус мастерства", 
                "Мировоззрение", "Предыстория", "Языки"
            ]
            
            for field in details:
                if field in self.character_sheet and self.character_sheet[field]:
                    field_frame = tk.Frame(scrollable_frame, bg='#0f3460', relief='groove', bd=1)
                    field_frame.pack(pady=5, padx=20, fill='x')
                    
                    tk.Label(field_frame, text=f"{field}:", font=('Arial', 12, 'bold'), 
                            fg='#ffffff', bg='#0f3460', anchor='w', width=20).pack(side='left', padx=10, pady=5)
                    
                    tk.Label(field_frame, text=str(self.character_sheet[field]), font=('Arial', 11), 
                            fg='#cccccc', bg='#0f3460', anchor='w', wraplength=350).pack(side='left', fill='x', expand=True, padx=10, pady=5)
            
            # Skills, Equipment, Features (if they exist)
            long_fields = ["Навыки", "Снаряжение", "Особенности и черты", "Предыстория персонажа"]
            
            for field in long_fields:
                if field in self.character_sheet and self.character_sheet[field]:
                    field_frame = tk.Frame(scrollable_frame, bg='#0f3460', relief='groove', bd=1)
                    field_frame.pack(pady=5, padx=20, fill='x')
                    
                    tk.Label(field_frame, text=f"{field}:", font=('Arial', 12, 'bold'), 
                            fg='#ffffff', bg='#0f3460').pack(anchor='w', padx=10, pady=(10, 5))
                    
                    # Use a text widget for longer content
                    text_widget = tk.Text(field_frame, font=('Arial', 10), height=4, 
                                        bg='#0f3460', fg='#cccccc', wrap='word',
                                        relief='flat', borderwidth=0)
                    text_widget.insert('1.0', self.character_sheet[field])
                    text_widget.config(state='disabled')
                    text_widget.pack(fill='x', padx=10, pady=(0, 10))
                            
        def show_dm_question(self):
            if 'question' in self.open_windows:
                self.open_windows['question'].lift()
                return
                
            question_window = tk.Toplevel(self.root)
            question_window.title("Вопрос мастеру")
            question_window.geometry("500x400")
            question_window.configure(bg='#16537e')
            
            self.open_windows['question'] = question_window
            
            def on_question_close():
                if 'question' in self.open_windows:
                    del self.open_windows['question']
                question_window.destroy()
            
            question_window.protocol("WM_DELETE_WINDOW", on_question_close)
            
            tk.Label(question_window, text="Задать вопрос мастеру", 
                    font=('Arial', 16, 'bold'), fg='#ffffff', bg='#16537e').pack(pady=20)
            
            question_text = tk.Text(question_window, font=('Arial', 12), height=10, width=50,
                                bg='#ffffff', fg='#000000')
            question_text.pack(pady=20, padx=20, fill='both', expand=True)
            
            char_count_label = tk.Label(question_window, text="0/300", font=('Arial', 10), 
                                    fg='#cccccc', bg='#16537e')
            char_count_label.pack(anchor='e', padx=20)
            
            def update_char_count(event=None):
                content = question_text.get("1.0", tk.END)
                length = len(content) - 1
                if length > 300:
                    question_text.delete("1.300", tk.END)
                    length = 300
                char_count_label.config(text=f"{length}/300")
            
            question_text.bind('<KeyRelease>', update_char_count)
            
            def send_question():
                question = question_text.get("1.0", tk.END).strip()
                if question:
                    self.ask_dm_question(question, question_window)  # Pass window reference
                    
            send_btn = tk.Button(question_window, text="Отправить вопрос", 
                            command=send_question, bg='#27ae60', fg='white',
                            font=('Arial', 14, 'bold'), relief='flat', pady=10)
            send_btn.pack(pady=20)
            
        def initialize_microphone(self):
            try:
                self.microphone = sr.Microphone()
                with self.microphone as source:
                    self.recognizer.adjust_for_ambient_noise(source, duration=1)
            except Exception as e:
                print(f"Ошибка инициализации микрофона: {e}")
                messagebox.showwarning("Микрофон", "Микрофон не обнаружен. Голосовой ввод недоступен.")
                
        def toggle_recording(self):
            if not self.is_recording:
                self.start_recording()
            else:
                self.stop_recording()
                
        def start_recording(self):
            if not self.microphone:
                messagebox.showerror("Ошибка", "Микрофон недоступен")
                return
                
            self.is_recording = True
            if hasattr(self, 'recording_status'):
                self.recording_status.config(text="Запись...")
            
            threading.Thread(target=self.record_audio, daemon=True).start()
            
        def record_audio(self):
            try:
                with self.microphone as source:
                    audio = self.recognizer.listen(source, timeout=10, phrase_time_limit=10)
                    
                text = self.recognizer.recognize_google(audio, language="ru-RU")
                
                self.root.after(0, lambda: self.handle_voice_input(text))
                
            except sr.WaitTimeoutError:
                self.root.after(0, lambda: self.handle_voice_error("Тайм-аут записи"))
            except sr.UnknownValueError:
                self.root.after(0, lambda: self.handle_voice_error("Речь не распознана"))
            except sr.RequestError as e:
                self.root.after(0, lambda: self.handle_voice_error(f"Ошибка сервиса: {e}"))
            except Exception as e:
                self.root.after(0, lambda: self.handle_voice_error(f"Ошибка: {e}"))
                
        def handle_voice_input(self, text):
            self.is_recording = False
            if hasattr(self, 'recording_status'):
                self.recording_status.config(text="")
                
            if hasattr(self, 'chat_text'):
                self.chat_text.insert(tk.END, text)
                self.validate_chat_length()
                
        def handle_voice_error(self, error):
            self.is_recording = False
            if hasattr(self, 'recording_status'):
                self.recording_status.config(text=f"Ошибка: {error}")
                
        def stop_recording(self):
            self.is_recording = False
            if hasattr(self, 'recording_status'):
                self.recording_status.config(text="")
        
        def show_dm_thinking_overlay(self):
            overlay = tk.Toplevel(self.root)
            overlay.title("Мастер думает...")
            overlay.geometry("300x150")
            overlay.configure(bg='#9b59b6')
            overlay.transient(self.root)
            overlay.resizable(False, False)
            
            # Center the overlay
            overlay.geometry("+{}+{}".format(
                self.root.winfo_rootx() + 550,
                self.root.winfo_rooty() + 300
            ))
            
            tk.Label(overlay, text="Мастер обдумывает ответ...", 
                    font=('Arial', 14, 'bold'), fg='#ffffff', bg='#9b59b6').pack(pady=30)
            
            progress = ttk.Progressbar(overlay, mode='indeterminate')
            progress.pack(pady=20)
            progress.start()
            
            return overlay
            
        def show_dm_dice_roll_overlay(self):
            overlay = tk.Toplevel(self.root)
            overlay.title("Бросок кубика")
            overlay.geometry("250x100")
            overlay.configure(bg='#e74c3c')
            overlay.transient(self.root)
            overlay.resizable(False, False)
            
            tk.Label(overlay, text="Мастер бросает кубик...", 
                    font=('Arial', 12, 'bold'), fg='#ffffff', bg='#e74c3c').pack(pady=30)
            
            overlay.after(2000, overlay.destroy)
            
        def get_dm_response_streaming(self, player_action):
            if self.dm_thinking:
                return
                
            self.dm_thinking = True
            
            thinking_overlay = self.show_dm_thinking_overlay()
            
            def get_response_in_background():
                try:
                    headers = {
                        "Authorization": f"Bearer {self.api_key}",
                        "Content-Type": "application/json"
                    }
                    
                    system_prompt = f"""Ты опытный мастер подземелий для D&D 4-й редакции. 
    Персонаж игрока: {self.character_sheet.get('Имя персонажа', 'Неизвестный')}
    Класс: {self.character_sheet.get('Класс', 'Неизвестный')}
    Раса: {self.character_sheet.get('Раса', 'Неизвестная')}
    Текущее ХП: {self.character_sheet.get('Очки здоровья', 20)}/{self.character_sheet.get('Максимальные очки здоровья', 20)}

    Создавай захватывающие ответы. Включай:
    - Яркие описания окружения
    - Взаимодействие с НПС
    - Боевые столкновения когда уместно
    - Проверки навыков и испытания
    - Развитие сюжета

    Отвечай на русском языке. Держи ответы в пределах 600 символов для читабельности.

    Иногда бросай случайные события (1к20):
    1-5: Опасная ситуация
    6-10: Небольшой вызов
    11-15: Нейтральное событие
    16-20: Благоприятное событие

    Если игрок получает урон, обнови его ХП и четко укажи это."""
                    
                    data = {
                        "model": "glm-4",
                        "messages": [
                            {"role": "system", "content": system_prompt},
                            {"role": "user", "content": player_action}
                        ],
                        "temperature": 0.8,
                        "max_tokens": 600
                    }
                    
                    response = requests.post(self.api_url, headers=headers, json=data, timeout=40)
                    if response.status_code == 200:
                        result = response.json()
                        dm_response = result['choices'][0]['message']['content']
                        
                        self.root.after(0, lambda: self.handle_dm_response(dm_response, thinking_overlay))
                    else:
                        self.root.after(0, lambda: self.handle_dm_error(thinking_overlay, "Ошибка API"))
                except requests.exceptions.Timeout:
                    self.root.after(0, lambda: self.handle_dm_error(thinking_overlay, "Тайм-аут подключения"))
                except requests.exceptions.ConnectionError:
                    self.root.after(0, lambda: self.handle_dm_error(thinking_overlay, "Нет подключения к интернету"))
                except Exception as e:
                    self.root.after(0, lambda: self.handle_dm_error(thinking_overlay, f"Неизвестная ошибка: {str(e)}"))
            
            threading.Thread(target=get_response_in_background, daemon=True).start()
            
        def handle_dm_response(self, response, thinking_overlay):
            thinking_overlay.destroy()
            self.dm_thinking = False
            
            self.process_dm_response_effects(response)
            
            self.chat_history.append({
                'sender': 'dm',
                'message': response,
                'timestamp': time.strftime("%H:%M")
            })
            
            self.refresh_chat_display()
            self.save_character()
            
            if random.randint(1, 15) == 1:
                self.dm_random_event()
                
        def handle_dm_error(self, thinking_overlay, error_msg):
            thinking_overlay.destroy()
            self.dm_thinking = False
            
            error_response = f"Мастер временно недоступен: {error_msg}. Попробуйте снова."
            
            self.chat_history.append({
                'sender': 'dm',
                'message': error_response,
                'timestamp': time.strftime("%H:%M")
            })
            
            self.refresh_chat_display()
                
        def process_dm_response_effects(self, response):
            damage_pattern = r'(\d+)\s*(?:урона|damage|повреждений|урон)'
            heal_pattern = r'восстанавливаете?\s*(\d+)\s*(?:хп|hp|здоровья|очков)'
            
            damage_match = re.search(damage_pattern, response.lower())
            heal_match = re.search(heal_pattern, response.lower())
            
            if damage_match:
                damage = int(damage_match.group(1))
                current_hp = self.character_sheet.get('Очки здоровья', 20)
                new_hp = max(0, current_hp - damage)
                self.character_sheet['Очки здоровья'] = new_hp
                
                if new_hp == 0:
                    self.handle_death_saves()
                    
            elif heal_match:
                healing = int(heal_match.group(1))
                current_hp = self.character_sheet.get('Очки здоровья', 20)
                max_hp = self.character_sheet.get('Максимальные очки здоровья', 20)
                new_hp = min(max_hp, current_hp + healing)
                self.character_sheet['Очки здоровья'] = new_hp
                
        def handle_death_saves(self):
            death_window = tk.Toplevel(self.root)
            death_window.title("Спасброски от смерти!")
            death_window.geometry("500x400")
            death_window.configure(bg='#8b0000')
            death_window.transient(self.root)
            death_window.grab_set()
            
            tk.Label(death_window, text="СПАСБРОСКИ ОТ СМЕРТИ!", 
                    font=('Arial', 20, 'bold'), fg='#ffffff', bg='#8b0000').pack(pady=30)
            
            tk.Label(death_window, text="Бросаем 3к20. Нужно 15+ хотя бы на одном кубике!", 
                    font=('Arial', 14), fg='#ffffff', bg='#8b0000').pack(pady=20)
            
            rolls = [random.randint(1, 20) for _ in range(3)]
            success = any(roll >= 15 for roll in rolls)
            
            rolls_text = f"Броски: {', '.join(map(str, rolls))}"
            tk.Label(death_window, text=rolls_text, 
                    font=('Arial', 16, 'bold'), fg='#ffff00', bg='#8b0000').pack(pady=20)
            
            if success:
                result_text = "УСПЕХ! Персонаж выжил с 1 ХП!"
                result_color = '#00ff00'
                self.character_sheet['Очки здоровья'] = 1
            else:
                result_text = "НЕУДАЧА! Персонаж погиб..."
                result_color = '#ff0000'
                
            tk.Label(death_window, text=result_text, 
                    font=('Arial', 16, 'bold'), fg=result_color, bg='#8b0000').pack(pady=30)
            
            tk.Button(death_window, text="Продолжить", command=death_window.destroy,
                    bg='#4a4a4a', fg='white', font=('Arial', 12, 'bold')).pack(pady=20)
                    
        def dm_random_event(self):
            self.show_dm_dice_roll_overlay()
            
            event_roll = random.randint(1, 20)
            
            event_window = tk.Toplevel(self.root)
            event_window.title("Случайное событие!")
            event_window.geometry("600x400")
            event_window.configure(bg='#9b59b6')
            event_window.transient(self.root)
            
            tk.Label(event_window, text="СЛУЧАЙНОЕ СОБЫТИЕ!", 
                    font=('Arial', 18, 'bold'), fg='#ffffff', bg='#9b59b6').pack(pady=30)
            
            if event_roll <= 5:
                event_text = "Опасная ситуация! Мастер бросил кубик судьбы..."
                color = '#e74c3c'
            elif event_roll <= 10:
                event_text = "Небольшое испытание ожидает вас впереди."
                color = '#f39c12'
            elif event_roll <= 15:
                event_text = "Ничего особенного не происходит."
                color = '#95a5a6'
            else:
                event_text = "Удача улыбается вам!"
                color = '#27ae60'
                
            tk.Label(event_window, text=f"Результат броска: {event_roll}", 
                    font=('Arial', 16, 'bold'), fg='#ffffff', bg='#9b59b6').pack(pady=10)
            
            tk.Label(event_window, text=event_text, font=('Arial', 14), 
                    fg='#ffffff', bg=color, padx=30, pady=20, wraplength=400).pack(pady=30)
            
            tk.Button(event_window, text="Продолжить приключение", 
                    command=event_window.destroy, bg='#34495e', fg='white',
                    font=('Arial', 12, 'bold')).pack(pady=30)
                    
        def ask_dm_question(self, question, question_window=None):
            if self.dm_thinking:
                messagebox.showwarning("Подождите", "Мастер уже обрабатывает запрос...")
                return
                
            self.dm_thinking = True
            
            thinking_overlay = self.show_dm_thinking_overlay()
            
            def get_answer_in_background():
                try:
                    headers = {
                        "Authorization": f"Bearer {self.api_key}",
                        "Content-Type": "application/json"
                    }
                    
                    system_prompt = """Ты опытный мастер D&D 4-й редакции. Отвечай на вопросы игроков кратко и по делу. 
    Разрешай переброски кубиков только в исключительных случаях. 
    Помогай с правилами и механиками игры. Отвечай на русском языке.
    Ответ должен быть в пределах 300 символов."""
                    
                    data = {
                        "model": "glm-4",
                        "messages": [
                            {"role": "system", "content": system_prompt},
                            {"role": "user", "content": question}
                        ],
                        "temperature": 0.7,
                        "max_tokens": 300
                    }
                    
                    response = requests.post(self.api_url, headers=headers, json=data, timeout=25)
                    if response.status_code == 200:
                        result = response.json()
                        answer = result['choices'][0]['message']['content']
                        
                        self.root.after(0, lambda: self.show_dm_answer(answer, thinking_overlay, question_window))
                    else:
                        self.root.after(0, lambda: self.show_dm_answer("Мастер занят, попробуйте позже.", thinking_overlay, question_window))
                except requests.exceptions.Timeout:
                    self.root.after(0, lambda: self.show_dm_answer("Мастер не отвечает. Попробуйте позже.", thinking_overlay, question_window))
                except requests.exceptions.ConnectionError:
                    self.root.after(0, lambda: self.show_dm_answer("Нет подключения к интернету.", thinking_overlay, question_window))
                except Exception as e:
                    self.root.after(0, lambda: self.show_dm_answer("Мастер временно недоступен.", thinking_overlay, question_window))
            
            threading.Thread(target=get_answer_in_background, daemon=True).start()
            
        def show_dm_answer(self, answer, thinking_overlay, question_window=None):
            thinking_overlay.destroy()
            self.dm_thinking = False
            
            # Show answer in the same context where question was asked
            if question_window and question_window.winfo_exists():
                # Create answer display in the question window
                answer_frame = tk.Frame(question_window, bg='#0f3460', relief='raised', bd=2)
                answer_frame.pack(pady=20, padx=20, fill='both', expand=True)
                
                tk.Label(answer_frame, text="Ответ мастера:", 
                        font=('Arial', 14, 'bold'), fg='#ffffff', bg='#0f3460').pack(pady=10)
                
                answer_text = tk.Text(answer_frame, font=('Arial', 12), height=6, width=50,
                                    bg='#16537e', fg='#ffffff', wrap='word')
                answer_text.pack(pady=10, padx=10, fill='both', expand=True)
                answer_text.insert('1.0', answer)
                answer_text.config(state='disabled')
                
                # Allow continuing conversation
                continue_frame = tk.Frame(question_window, bg='#16537e')
                continue_frame.pack(pady=10, padx=20, fill='x')
                
                tk.Label(continue_frame, text="Дополнительный вопрос:", 
                        font=('Arial', 12, 'bold'), fg='#ffffff', bg='#16537e').pack(anchor='w')
                
                follow_up_text = tk.Text(continue_frame, font=('Arial', 11), height=3, width=50,
                                    bg='#ffffff', fg='#000000')
                follow_up_text.pack(pady=5, fill='x')
                
                def send_follow_up():
                    follow_up = follow_up_text.get("1.0", tk.END).strip()
                    if follow_up:
                        self.ask_dm_question(follow_up, question_window)
                
                tk.Button(continue_frame, text="Задать дополнительный вопрос", 
                        command=send_follow_up, bg='#f39c12', fg='white',
                        font=('Arial', 12)).pack(pady=10)
            else:
                # Fallback to messagebox if question window is closed
                messagebox.showinfo("Ответ мастера", answer)
                
        def start_adventure(self):
            welcome_message = f"""Добро пожаловать в мир приключений, {self.character_sheet.get('Имя персонажа', self.nickname)}!
            
    Вы {self.character_sheet.get('Раса', 'неизвестной расы')} {self.character_sheet.get('Класс', 'неизвестного класса')}.
            
    Вы просыпаетесь в таверне "Спящий дракон". Солнце уже высоко, а вокруг вас шумят другие путешественники. 
    На столе лежит загадочное письмо с вашим именем...
            
    Что вы делаете?"""

            self.chat_history.append({
                'sender': 'dm',
                'message': welcome_message,
                'timestamp': time.strftime("%H:%M")
            })
            
            self.save_character()
            
        def clear_screen(self):
            for widget in self.root.winfo_children():
                widget.destroy()
                
        def run(self):
            try:
                pygame.mixer.init()
            except:
                pass
                
            self.root.mainloop()

    if __name__ == "__main__":
        game = DnDGame()
        game.run()



        def setup_lobby_screen(self):
            self.clear_screen()
            
            lobby_frame = tk.Frame(self.root, bg='#000000') # Black background
            lobby_frame.pack(fill='both', expand=True)
            
            # You can add some drawings or placeholder images here
            # For now, just a message
            tk.Label(lobby_frame, text='Ожидание начала игры...', 
                    font=('Arial', 24, 'bold'), fg='#ffffff', bg='#000000').pack(pady=200)
            
            tk.Label(lobby_frame, text='Мастер скоро начнет приключение.', 
                    font=('Arial', 16), fg='#cccccc', bg='#000000').pack(pady=20)

