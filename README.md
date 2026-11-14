# esp32s3_mpy_OV5640

[mpy_cam-v1.27.0-ESP32_GENERIC_S2.zip](https://github.com/cnadler86/micropython-camera-API/releases/download/v0.6.2/mpy_cam-v1.27.0-ESP32_GENERIC_S2.zip)

[Online Flasher](https://esptool.spacehuhn.com)

[Online IDE](https://viper-ide.org)

``` py
import network
import socket
import time
from camera import Camera, FrameSize, PixelFormat

print("Ініціалізація камери для стріму...")
cam = Camera(
    data_pins=[11, 9, 8, 10, 12, 18, 17, 16],
    pclk_pin=13,
    vsync_pin=6,
    href_pin=7,
    xclk_pin=15,
    sda_pin=4,
    scl_pin=5,
    xclk_freq=10000000,
    pixel_format=PixelFormat.RGB565,
    frame_size=FrameSize.QVGA,
)
print("Камера готова!")

ssid = ?
password = ?

sta = network.WLAN(network.STA_IF)
sta.active(True)
sta.connect(ssid, password)

timeout = 15
while not sta.isconnected() and timeout > 0:
    time.sleep(1)
    timeout -= 1

ip_display = sta.ifconfig()[0] if sta.isconnected() else "192.168.4.1"
if not sta.isconnected():
    sta.active(False)
    ap = network.WLAN(network.AP_IF)
    ap.active(True)
    ap.config(essid="ESP32S3-Cam-Test", password="12345678")

print(f"Веб-сервер запущено: http://{ip_display}")

addr = socket.getaddrinfo("0.0.0.0", 80)[0][-1]
s = socket.socket()
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(addr)
s.listen(1)

while True:
    cl = None
    try:
        cl, addr = s.accept()
        request = cl.recv(1024)

        if b"/frame" in request:
            img = bytes(cam.capture())
            cam.free_buffer()
            cl.send(f"HTTP/1.1 200 OK\r\nContent-Type: application/octet-stream\r\nContent-Length: {len(img)}\r\n\r\n")
            cl.send(img)
        else:
            html = """<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>ESP32-S3 Temporal Denoised Stream</title>
    <style>
        body { background: #111; color: #eee; text-align: center; font-family: sans-serif; margin-top: 20px; }
        canvas { 
            border: 2px solid #555; 
            border-radius: 8px; 
            width: 640px; 
            height: 480px; 
            image-rendering: auto; 
            background: #000;
        }
        #fps { color: #aaa; font-size: 16px; margin-top: 10px; font-weight: bold; }
    </style>
</head>
<body>
    <h2>ESP32-S3 Temporal Denoised Stream (2x)</h2>
    <canvas id="cv" width="320" height="240"></canvas>
    <div id="fps">FPS: --</div>
    <script>
        const cv = document.getElementById('cv');
        const ctx = cv.getContext('2d');
        const imgData = ctx.createImageData(320, 240);
        
        // Буфер для збереження попереднього згладженого кадру (Temporal Buffer)
        let prevBuffer = null;
        
        let lastTime = performance.now();
        let frameCount = 0;

        function fetchFrame() {
            fetch('/frame?t=' + Date.now())
                .then(r => r.arrayBuffer())
                .then(buf => {
                    const data = new DataView(buf);
                    const width = 320;
                    const height = 240;
                    const totalPixels = width * height;
                    
                    // Якщо буфер попереднього кадру ще не створено — ініціалізуємо нулями
                    if (!prevBuffer) {
                        prevBuffer = new Float32Array(totalPixels * 3); // зберігаємо R, G, B для кожного пікселя
                    }

                    const pixels = imgData.data;
                    const alpha = 0.4; // Коефіцієнт згладжування: чим ближче до 0, тим сильніше гаситься шум за рахунок попередніх кадрів

                    for (let i = 0; i < totalPixels; i++) {
                        const rgb565 = data.getUint16(i * 2, false);
                        
                        // Поточні значення кольору
                        let curR = ((rgb565 >> 11) & 0x1F) * 8.225;
                        let curG = ((rgb565 >> 5) & 0x3F) * 4.047;
                        let curB = (rgb565 & 0x1F) * 8.225;

                        let idx3 = i * 3;
                        let idx4 = i * 4;

                        if (frameCount === 0 && prevBuffer[idx3] === 0) {
                            prevBuffer[idx3]     = curR;
                            prevBuffer[idx3 + 1] = curG;
                            prevBuffer[idx3 + 2] = curB;
                        }

                        // Часовий фільтр (Temporal IIR Filter): змішуємо новий кадр з попереднім
                        let r = prevBuffer[idx3]     * (1 - alpha) + curR * alpha;
                        let g = prevBuffer[idx3 + 1] * (1 - alpha) + curG * alpha;
                        let b = prevBuffer[idx3 + 2] * (1 - alpha) + curB * alpha;

                        // Зберігаємо поточний стан у буфері для наступного кадру
                        prevBuffer[idx3]     = r;
                        prevBuffer[idx3 + 1] = g;
                        prevBuffer[idx3 + 2] = b;

                        // Записуємо в рендер canvas
                        pixels[idx4]     = r;
                        pixels[idx4 + 1] = g;
                        pixels[idx4 + 2] = b;
                        pixels[idx4 + 3] = 255;
                    }

                    ctx.putImageData(imgData, 0, 0);

                    // Підрахунок FPS
                    frameCount++;
                    let now = performance.now();
                    if (now - lastTime >= 1000) {
                        document.getElementById('fps').innerText = 'FPS: ' + frameCount;
                        frameCount = 0;
                        lastTime = now;
                    }

                    requestAnimationFrame(fetchFrame);
                })
                .catch(err => {
                    setTimeout(fetchFrame, 500);
                });
        }

        fetchFrame();
    </script>
</body>
</html>"""
            cl.send("HTTP/1.1 200 OK\r\nContent-Type: text/html; charset=utf-8\r\n\r\n")
            cl.send(html)
    except Exception as e:
        pass
    finally:
        if cl:
            cl.close()
```



![](https://diyables.io/images/products/esp32-s3-wroom-n16r8-cam-development-board-ov5640-3.jpg)
