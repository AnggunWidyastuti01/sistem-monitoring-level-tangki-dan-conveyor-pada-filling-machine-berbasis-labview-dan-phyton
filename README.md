import sys
import serial
import serial.tools.list_ports
from PyQt6.QtWidgets import (QApplication, QMainWindow, QWidget, QVBoxLayout, 
                           QHBoxLayout, QComboBox, QPushButton, QTabWidget, 
                           QLabel, QSpinBox, QMessageBox, QGroupBox)
from PyQt6.QtCore import QTimer, pyqtSlot, Qt
from PyQt6.QtGui import QPen
import pyqtgraph as pg
from datetime import datetime
import numpy as np
import traceback
import pyodbc

class SerialControlGUI(QMainWindow):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Kontrol Serial ESP32")
        self.setGeometry(100, 100, 800, 600)
        
        # Inisialisasi variabel
        self.serial_port = None
        self.connected = False
        self.sensor_data = []
        self.timestamp_data = []
        self.last_timestamp = None  # Menyimpan timestamp terakhir
        self.data_gap_threshold = 5  # Threshold untuk gap data (dalam detik)
        
        # Setup UI
        self.setup_ui()
        
        # Timer untuk update port dan grafik
        self.port_timer = QTimer()
        self.port_timer.timeout.connect(self.update_ports)
        self.port_timer.start(1000)
        
        self.plot_timer = QTimer()
        self.plot_timer.timeout.connect(self.update_plot)
        
    def setup_ui(self):
        central_widget = QWidget()
        self.setCentralWidget(central_widget)
        layout = QVBoxLayout(central_widget)
        
        # Grup Koneksi
        connection_group = QGroupBox("Pengaturan Koneksi")
        connection_layout = QHBoxLayout()
        
        # ComboBox Port
        self.port_combo = QComboBox()
        self.update_ports()
        connection_layout.addWidget(QLabel("Port:"))
        connection_layout.addWidget(self.port_combo)
        
        # ComboBox Baudrate
        self.baud_combo = QComboBox()
        self.baud_combo.addItems(['9600', '115200'])
        connection_layout.addWidget(QLabel("Baudrate:"))
        connection_layout.addWidget(self.baud_combo)
        
        # Tombol Connect
        self.connect_button = QPushButton("Connect")
        self.connect_button.clicked.connect(self.toggle_connection)
        connection_layout.addWidget(self.connect_button)
        
        connection_group.setLayout(connection_layout)
        layout.addWidget(connection_group)
        
        # Tab Widget
        self.tab_widget = QTabWidget()
        
        # Tab Sensor
        sensor_tab = QWidget()
        sensor_layout = QVBoxLayout()
        
        # Grup Kontrol Sensor
        sensor_control_group = QGroupBox("Kontrol Sensor")
        sensor_control_layout = QHBoxLayout()
        
        # ComboBox Jenis Sensor
        self.sensor_type_combo = QComboBox()
        self.sensor_type_combo.addItems(['Ultrasonic', 'Infrared'])
        sensor_control_layout.addWidget(QLabel("Jenis Sensor:"))
        sensor_control_layout.addWidget(self.sensor_type_combo)
        
        # SpinBox Jumlah Pembacaan
        self.reading_count = QSpinBox()
        self.reading_count.setRange(1, 100)
        sensor_control_layout.addWidget(QLabel("Jumlah:"))
        sensor_control_layout.addWidget(self.reading_count)
        
        # SpinBox Delay
        self.sensor_delay = QSpinBox()
        self.sensor_delay.setRange(100, 5000)
        self.sensor_delay.setSingleStep(100)
        self.sensor_delay.setValue(1000)
        sensor_control_layout.addWidget(QLabel("Delay (ms):"))
        sensor_control_layout.addWidget(self.sensor_delay)
        
        # Tombol Baca Sensor
        self.read_sensor_button = QPushButton("Baca Sensor")
        self.read_sensor_button.clicked.connect(self.read_sensor)
        sensor_control_layout.addWidget(self.read_sensor_button)
        
        sensor_control_group.setLayout(sensor_control_layout)
        sensor_layout.addWidget(sensor_control_group)

        # Grup Status Sensor
        sensor_status_group = QGroupBox("Status Sensor")
        sensor_status_layout = QVBoxLayout()
        
        # Label untuk status sensor
        self.ultrasonik_status = QLabel("Sensor Ultrasonic: -")
        self.infrared_status = QLabel("Sensor Infrared: -")
        self.ultrasonik_status.setStyleSheet("font-size: 12pt; font-weight: bold;")
        self.infrared_status.setStyleSheet("font-size: 12pt; font-weight: bold;")
        
        # Tambahkan label untuk status string ultrasonic
        self.ultrasonik_string_status = QLabel("Status: -")
        self.ultrasonik_string_status.setStyleSheet("font-size: 12pt; font-weight: bold; color: blue;")
        
        sensor_status_layout.addWidget(self.ultrasonik_status)
        sensor_status_layout.addWidget(self.ultrasonik_string_status)
        sensor_status_layout.addWidget(self.infrared_status)
        sensor_status_group.setLayout(sensor_status_layout)
        sensor_layout.addWidget(sensor_status_group)
        
        # Grafik
        self.plot_widget = pg.PlotWidget()
        self.plot_widget.setBackground('w')
        self.plot_widget.setTitle("Data Sensor Real-time")
        self.plot_widget.setLabel('left', 'Jarak (cm)')
        self.plot_widget.setLabel('bottom', 'Waktu (HH:MM:SS)')
        self.plot_widget.showGrid(x=True, y=True)
        self.plot_curve = self.plot_widget.plot(pen='b', name='Ultrasonic')
        sensor_layout.addWidget(self.plot_widget)
        
        sensor_tab.setLayout(sensor_layout)
        
        # Tab Aktuator
        actuator_tab = QWidget()
        actuator_layout = QVBoxLayout()
        
        # Grup Kontrol Aktuator
        actuator_control_group = QGroupBox("Kontrol Aktuator")
        actuator_control_layout = QHBoxLayout()
        
        # ComboBox Jenis Aktuator
        self.actuator_type_combo = QComboBox()
        self.actuator_type_combo.addItems(['Motor', 'Pompa'])
        actuator_control_layout.addWidget(QLabel("Jenis Aktuator:"))
        actuator_control_layout.addWidget(self.actuator_type_combo)
        
        # ComboBox Aksi
        self.action_combo = QComboBox()
        self.actuator_type_combo.currentTextChanged.connect(self.update_action_combo)
        actuator_control_layout.addWidget(QLabel("Aksi:"))
        actuator_control_layout.addWidget(self.action_combo)
        
        # SpinBox Delay
        self.actuator_delay = QSpinBox()
        self.actuator_delay.setRange(100, 5000)
        self.actuator_delay.setSingleStep(100)
        self.actuator_delay.setValue(1000)
        actuator_control_layout.addWidget(QLabel("Delay (ms):"))
        actuator_control_layout.addWidget(self.actuator_delay)
        
        # Tombol Kontrol Aktuator
        self.control_actuator_button = QPushButton("Kontrol Aktuator")
        self.control_actuator_button.clicked.connect(self.control_actuator)
        actuator_control_layout.addWidget(self.control_actuator_button)
        
        actuator_control_group.setLayout(actuator_control_layout)
        actuator_layout.addWidget(actuator_control_group)

        # Grup Status Aktuator
        actuator_status_group = QGroupBox("Status Aktuator")
        actuator_status_layout = QVBoxLayout()
        
        # Label untuk status aktuator
        self.motor_status = QLabel("Motor: Berhenti")
        self.pompa_status = QLabel("Pompa: Mati")
        self.motor_status.setStyleSheet("font-size: 12pt; font-weight: bold;")
        self.pompa_status.setStyleSheet("font-size: 12pt; font-weight: bold;")
        
        actuator_status_layout.addWidget(self.motor_status)
        actuator_status_layout.addWidget(self.pompa_status)
        actuator_status_group.setLayout(actuator_status_layout)
        actuator_layout.addWidget(actuator_status_group)
        
        actuator_tab.setLayout(actuator_layout)
        
        # Tambahkan tab ke widget
        self.tab_widget.addTab(sensor_tab, "Sensor")
        self.tab_widget.addTab(actuator_tab, "Aktuator")
        layout.addWidget(self.tab_widget)
        
        # Update combo aksi pertama kali
        self.update_action_combo()
        
    def update_ports(self):
        """Update daftar port yang tersedia"""
        available_ports = [port.device for port in serial.tools.list_ports.comports()]
        current_text = self.port_combo.currentText()
        
        self.port_combo.clear()
        self.port_combo.addItems(available_ports)
        
        if current_text in available_ports:
            self.port_combo.setCurrentText(current_text)
            
    def update_action_combo(self):
        """Update pilihan aksi berdasarkan jenis aktuator"""
        self.action_combo.clear()
        self.action_combo.addItems(['Hidup', 'Mati'])
            
    def toggle_connection(self):
        """Menghubungkan atau memutuskan koneksi serial"""
        if not self.connected:
            try:
                port = self.port_combo.currentText()
                if not port:
                    QMessageBox.warning(self, "Peringatan", "Silakan pilih port terlebih dahulu")
                    return
                    
                baudrate = int(self.baud_combo.currentText())
                self.serial_port = serial.Serial(port, baudrate, timeout=1)
                self.connected = True
                self.connect_button.setText("Disconnect")
                self.plot_timer.start(100)
                QMessageBox.information(self, "Sukses", "Berhasil terhubung ke port serial")
            except serial.SerialException as e:
                self.connected = False
                self.serial_port = None
                QMessageBox.critical(self, "Error", f"Gagal terhubung ke port serial: {str(e)}\n\nPastikan:\n1. Port COM tidak digunakan aplikasi lain\n2. ESP32 terhubung dengan benar\n3. Port COM yang dipilih benar")
            except Exception as e:
                self.connected = False
                self.serial_port = None
                QMessageBox.critical(self, "Error", f"Gagal terhubung ke port serial: {str(e)}")
        else:
            try:
                self.plot_timer.stop()
                if self.serial_port:
                    self.serial_port.close()
                self.connected = False
                self.serial_port = None
                self.connect_button.setText("Connect")
                QMessageBox.information(self, "Sukses", "Berhasil memutuskan koneksi serial")
            except Exception as e:
                QMessageBox.critical(self, "Error", f"Gagal memutuskan koneksi: {str(e)}")
                
    def read_sensor(self):
        """Mengirim perintah untuk membaca sensor"""
        if not self.connected:
            QMessageBox.warning(self, "Peringatan", "Silakan hubungkan port serial terlebih dahulu")
            return
            
        try:
            jenis = self.sensor_type_combo.currentText()
            jumlah = self.reading_count.value()
            delay = self.sensor_delay.value()
            
            print(f"[DEBUG] Mengirim perintah sensor: jenis={jenis}, jumlah={jumlah}, delay={delay}")
            
            # Reset status sensor sesuai jenisnya
            if jenis == "Ultrasonic":
                # Hapus clear data agar data sebelumnya tetap tersimpan
                self.ultrasonik_status.setText("Sensor Ultrasonic: Memulai pembacaan...")
                print("[DEBUG] Status ultrasonic direset")
            else:
                self.infrared_status.setText("Sensor Infrared: Memulai pembacaan...")
            
            command = f"Sensor:{jenis}:{jumlah}:{delay}\n"
            print(f"[DEBUG] Command yang dikirim: {command.strip()}")
            self.serial_port.write(command.encode())
            print("[DEBUG] Command berhasil dikirim")
        except Exception as e:
            print(f"[DEBUG] Error dalam read_sensor: {str(e)}")
            QMessageBox.critical(self, "Error", f"Gagal mengirim perintah: {str(e)}")
            
    def control_actuator(self):
        """Mengirim perintah untuk mengontrol aktuator"""
        if not self.connected:
            QMessageBox.warning(self, "Peringatan", "Silakan hubungkan port serial terlebih dahulu")
            return
            
        try:
            device = self.actuator_type_combo.currentText()
            aksi = self.action_combo.currentText()
            delay = self.actuator_delay.value()
            
            command = f"Aktuator:{device}:{aksi}:{delay}\n"
            self.serial_port.write(command.encode())
        except Exception as e:
            QMessageBox.critical(self, "Error", f"Gagal mengirim perintah: {str(e)}")
            
    def update_plot(self):
        """Update grafik dengan data terbaru"""
        if not self.connected or not self.serial_port:
            print("[DEBUG] Tidak terkoneksi atau port serial tidak tersedia")
            return
            
        try:
            if self.serial_port.in_waiting:
                line = self.serial_port.readline().decode().strip()
                print(f"[DEBUG] Data diterima: {line}")
                
                # Debug untuk melihat apakah string mengandung "Ultrasonik"
                print(f"[DEBUG] Apakah mengandung 'Ultrasonik': {'Ultrasonik' in line}")
                print(f"[DEBUG] Apakah mengandung 'Ultrasonic': {'Ultrasonic' in line}")
                
                if "Ultrasonik" in line or "Ultrasonic" in line:  # Mencoba kedua kemungkinan
                    try:
                        print(f"[DEBUG] Memproses data ultrasonic: {line}")
                        
                        # Parse nilai dengan lebih teliti
                        parts = line.split(":")
                        print(f"[DEBUG] Parts setelah split: {parts}")
                        
                        if len(parts) >= 2:
                            value_str = parts[1].strip().split()[0]  # Ambil angka sebelum "cm"
                            print(f"[DEBUG] Value string sebelum parsing: {value_str}")
                            
                            try:
                                value = float(value_str)
                                print(f"[DEBUG] Nilai ultrasonic yang di-parse: {value}")
                                
                                current_timestamp = datetime.now().timestamp()
                                
                                # Cek apakah ada gap yang terlalu besar
                                if self.last_timestamp is not None:
                                    time_diff = current_timestamp - self.last_timestamp
                                    if time_diff > self.data_gap_threshold:
                                        print(f"[DEBUG] Gap terdeteksi: {time_diff} detik")
                                        # Tambahkan titik kosong untuk menunjukkan gap
                                        self.sensor_data.append(None)
                                        self.timestamp_data.append(current_timestamp - 0.1)
                                
                                # Update data
                                self.sensor_data.append(value)
                                self.timestamp_data.append(current_timestamp)
                                self.last_timestamp = current_timestamp
                                
                                print(f"[DEBUG] Data ditambahkan: {value} cm")
                                print(f"[DEBUG] Jumlah data saat ini: {len(self.sensor_data)}")
                                
                                # Update status ultrasonik dengan QTimer untuk memastikan update di thread utama
                                QTimer.singleShot(0, lambda: self.ultrasonik_status.setText(f"Sensor Ultrasonic: {value:.1f} cm"))
                                print("[DEBUG] Status ultrasonic diupdate")
                                
                                # Update status string berdasarkan nilai
                                status_text = ""
                                if value < 10:
                                    status_text = "Sangat Dekat"
                                elif value < 30:
                                    status_text = "Dekat"
                                elif value < 50:
                                    status_text = "Sedang"
                                elif value < 100:
                                    status_text = "Jauh"
                                else:
                                    status_text = "Sangat Jauh"
                                
                                QTimer.singleShot(0, lambda: self.ultrasonik_string_status.setText(f"Status: {status_text}"))
                                print("[DEBUG] Status string diupdate")
                                
                                # Update grafik dengan QTimer untuk memastikan update di thread utama
                                QTimer.singleShot(0, self.update_graph)
                                print("[DEBUG] Grafik diupdate")
                                
                                # Simpan data ke database
                                self.save_to_database(value, "Terdeteksi", self.motor_status.text(), self.pompa_status.text())
                                
                            except ValueError as ve:
                                print(f"[DEBUG] Error konversi ke float: {str(ve)}")
                                raise
                    except (ValueError, IndexError) as e:
                        print(f"[DEBUG] Error parsing ultrasonik data: {line}, Error: {str(e)}")
                        print(f"[DEBUG] Stack trace: {traceback.format_exc()}")
                elif "Sensor Infrared:" in line:
                    try:
                        # Cek apakah mengandung kata "Tidak Terdeteksi"
                        is_detected = "Tidak Terdeteksi" not in line
                        status = "Terdeteksi" if is_detected else "Tidak Terdeteksi"
                        current_time = datetime.now().strftime("%H:%M:%S")
                        
                        # Set warna teks sesuai status
                        color = "green" if is_detected else "red"
                        self.infrared_status.setStyleSheet(f"font-size: 12pt; font-weight: bold; color: {color}")
                        self.infrared_status.setText(f"Sensor Infrared: {status} ({current_time})")
                        
                        print(f"[DEBUG] Status Infrared: {status}")  # Debug print
                        
                        # Simpan data ke database
                        self.save_to_database(None, status, self.motor_status.text(), self.pompa_status.text())
                        
                    except Exception as e:
                        print(f"[DEBUG] Error parsing infrared data: {line}, Error: {str(e)}")
                elif "Motor" in line:
                    status = "Hidup" if "Hidup" in line else "Mati"
                    if "kembali ke kondisi semula" not in line:
                        self.motor_status.setText(f"Motor: {status}")
                    else:
                        self.motor_status.setText("Motor: Mati")
                        
                    # Simpan data ke database
                    self.save_to_database(None, self.infrared_status.text().split(": ")[1], f"Motor: {status}", self.pompa_status.text())
                    
                elif "Pompa" in line:
                    status = "Hidup" if "Hidup" in line else "Mati"
                    if "kembali ke kondisi semula" not in line:
                        self.pompa_status.setText(f"Pompa: {status}")
                    else:
                        self.pompa_status.setText("Pompa: Mati")
                        
                    # Simpan data ke database
                    self.save_to_database(None, self.infrared_status.text().split(": ")[1], self.motor_status.text(), f"Pompa: {status}")
                    
        except serial.SerialException as e:
            self.plot_timer.stop()
            self.connected = False
            if self.serial_port:
                try:
                    self.serial_port.close()
                except:
                    pass
            self.serial_port = None
            self.connect_button.setText("Connect")
            QMessageBox.critical(self, "Error", f"Koneksi terputus: {str(e)}")
        except Exception as e:
            print(f"[DEBUG] Error updating plot: {str(e)}")

    def update_graph(self):
        """Update grafik dengan data terbaru"""
        if len(self.sensor_data) > 0:
            try:
                # Konversi timestamp ke format waktu yang lebih mudah dibaca
                x_data = np.array(self.timestamp_data)
                y_data = np.array(self.sensor_data, dtype=float)  # Konversi ke float
                
                # Buat plot dengan garis putus-putus untuk gap data
                self.plot_widget.clear()
                
                # Plot data normal
                valid_data = ~np.isnan(y_data)
                if np.any(valid_data):
                    self.plot_widget.plot(x=x_data[valid_data], y=y_data[valid_data], 
                                        pen='b', name='Ultrasonic')
                
                # Plot gap data dengan garis putus-putus
                gap_data = np.isnan(y_data)
                if np.any(gap_data):
                    self.plot_widget.plot(x=x_data[gap_data], y=y_data[gap_data], 
                                        pen=pg.mkPen('b', style=Qt.PenStyle.DashLine), name='Gap')
                
                # Set label sumbu X ke format waktu
                self.plot_widget.setLabel('bottom', 'Waktu', units='HH:MM:SS')
                
                # Konversi timestamp ke format waktu yang mudah dibaca
                time_axis = pg.DateAxisItem(orientation='bottom')
                self.plot_widget.setAxisItems({'bottom': time_axis})
                
                # Auto-scale Y axis dengan margin
                y_min = np.nanmin(y_data) if len(y_data) > 0 else 0
                y_max = np.nanmax(y_data) if len(y_data) > 0 else 100
                margin = (y_max - y_min) * 0.1  # 10% margin
                self.plot_widget.setYRange(y_min - margin, y_max + margin)
                
                # Auto-scale X axis
                if len(x_data) > 0:
                    self.plot_widget.setXRange(x_data[0], x_data[-1] + 1)
                
                # Force update
                self.plot_widget.replot()
                print("[DEBUG] Grafik berhasil diupdate")
            except Exception as e:
                print(f"[DEBUG] Error saat update grafik: {str(e)}")
                print(f"[DEBUG] Stack trace: {traceback.format_exc()}")

    def connect_db(self):
        try:
            # Tambahkan Trusted_Connection=yes untuk Windows Authentication
            conn_str = (
                'DRIVER={SQL Server};'
                'SERVER=LAPTOP-8OUR0KIE\\MSSQLSERVER01;'
                'DATABASE=AnggunWidyastuti;'
                'Trusted_Connection=yes;'  # Menggunakan Windows Authentication
            )
            
            print("[DEBUG] Mencoba koneksi ke database...")
            conn = pyodbc.connect(conn_str)
            print("[DEBUG] Koneksi database berhasil!")
            return conn
        except Exception as e:
            print(f"[DEBUG] Error koneksi database: {str(e)}")
            self.show_message("Koneksi DB Gagal", 
                f"Gagal terhubung ke database:\n{str(e)}\n\n"
                "Pastikan:\n"
                "1. SQL Server sudah berjalan\n"
                "2. Database 'AnggunWidyastuti' sudah ada\n"
                "3. Windows Authentication sudah diaktifkan di SQL Server")
            return None

    def save_to_database(self, ultrasonic_value, infrared_status, motor_status, pompa_status):
        conn = self.connect_db()
        if conn:
            try:
                cursor = conn.cursor()
                query = """
                    INSERT INTO SensorAktuatorLog (
                        ultrasonic, infrared_status, motor_status, pompa_status, timestamp
                    ) VALUES (?, ?, ?, ?, GETDATE())
                """
                cursor.execute(query, (
                    ultrasonic_value,
                    infrared_status,
                    motor_status,
                    pompa_status
                ))
                conn.commit()
                cursor.close()
                conn.close()
                print(f"[DEBUG] Data berhasil disimpan ke database: {ultrasonic_value}, {infrared_status}, {motor_status}, {pompa_status}")
            except Exception as e:
                print(f"[DEBUG] Error saat menyimpan ke database: {str(e)}")
                self.show_message("Gagal Simpan DB", str(e))

    def show_message(self, title, message):
        QMessageBox.critical(self, title, message)
    
if __name__ == '__main__':
    app = QApplication(sys.argv)
    window = SerialControlGUI()
    window.show()
    sys.exit(app.exec()) 
