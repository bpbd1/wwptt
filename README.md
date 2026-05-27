#WWPTT
1. Install Server di VPS Ubuntu/Debian
SSH ke VPS, terus jalanin:bash# Update + install python3
sudo apt update
sudo apt install python3 python3-pip -y

# Bikin folder
mkdir wwptt && cd wwptt

# Buat file server.py, paste code yang tadi gue kasih
nano server.py
# ctrl+shift+v buat paste, ctrl+o enter ctrl+x buat save

# Buka port UDP di firewall
sudo ufw allow 5005/udp
sudo ufw reload

# Jalankan server
python3 server.pyServer bakal jalan dan nunggu HP connect. Biar auto start pas VPS reboot, pakai screen atau systemd. Gue kasih yang gampang pakai screen:bashsudo apt install screen -y
screen -S wwptt
python3 server.py
# tekan ctrl+A lalu D buat lepas dari screenCatat IP VPS lu: curl ifconfig.me2. Buat App Android di Android Studio
Step 1: New ProjectAndroid Studio > New Project > Empty ActivityName: WWPTTLanguage: KotlinMin SDK: 21FinishStep 2: Tambah Permission di AndroidManifest.xml
Buka app/manifests/AndroidManifest.xml, tambah di atas <application>:xml<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />Step 3: Layout activity_main.xml
Ganti isi jadi gini:xml<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="20dp"
    android:gravity="center">

    <EditText
        android:id="@+id/etName"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="Nama lu"/>

    <EditText
        android:id="@+id/etIP"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="IP VPS"
        android:text="GANTI_IP_VPS_LU"/>

    <TextView
        android:id="@+id/tvSpeaker"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Status: Offline"
        android:textSize="18sp"
        android:layout_marginTop="20dp"/>

    <Button
        android:id="@+id/btnPTT"
        android:layout_width="200dp"
        android:layout_height="200dp"
        android:text="TEKAN BUAT NGOMONG"
        android:textSize="16sp"
        android:layout_marginTop="40dp"
        android:backgroundTint="#FF5722"/>

</LinearLayout>Step 4: Full MainActivity.ktkotlinpackage com.wwptt.radio

import android.Manifest
import android.content.pm.PackageManager
import android.media.AudioFormat
import android.media.AudioRecord
import android.media.AudioTrack
import android.media.MediaRecorder
import android.os.Bundle
import android.view.MotionEvent
import android.widget.*
import androidx.appcompat.app.AppCompatActivity
import androidx.core.app.ActivityCompat
import java.net.DatagramPacket
import java.net.DatagramSocket
import java.net.InetAddress
import kotlin.concurrent.thread

class MainActivity : AppCompatActivity() {

    private lateinit var tvSpeaker: TextView
    private lateinit var btnPTT: Button
    private lateinit var etName: EditText
    private lateinit var etIP: EditText

    private var socket: DatagramSocket? = null
    private var isReceiving = false
    private var isRecording = false
    private var myName = ""

    private val sampleRate = 16000
    private val channelConfig = AudioFormat.CHANNEL_OUT_MONO
    private val audioFormat = AudioFormat.ENCODING_PCM_16BIT

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        tvSpeaker = findViewById(R.id.tvSpeaker)
        btnPTT = findViewById(R.id.btnPTT)
        etName = findViewById(R.id.etName)
        etIP = findViewById(R.id.etIP)

        ActivityCompat.requestPermissions(this,
            arrayOf(Manifest.permission.RECORD_AUDIO, Manifest.permission.INTERNET), 1)

        btnPTT.setOnTouchListener { _, event ->
            when (event.action) {
                MotionEvent.ACTION_DOWN -> {
                    myName = etName.text.toString().ifEmpty { "User" }
                    connectToServer()
                    startRecording()
                }
                MotionEvent.ACTION_UP -> stopRecording()
            }
            true
        }
    }

    private fun connectToServer() {
        thread {
            try {
                socket = DatagramSocket()
                val serverIP = etIP.text.toString()
                val serverPort = 5005

                // Kirim REGISTER
                val regData = "REGISTER:$myName".toByteArray()
                val regPacket = DatagramPacket(regData, regData.size,
                    InetAddress.getByName(serverIP), serverPort)
                socket?.send(regPacket)

                isReceiving = true
                startReceiving(serverIP, serverPort)

                runOnUiThread { tvSpeaker.text = "Status: Terhubung" }
            } catch (e: Exception) {
                runOnUiThread { tvSpeaker.text = "Error: ${e.message}" }
            }
        }
    }

    private fun startReceiving(serverIP: String, serverPort: Int) {
        thread {
            val buffer = ByteArray(8192)
            while (isReceiving) {
                try {
                    val packet = DatagramPacket(buffer, buffer.size)
                    socket?.receive(packet)

                    val fullData = String(packet.data, 0, packet.length)
                    val sepIndex = fullData.indexOf('|')
                    if (sepIndex > 0) {
                        val speakerName = fullData.substring(0, sepIndex)
                        val audioData = packet.data.copyOfRange(sepIndex + 1, packet.length)

                        runOnUiThread { tvSpeaker.text = "$speakerName lagi ngomong..." }
                        playAudio(audioData)
                    }
                } catch (e: Exception) {}
            }
        }
    }

    private fun startRecording() {
        isRecording = true
        thread {
            val bufferSize = AudioRecord.getMinBufferSize(
                sampleRate, AudioFormat.CHANNEL_IN_MONO, audioFormat)
            val audioRecord = AudioRecord(
                MediaRecorder.AudioSource.MIC, sampleRate,
                AudioFormat.CHANNEL_IN_MONO, audioFormat, bufferSize)

            audioRecord.startRecording()
            val buffer = ByteArray(bufferSize)

            while (isRecording) {
                val read = audioRecord.read(buffer, 0, buffer.size)
                if (read > 0) {
                    val packet = DatagramPacket(buffer, read,
                        InetAddress.getByName(etIP.text.toString()), 5005)
                    socket?.send(packet)
                }
            }
            audioRecord.stop()
            audioRecord.release()
        }
    }

    private fun stopRecording() {
        isRecording = false
        runOnUiThread { tvSpeaker.text = "Status: Mendengarkan..." }
    }

    private fun playAudio(data: ByteArray) {
        val minBufSize = AudioTrack.getMinBufferSize(
            sampleRate, channelConfig, audioFormat)
        val audioTrack = AudioTrack(
            android.media.AudioManager.STREAM_MUSIC,
            sampleRate, channelConfig, audioFormat,
            minBufSize, AudioTrack.MODE_STREAM)

        audioTrack.play()
        audioTrack.write(data, 0, data.size)
        audioTrack.stop()
        audioTrack.release()
    }

    override fun onDestroy() {
        super.onDestroy()
        isReceiving = false
        socket?.close()
    }
}3. Cara Uji Coba
Server VPS: pastikan python3 server.py masih jalanBuild APK: Android Studio > Build > Build APK > install di 2 HPBuka App di HP1: isi Nama "Budi", isi IP VPS, tekan PTTBuka App di HP2: isi Nama "Ani", isi IP VPS yang samaTes: tekan PTT di HP1, HP2 bakal muncul "Budi lagi ngomong..." + keluar suaraTroubleshoot:Nggak ada suara: cek firewall VPS sudo ufw statusDelay parah: turunin sampleRate jadi 8000 di kedua sisiApp force close: cek permission RECORD_AUDIO udah di-allowMau gue kasih juga versi yang ada tombol pilih frekuensi/channel biar bisa pisah grup?
