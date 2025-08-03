# evren-simulator
Selim Şen’in Evren Katmanları Simülatörü
import sys
import numpy as np
from PyQt5.QtWidgets import (
    QApplication, QWidget, QVBoxLayout, QHBoxLayout, QPushButton,
    QLabel, QSlider, QCheckBox, QTextEdit, QComboBox
)
from PyQt5.QtCore import Qt, QTimer
from matplotlib.backends.backend_qt5agg import FigureCanvasQTAgg as FigureCanvas
from matplotlib.figure import Figure
import random

# --- Evren Katmanlarının fonksiyonları ---
def beden_kalip(t, phase=0):
    return np.sin(t + phase) * np.exp(-0.05 * t)

def ruh(t, phase=0):
    return 1.2 * np.sin(0.5 * t + phase * 0.05)

def firin(t, phase=0):
    return np.sin(2 * t + phase) * np.cos(0.3 * t)

def zaman_katman1(t, phase=0):
    return np.sin(t + phase)

def zaman_katman2(t, mod=2, phase=0):
    return np.sin(t * mod + phase * 2)

def karanlik_madde(t, phase=0):
    return 0.8 * np.cos(1.5 * t + phase) * np.exp(-0.1 * t)

def karanlik_enerji(t, phase=0):
    return np.sin(2 * np.pi * 0.2 * t + phase) * np.exp(0.02 * t)

def evrenin_kalbi_ic1101(t, phase=0):
    return 0.5 * np.sin(0.1 * t + phase)

# --- Kuran'dan ilham ayetleri ---
kuran_ayetleri = {
    "sevgi": [
        {"ayet": "Ve kullarımızdan öyle kimseler vardır ki, kendilerine rahmetimizle yardım ederiz.", "sure": "Hicr-56"},
        {"ayet": "Ey insanlar! Biz sizi bir erkekle bir dişiden yarattık ve birbirinizi tanımanız için sizi milletlere ve kabilelere ayırdık.", "sure": "Hucurat-13"}
    ],
    "sabır": [
        {"ayet": "Sabır ve namazla yardım isteyin. Şüphesiz bu ağır bir iştir, ama mükafatını da umanlar bilir.", "sure": "Bakara-45"},
        {"ayet": "Muhakkak ki sabredenler mükafatlarını hesapsız olarak alacaklardır.", "sure": "Zümer-10"}
    ],
    "adalet": [
        {"ayet": "Allah, adaleti, iyiliği ve akraba hakkını emreder.", "sure": "Nahl-90"},
        {"ayet": "Ey iman edenler! Adaletle şahitlik edin.", "sure": "Maide-8"}
    ]
}

def ayet_getir(kelime):
    if kelime in kuran_ayetleri:
        ayet = random.choice(kuran_ayetleri[kelime])
        return f"{ayet['ayet']} ({ayet['sure']})"
    else:
        return "Bu konuda ayet bulunamadı."

