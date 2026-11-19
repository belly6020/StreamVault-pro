name: streamvault-player
display_name: StreamVault Player
version: 1.1.0
description: IPTV player that loads M3U playlists and plays channels through VLC
language: python
entrypoint: main.py
supported_languages:
  - en
  - ar
  - fr
  - es
default_language: en
requirements:
  system_packages:
    - python3
    - python3-tk
    - python3-vlc
    - vlc
files:
  main.py: |
    import re
    import tkinter as tk
    from tkinter import messagebox, ttk
    from urllib.request import urlopen

    import vlc


    TRANSLATIONS = {
        "en": {
            "app_title": "StreamVault Player",
            "url_label": "Playlist URL",
            "load": "Load playlist",
            "play": "Play",
            "stop": "Stop",
            "language": "Language",
            "status_enter_url": "Enter an M3U playlist URL, then click Load",
            "status_loading": "Loading...",
            "status_count": "Channels: {count}",
            "status_playing": "Playing: {name}",
            "status_error": "Could not load playlist: {error}",
            "warn_title": "Warning",
            "warn_empty_url": "Please enter a valid URL",
            "group_general": "General",
            "unknown": "Unknown",
        },
        "ar": {
            "app_title": "مشغل StreamVault",
            "url_label": "رابط القائمة",
            "load": "تحميل القائمة",
            "play": "تشغيل",
            "stop": "إيقاف",
            "language": "اللغة",
            "status_enter_url": "أدخل رابط قائمة M3U ثم اضغط تحميل",
            "status_loading": "جارٍ التحميل...",
            "status_count": "عدد القنوات: {count}",
            "status_playing": "تشغيل: {name}",
            "status_error": "تعذّر تحميل القائمة: {error}",
            "warn_title": "تنبيه",
            "warn_empty_url": "الرجاء إدخال رابط صحيح",
            "group_general": "عام",
            "unknown": "غير معروف",
        },
        "fr": {
            "app_title": "StreamVault Player",
            "url_label": "URL de la playlist",
            "load": "Charger la playlist",
            "play": "Lire",
            "stop": "Arrêter",
            "language": "Langue",
            "status_enter_url": "Saisissez l'URL d'une playlist M3U puis cliquez sur Charger",
            "status_loading": "Chargement...",
            "status_count": "Chaînes : {count}",
            "status_playing": "Lecture : {name}",
            "status_error": "Impossible de charger la playlist : {error}",
            "warn_title": "Attention",
            "warn_empty_url": "Veuillez saisir une URL valide",
            "group_general": "Général",
            "unknown": "Inconnu",
        },
        "es": {
            "app_title": "StreamVault Player",
            "url_label": "URL de la lista",
            "load": "Cargar lista",
            "play": "Reproducir",
            "stop": "Detener",
            "language": "Idioma",
            "status_enter_url": "Introduce la URL de una lista M3U y pulsa Cargar",
            "status_loading": "Cargando...",
            "status_count": "Canales: {count}",
            "status_playing": "Reproduciendo: {name}",
            "status_error": "No se pudo cargar la lista: {error}",
            "warn_title": "Aviso",
            "warn_empty_url": "Introduce una URL válida",
            "group_general": "General",
            "unknown": "Desconocido",
        },
    }

    LANGUAGE_NAMES = {"en": "English", "ar": "العربية", "fr": "Français", "es": "Español"}


    def parse_m3u(text, default_group, unknown_name):
        channels = []
        name, group = None, default_group
        for raw in text.splitlines():
            line = raw.strip()
            if line.startswith("#EXTINF"):
                name = line.rsplit(",", 1)[-1].strip() or unknown_name
                match = re.search(r'group-title="([^"]*)"', line)
                group = match.group(1) if match and match.group(1) else default_group
            elif line and not line.startswith("#"):
                channels.append({"name": name or unknown_name, "group": group, "url": line})
                name, group = None, default_group
        return channels


    class StreamVaultApp:
        def __init__(self, root):
            self.root = root
            self.lang = "en"
            self.channels = []
            self.status_key = "status_enter_url"
            self.status_args = {}

            self.instance = vlc.Instance()
            self.player = self.instance.media_player_new()

            top = tk.Frame(root)
            top.pack(fill=tk.X, padx=8, pady=8)
            self.url_label = tk.Label(top)
            self.url_label.pack(side=tk.LEFT, padx=(0, 6))
            self.url_entry = tk.Entry(top)
            self.url_entry.pack(side=tk.LEFT, fill=tk.X, expand=True, padx=(0, 8))
            self.load_button = tk.Button(top, command=self.load_playlist)
            self.load_button.pack(side=tk.LEFT)

            lang_frame = tk.Frame(root)
            lang_frame.pack(fill=tk.X, padx=8)
            self.lang_label = tk.Label(lang_frame)
            self.lang_label.pack(side=tk.LEFT)
            self.lang_var = tk.StringVar(value=LANGUAGE_NAMES[self.lang])
            self.lang_box = ttk.Combobox(lang_frame, textvariable=self.lang_var,
                                         values=list(LANGUAGE_NAMES.values()), state="readonly", width=12)
            self.lang_box.pack(side=tk.LEFT, padx=6)
            self.lang_box.bind("<<ComboboxSelected>>", self.change_language)

            self.video_frame = tk.Frame(root, bg="black", height=360)
            self.video_frame.pack(fill=tk.X, padx=8, pady=6)
            self.video_frame.pack_propagate(False)

            controls = tk.Frame(root)
            controls.pack(fill=tk.X, padx=8, pady=6)
            self.play_button = tk.Button(controls, command=self.play_selected)
            self.play_button.pack(side=tk.LEFT)
            self.stop_button = tk.Button(controls, command=self.player.stop)
            self.stop_button.pack(side=tk.LEFT, padx=6)

            list_frame = tk.Frame(root)
            list_frame.pack(fill=tk.BOTH, expand=True, padx=8, pady=(0, 8))
            scrollbar = tk.Scrollbar(list_frame)
            scrollbar.pack(side=tk.RIGHT, fill=tk.Y)
            self.listbox = tk.Listbox(list_frame, yscrollcommand=scrollbar.set)
            self.listbox.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)
            scrollbar.config(command=self.listbox.yview)
            self.listbox.bind("<Double-Button-1>", lambda e: self.play_selected())

            self.status = tk.Label(root, anchor="w")
            self.status.pack(fill=tk.X, padx=8, pady=(0, 6))

            self.apply_language()
            self.root.update()
            self.player.set_xwindow(self.video_frame.winfo_id())
            self.root.protocol("WM_DELETE_WINDOW", self.on_close)

        def t(self, key, **kwargs):
            text = TRANSLATIONS[self.lang].get(key, TRANSLATIONS["en"][key])
            return text.format(**kwargs) if kwargs else text

        def apply_language(self):
            self.root.title(self.t("app_title"))
            self.url_label.config(text=self.t("url_label"))
            self.load_button.config(text=self.t("load"))
            self.play_button.config(text=self.t("play"))
            self.stop_button.config(text=self.t("stop"))
            self.lang_label.config(text=self.t("language"))
            self.set_status(self.status_key, **self.status_args)
            self.refresh_listbox()

        def change_language(self, event=None):
            selected = self.lang_var.get()
            for code, name in LANGUAGE_NAMES.items():
                if name == selected:
                    self.lang = code
                    break
            self.apply_language()

        def set_status(self, key, **kwargs):
            self.status_key, self.status_args = key, kwargs
            self.status.config(text=self.t(key, **kwargs))

        def refresh_listbox(self):
            self.listbox.delete(0, tk.END)
            for ch in self.channels:
                self.listbox.insert(tk.END, f"[{ch['group']}] {ch['name']}")

        def load_playlist(self):
            url = self.url_entry.get().strip()
            if not url:
                messagebox.showwarning(self.t("warn_title"), self.t("warn_empty_url"))
                return
            self.set_status("status_loading")
            self.root.update_idletasks()
            try:
                with urlopen(url, timeout=15) as response:
                    text = response.read().decode("utf-8", errors="ignore")
                self.channels = parse_m3u(text, self.t("group_general"), self.t("unknown"))
            except Exception as exc:
                self.set_status("status_error", error=str(exc))
                return
            self.refresh_listbox()
            self.set_status("status_count", count=len(self.channels))

        def play_selected(self):
            selection = self.listbox.curselection()
            if not selection:
                return
            channel = self.channels[selection[0]]
            self.set_status("status_playing", name=channel["name"])
            media = self.instance.media_new(channel["url"])
            self.player.set_media(media)
            self.player.play()

        def on_close(self):
            self.player.stop()
            self.player.release()
            self.instance.release()
            self.root.destroy()


    if __name__ == "__main__":
        root = tk.Tk()
        StreamVaultApp(root)
        root.mainloop()
