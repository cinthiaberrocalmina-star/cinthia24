<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Escáner QR Pro</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/html5-qrcode/2.3.8/html5-qrcode.min.js"></script>
    <style>
        :root {
            --primary: #2563eb;
            --primary-hover: #1d4ed8;
            --background: #f8fafc;
            --surface: #ffffff;
            --text: #0f172a;
            --border: #e2e8f0;
        }

        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
            background-color: var(--background);
            color: var(--text);
            margin: 0;
            padding: 20px;
            display: flex;
            flex-direction: column;
            align-items: center;
        }

        .container {
            max-width: 500px;
            width: 100%;
            background: var(--surface);
            padding: 20px;
            border-radius: 16px;
            box-shadow: 0 4px 6px -1px rgb(0 0 0 / 0.1);
            box-sizing: border-box;
        }

        h1 {
            font-size: 1.5rem;
            text-align: center;
            margin-top: 0;
            margin-bottom: 20px;
        }

        .btn {
            display: block;
            width: 100%;
            padding: 12px;
            background-color: var(--primary);
            color: white;
            border: none;
            border-radius: 8px;
            font-size: 1rem;
            font-weight: 600;
            cursor: pointer;
            transition: background 0.2s;
            margin-bottom: 15px;
        }

        .btn:hover {
            background-color: var(--primary-hover);
        }

        .btn-stop {
            background-color: #dc2626;
        }
        .btn-stop:hover {
            background-color: #b91c1c;
        }

        #reader {
            width: 100%;
            border-radius: 12px;
            overflow: hidden;
            border: 2px solid var(--border);
            background: #000;
            display: none;
        }

        .history-section {
            margin-top: 25px;
        }

        .history-header {
            display: flex;
            justify-content: space-between;
            align-items: center;
            border-bottom: 2px solid var(--border);
            padding-bottom: 8px;
            margin-bottom: 10px;
        }

        .history-title {
            font-weight: 600;
            font-size: 1.1rem;
        }

        .clear-btn {
            background: none;
            border: none;
            color: #64748b;
            cursor: pointer;
            font-size: 0.9rem;
        }

        .clear-btn:hover {
            color: #dc2626;
        }

        .scan-list {
            list-style: none;
            padding: 0;
            margin: 0;
            max-height: 250px;
            overflow-y: auto;
        }

        .scan-item {
            background: var(--background);
            padding: 10px 12px;
            border-radius: 8px;
            margin-bottom: 8px;
            border: 1px solid var(--border);
            font-size: 0.9rem;
            word-break: break-all;
        }

        .scan-time {
            font-size: 0.75rem;
            color: #64748b;
            display: block;
            margin-bottom: 4px;
        }
    </style>
</head>
<body>

<div class="container">
    <h1>Escáner de Códigos QR</h1>
    
    <button id="btn-toggle" class="btn">Activar Cámara</button>
    
    <div id="reader"></div>

    <div class="history-section">
        <div class="history-header">
            <span class="history-title">Registros Guardados</span>
            <button id="btn-clear" class="clear-btn">Limpiar todo</button>
        </div>
        <ul id="scan-list" class="scan-list">
            </ul>
    </div>
</div>

<script>
    const btnToggle = document.getElementById('btn-toggle');
    const btnClear = document.getElementById('btn-clear');
    const readerDiv = document.getElementById('reader');
    const scanList = document.getElementById('scan-list');

    let html5QrCode;
    let isScanning = false;

    // Configuración de la cámara: Usar la cámara trasera por defecto ('environment')
    const config = { fps: 10, qrbox: { width: 250, height: 250 } };

    // Al cargar la página, pintar el historial guardado en localStorage
    document.addEventListener('DOMContentLoaded', loadHistory);

    // Evento para iniciar/detener la cámara
    btnToggle.addEventListener('click', () => {
        if (!isScanning) {
            startScanner();
        } else {
            stopScanner();
        }
    });

    // Evento para limpiar el historial
    btnClear.addEventListener('click', () => {
        if(confirm("¿Estás seguro de que quieres borrar todos los registros?")) {
            localStorage.removeItem('qr_history');
            renderHistory([]);
        }
    });

    function startScanner() {
        readerDiv.style.display = 'block';
        html5QrCode = new Html5Qrcode("reader");
        
        html5QrCode.start(
            { facingMode: "environment" }, // Prioriza la cámara trasera en móviles
            config,
            onScanSuccess
        ).then(() => {
            isScanning = true;
            btnToggle.textContent = "Detener Cámara";
            btnToggle.classList.add('btn-stop');
        }).catch(err => {
            console.error("Error al iniciar la cámara:", err);
            alert("No se pudo acceder a la cámara. Asegúrate de dar los permisos necesarios.");
            readerDiv.style.display = 'none';
        });
    }

    function stopScanner() {
        if (html5QrCode && isScanning) {
            html5QrCode.stop().then(() => {
                isScanning = false;
                btnToggle.textContent = "Activar Cámara";
                btnToggle.classList.remove('btn-stop');
                readerDiv.style.display = 'none';
            }).catch(err => console.error("Error al detener la cámara:", err));
        }
    }

    // Qué pasa cuando encuentra un código QR válido
    function onScanSuccess(decodedText, decodedResult) {
        // Detener la cámara inmediatamente para evitar escaneos duplicados en milisegundos
        stopScanner();

        // Obtener la fecha y hora actual
        const now = new Date();
        const dateTimeStr = now.toLocaleDateString() + ' ' + now.toLocaleTimeString();

        // Crear el nuevo objeto de registro
        const newRecord = {
            data: decodedText,
            timestamp: dateTimeStr
        };

        // Guardar en LocalStorage
        saveToLocalStorage(newRecord);

        // Alerta de éxito y recargar lista
        alert(`¡QR Detectado con éxito!\nContenido: ${decodedText}`);
        loadHistory();
    }

    function saveToLocalStorage(record) {
        let history = JSON.parse(localStorage.getItem('qr_history')) || [];
        // Añadir al principio de la lista para que el más nuevo salga primero
        history.unshift(record); 
        localStorage.setItem('qr_history', JSON.stringify(history));
    }

    function loadHistory() {
        let history = JSON.parse(localStorage.getItem('qr_history')) || [];
        renderHistory(history);
    }

    function renderHistory(history) {
        scanList.innerHTML = '';
        if (history.length === 0) {
            scanList.innerHTML = '<li class="scan-item" style="color: #64748b; text-align:center;">No hay registros guardados.</li>';
            return;
        }

        history.forEach(item => {
            const li = document.createElement('li');
            li.className = 'scan-item';
            li.innerHTML = `
                <span class="scan-time">${item.timestamp}</span>
                <strong>Contenido:</strong> ${escapeHtml(item.data)}
            `;
            scanList.appendChild(li);
        });
    }

    // Función de seguridad para evitar inyección de código si el QR contiene HTML
    function escapeHtml(text) {
        const div = document.createElement('div');
        div.innerText = text;
        return div.innerHTML;
    }
</script>

</body>
</html>
