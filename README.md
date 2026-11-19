# -evirici
from kivy.app import App
from kivy.uix.boxlayout import BoxLayout
from kivy.uix.label import Label
from kivy.uix.textinput import TextInput
from kivy.uix.button import Button
from kivy.uix.checkbox import CheckBox
from kivy.core.clipboard import Clipboard

# --- ÇEVİRİ ALGORİTMASI ---
VOWELS = set("aeıioöuüAEIİOÖUÜ")
CONJUNCTIONS = {"ve", "veya", "ile", "ama", "fakat", "lakin", "ancak", "çünkü", "de", "da", "ki", "ya", "hem", "ne", "oysa", "halbuki"}
TIME_WORDS = {"dün", "bugün", "yarın", "şimdi", "sonra", "önce", "sabah", "akşam", "gece", "geçen", "hafta", "ay", "yıl"}
CUSTOM_DICT = {"okul": "lukuo", "oyun": "niyuo"}

INFLECTIONAL_SUFFIXES = [
    "larından", "lerinden", "larımız", "lerimiz", "larınız", "leriniz",
    "ları", "leri", "lar", "ler",
    "ımız", "imiz", "umuz", "ümüz", "ınız", "iniz", "unuz", "ünüz",
    "dan", "den", "tan", "ten", "da", "de", "ta", "te",
    "nın", "nin", "nun", "nün", "ın", "in", "un", "ün",
    "ya", "ye", "yı", "yi", "yu", "yü", "a", "e", "ı", "i", "u", "ü",
    "dım", "dim", "dum", "düm", "tım", "tim", "tum", "tüm",
    "dın", "din", "dun", "dün", "tın", "tin", "tun", "tün",
    "dı", "di", "du", "dü", "tı", "ti", "tu", "tü",
    "mış", "miş", "muş", "müş", "yor", "acak", "ecek", "r", "ar", "er"
]

def tr_lower(text):
    return text.replace("I", "ı").replace("İ", "i").lower()

def tr_upper_first(text):
    if not text:
        return text
    first = text
    first_up = "İ" if first == "i" else ("I" if first == "ı" else first.upper())
    return first_up + text[1:]

def split_root_suffix(word):
    if "+" in word:
        parts = word.split("+", 1)
        return parts, parts
    lower_w = tr_lower(word)
    for root in sorted(CUSTOM_DICT.keys(), key=len, reverse=True):
        if lower_w.startswith(root) and len(lower_w) > len(root):
            return word[:len(root)], word[len(root):]
    for suffix in INFLECTIONAL_SUFFIXES:
        if lower_w.endswith(suffix) and len(lower_w) - len(suffix) >= 2:
            return word[:-len(suffix)], word[-len(suffix):]
    return word, ""

def fix_pronunciation(word):
    if len(word) < 2:
        return word
    if word[-1] in VOWELS and len(word) >= 3 and word[-2] in VOWELS:
        return word
    if word not in VOWELS and word not in VOWELS:
        vowel = next((c for c in word if c in VOWELS), "i")
        return word + vowel + word[1:]
    return word

def transform_root(root):
    lower_root = tr_lower(root)
    if lower_root in CUSTOM_DICT:
        return CUSTOM_DICT[lower_root]
    if len(lower_root) <= 1:
        return lower_root
    first_char, last_char, middle = lower_root, lower_root[-1], lower_root[1:-1]
    swapped = last_char + middle + first_char
    if first_char in VOWELS and last_char not in VOWELS and len(swapped) > 3:
        swapped = swapped + first_char
    else:
        swapped = fix_pronunciation(swapped)
    return swapped

def translate_word(word, is_sentence_start=False):
    if word.startswith("*"):
        return word[1:]
    if not is_sentence_start and word.isupper():
        return word
    if any(ch.isdigit() for ch in word):
        return word
    clean_word = word.lstrip("*")
    if tr_lower(clean_word) in CONJUNCTIONS:
        return clean_word
    root, suffix = split_root_suffix(clean_word)
    return transform_root(root) + tr_lower(suffix)

def detect_role(word, index, total_words):
    w = tr_lower(word.replace("+", ""))
    if index == total_words - 1:
        return "Yüklem"
    if w in TIME_WORDS:
        return "Zaman"
    if w in {"ben", "sen", "o", "biz", "siz", "onlar"} or index == 0:
        return "Özne"
    if w.endswith(("da", "de", "ta", "te", "dan", "den", "tan", "ten", "ya", "ye", "a", "e")):
        return "Diğer"
    return "Nesne"

def turkce_to_agaca(sentence, apply_word_order=True):
    punct = ""
    if sentence and sentence[-1] in ".!?":
        punct = sentence[-1]
        sentence = sentence[:-1]
    words = sentence.strip().split()
    if not words:
        return ""
    analyzed = []
    for i, w in enumerate(words):
        role = detect_role(w, i, len(words))
        translated = translate_word(w, is_sentence_start=(i == 0))
        analyzed.append({"role": role, "text": translated})
    if apply_word_order:
        order_priority = {"Özne": 1, "Yüklem": 2, "Nesne": 3, "Diğer": 4, "Zaman": 5}
        analyzed.sort(key=lambda x: order_priority.get(x["role"], 99))
    result_words = [item["text"] for item in analyzed]
    if result_words:
        result_words = tr_upper_first(result_words)
    return " ".join(result_words) + punct

# --- ANDROID ARAYÜZÜ (KIVY) ---
class AgacaApp(App):
    def build(self):
        layout = BoxLayout(orientation="vertical", padding=20, spacing=15)

        title = Label(text="Türkçe -> Agaca Çeviri", font_size="24sp", size_hint=(1, 0.15), bold=True)
        layout.add_widget(title)

        self.input_box = TextInput(hint_text="Türkçe metni buraya yazın...", font_size="18sp", size_hint=(1, 0.25))
        layout.add_widget(self.input_box)

        order_box = BoxLayout(orientation="horizontal", size_hint=(1, 0.1))
        self.order_check = CheckBox(active=True, size_hint=(0.2, 1))
        order_label = Label(text="Cümle sırasını uygula (Ö-Y-N-Diğer-Zaman)", font_size="15sp", halign="left")
        order_box.add_widget(self.order_check)
        order_box.add_widget(order_label)
        layout.add_widget(order_box)

        translate_btn = Button(text="Çevir", font_size="20sp", size_hint=(1, 0.15), background_color=(0.1, 0.4, 0.8, 1))
        translate_btn.bind(on_press=self.do_translate)
        layout.add_widget(translate_btn)

        self.output_box = TextInput(hint_text="Çeviri sonucu...", readonly=True, font_size="18sp", size_hint=(1, 0.25))
        layout.add_widget(self.output_box)

        copy_btn = Button(text="Sonucu Kopyala", font_size="18sp", size_hint=(1, 0.12))
        copy_btn.bind(on_press=self.copy_result)
        layout.add_widget(copy_btn)

        return layout

    def do_translate(self, instance):
        text = self.input_box.text
        self.output_box.text = turkce_to_agaca(text, self.order_check.active)

    def copy_result(self, instance):
        if self.output_box.text:
            Clipboard.copy(self.output_box.text)

if __name__ == "__main__":
    AgacaApp().run()