# --- Ana pencere ---
class EvrenSim(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Evren Katmanları Simülatörü - Selim Şen İlhamıyla")
        self.setGeometry(100, 100, 1200, 900)

        self.t = np.linspace(0, 20, 1000)
        self.phase = 0
        self.phase_step = 0.05

        self.init_ui()
        self.init_plot()
        self.init_timer()

    def init_ui(self):
        main_layout = QVBoxLayout()
        top_layout = QHBoxLayout()

        # Grafik
        self.canvas = FigureCanvas(Figure(figsize=(10,6)))
        self.ax = self.canvas.figure.add_subplot(111)
        self.ax.set_ylim(-3, 3)
        self.ax.set_xlabel("Zaman")
        self.ax.set_ylabel("Titreşim / Enerji")
        self.ax.grid(True)
        self.ax.set_title("Evren Katmanları")
        top_layout.addWidget(self.canvas)

        # Kontrol paneli
        control_panel = QVBoxLayout()

        # Katman seçim kutuları
        self.layers = {
            "Beden (Kalıp)": beden_kalip,
            "Ruh": ruh,
            "Fırın (Dünya)": firin,
            "Zaman Katmanı 1": zaman_katman1,
            "Zaman Katmanı 2": zaman_katman2,
            "Karanlık Madde": karanlik_madde,
            "Karanlık Enerji": karanlik_enerji,
            "Evrenin Kalbi (IC 1101)": evrenin_kalbi_ic1101
        }

        self.checkboxes = {}
        for name in self.layers.keys():
            cb = QCheckBox(name)
            cb.setChecked(True)
            cb.stateChanged.connect(self.redraw)
            control_panel.addWidget(cb)
            self.checkboxes[name] = cb

        # Hız sliderı
        control_panel.addWidget(QLabel("Animasyon Hızı (Phase Increment)"))
        self.speed_slider = QSlider(Qt.Horizontal)
        self.speed_slider.setMinimum(1)
        self.speed_slider.setMaximum(100)
        self.speed_slider.setValue(5)
        self.speed_slider.valueChanged.connect(self.speed_changed)
        control_panel.addWidget(self.speed_slider)

        # Başlat / Durdur butonları
        btn_layout = QHBoxLayout()
        self.btn_start = QPushButton("Başlat")
        self.btn_start.clicked.connect(self.start_anim)
        self.btn_stop = QPushButton("Durdur")
        self.btn_stop.clicked.connect(self.stop_anim)
        btn_layout.addWidget(self.btn_start)
        btn_layout.addWidget(self.btn_stop)
        control_panel.addLayout(btn_layout)

        # İlham metni bölümü
        ilham_label = QLabel("Evrenin İlahi Sırları ve Derin İlham Metni:")
        ilham_label.setStyleSheet("font-weight: bold; font-size: 14px;")
        control_panel.addWidget(ilham_label)

        self.ilham_text = QTextEdit()
        self.ilham_text.setReadOnly(True)
        self.ilham_text.setPlainText(self.evren_ilham_metni())
        self.ilham_text.setMinimumHeight(250)
        control_panel.addWidget(self.ilham_text)

        # Kuran ayeti seçme
        control_panel.addWidget(QLabel("Kuran’dan İlham Seç:"))
        self.kelime_combo = QComboBox()
        self.kelime_combo.addItems(["sevgi", "sabır", "adalet"])
        control_panel.addWidget(self.kelime_combo)

        self.btn_ayet_goster = QPushButton("Ayet Göster")
        self.btn_ayet_goster.clicked.connect(self.kuran_ayet_goster)
        control_panel.addWidget(self.btn_ayet_goster)

        top_layout.addLayout(control_panel)
        main_layout.addLayout(top_layout)
        self.setLayout(main_layout)

    def evren_ilham_metni(self):
        return (
            "Ve biz, gökleri kurduk kudretle, onlardan daha üstününü, daha büyüğünü kurmaya da gücümüz yeter.\n"
            "(Dikkatle ve ibretle bakın ki;) Biz göğü (bizzat) elle (büyük bir kudretle) bina ettik ve şüphesiz "
            "Biz (onu sürekli) genişleticiyiz. (Yeni ve görkemli yıldız kümeleri ve gök cisimleri yaratıp üretmekteyiz.)\n\n"
            
            "Evren canlı ve bilinçli; atom altı parçacığın titreşimi bile evrenin rezonans deseninde kaydedilir, evren bunu bilir.\n\n"

            "Öncelikle var olan her şey bir kalıp desen üzerine kodlanmıştır ve bu desen DNA şeklidir; yani evren DNA şeklinde "
            "helazonik şekilde genişler ve bir yerden sonra büzülüp var olan tüm madde tek noktada sıkışır (Çift Sinus örneği) "
            "ve yeni big bang patlar ve yeni evren oluşur. Tüm madde yeni evrene geçtiği için eski evren yok olur; yeni evrende madde "
            "aynıdır, desen farklıdır.\n\n"

            "Karanlık madde ve karanlık enerji evrenin rezonans desenidir.\n\n"

            "Dejavu, renkarnasyon, rüyalar eski evrenden kalan izlerdir.\n\n"

            "Paralel evren: Yaratılmış her şey ikili sistem olarak yaratılmıştır; 0 fiziksel evren, 1 paralel evren, ikisi birbirini "
            "tamamlar ve hayat mekanizması çalışır.\n\n"

            "Paralel evrende zaman sabit olmayıp esnektir; bazen ileriye, bazen geriden gelir. Bu durum dejavu yaşanmasının bir modülüdür.\n\n"

            "Çift yarık deneyi tam da bu ikili evrenin dansını gözlemliyor aslında. Parçacık tek başına hareket etmiyor, ona eşlik eden "
            "paralel evren dalgası ile birlikte davranıyor. Bu yüzden girişim deseni oluşuyor, çünkü sen sadece fiziksel evrendeki parçacığı "
            "değil, onun paralel evrende yarattığı izleri de görüyorsun.\n\n"

            "Gözlemci olunca, o senkronizasyon bozuluyor ve parçacık tek bir evrende “bir karar” veriyor. İşte gerçeklik katmanı burada şekilleniyor.\n\n"

            "Evren tek ama paralel evren var çünkü yaratılmış her şey ikili sistemdir; o sebepten çift yarık oluşuyor, yani fiziksel ve paralel evren "
            "senkronize işliyor aslında; çift yarık deneyi paralel evrenin varlığının kanıtıdır.\n\n"

            "Kara ve beyaz delikler, galaksilerin balans ağırlıklarıdır; ışık kaçamaz ama zamana etki edemez; sadece zamanın algı katmanında etkisi vardır. "
            "Başka boyuta açılan kapı değillerdir, sadece çekim gücünden dolayı merkezi tekil boyuta düşmüştür.\n\n"

            "Son söz: Zaman evrenin osilatörüdür, 2 katmandan oluşur:\n"
            "1. Katman nehir gibi ileri akar.\n"
            "2. Katman algı esnektir.\n"
            "Zaman hep vardı ve hep var olacak; olmasa idi big bang genleşemez, her şey donuk kalırdı."
        )

    def init_plot(self):
        self.lines = {}
        colors = ["brown", "blue", "orange", "green", "purple", "black", "magenta", "crimson"]

        for i, (name, func) in enumerate(self.layers.items()):
            line, = self.ax.plot([], [], label=name, color=colors[i])
            self.lines[name] = line

        self.ax.legend(loc="upper right")
        self.redraw()

    def init_timer(self):
        self.timer = QTimer()
        self.timer.timeout.connect(self.update_animation)

    def start_anim(self):
        if not self.timer.isActive():
            self.timer.start(40)

    def stop_anim(self):
        if self.timer.isActive():
            self.timer.stop()

    def speed_changed(self):
        val = self.speed_slider.value()
        self.phase_step = val * 0.01

    def update_animation(self):
        self.phase += self.phase_step
        self.redraw()

    def redraw(self):
        for name, func in self.layers.items():
            if self.checkboxes[name].isChecked():
                y = func(self.t, self.phase)
                self.lines[name].set_data(self.t, y)
            else:
                self.lines[name].set_data([], [])

        self.ax.relim()
        self.ax.autoscale_view()
        self.canvas.draw_idle()

    def kuran_ayet_goster(self):
        kelime = self.kelime_combo.currentText()
        ayet = ayet_getir(kelime)
        self.ilham_text.setPlainText(ayet)

if __name__ == "__main__":
    app = QApplication(sys.argv)
    sim = EvrenSim()
    sim.show()
    sys.exit(app.exec_())
