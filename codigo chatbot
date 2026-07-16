import os
import sys
import json
import time
import hmac
import hashlib
import logging
import sqlite3
import threading
from datetime import datetime, timezone
from flask import Flask, request, jsonify
import requests
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()  # Lee el archivo .env (si existe) y carga sus variables al entorno

# Configuración de codificación para evitar errores al imprimir emojis en Windows
if sys.platform.startswith('win'):
    try:
        sys.stdout.reconfigure(encoding='utf-8', errors='backslashreplace')
        sys.stderr.reconfigure(encoding='utf-8', errors='backslashreplace')
    except AttributeError:
        pass

# ==========================================
# 📝 LOGGING
# ==========================================
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    handlers=[
        logging.StreamHandler(),
        logging.FileHandler("cess_bot.log", encoding="utf-8"),
    ],
)
log = logging.getLogger("cess_bot")

app = Flask(__name__)

# ==========================================
# ⚙️ CONFIGURACIÓN DE CREDENCIALES
# ==========================================
API_VERSION = "v25.0"

VARIABLES_REQUERIDAS = [
    "OPENAI_API_KEY",
    "META_TOKEN",
    "PHONE_NUMBER_ID",
    "NUMERO_ASESOR",
    "META_APP_SECRET",
   
]

faltantes = [v for v in VARIABLES_REQUERIDAS if not os.environ.get(v)]
if faltantes:
    log.error(f"❌ Faltan variables de entorno obligatorias: {', '.join(faltantes)}")
    sys.exit(1)

OPENAI_API_KEY = os.environ["OPENAI_API_KEY"]
META_TOKEN = os.environ["META_TOKEN"]
PHONE_NUMBER_ID = os.environ["PHONE_NUMBER_ID"]
VERIFY_TOKEN = os.environ.get("VERIFY_TOKEN", "vibecode")
NUMERO_ASESOR = os.environ["NUMERO_ASESOR"]
META_APP_SECRET = os.environ["META_APP_SECRET"]


client = OpenAI(api_key=OPENAI_API_KEY)

# ==========================================
# 🧠 HISTORIAL DE CONVERSACIÓN (SQLite, ventana deslizante)
# ==========================================
VENTANA_HISTORIAL = int(os.environ.get("VENTANA_HISTORIAL", "10"))
MAX_GUARDADOS_POR_NUMERO = 30
DIAS_RETENCION_HISTORIAL = int(os.environ.get("DIAS_RETENCION_HISTORIAL", "90"))

DB_PATH = os.environ.get("HISTORIAL_DB_PATH", "historial_cess.db")

# Lock para serializar escrituras concurrentes a SQLite (defensa extra junto con WAL)
_db_lock = threading.Lock()


def conectar():
    con = sqlite3.connect(DB_PATH, timeout=10)
    con.execute("PRAGMA journal_mode=WAL")
    con.execute("PRAGMA busy_timeout=10000")
    return con


def inicializar_db():
    con = conectar()
    con.execute("""
        CREATE TABLE IF NOT EXISTS historial (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            numero TEXT NOT NULL,
            rol TEXT NOT NULL,
            contenido TEXT NOT NULL,
            creado_en TEXT NOT NULL
        )
    """)
    con.execute("CREATE INDEX IF NOT EXISTS idx_historial_numero ON historial(numero)")

    # Tabla de deduplicación de mensajes entrantes (evita reprocesar reintentos de Meta)
    con.execute("""
        CREATE TABLE IF NOT EXISTS mensajes_procesados (
            wamid TEXT PRIMARY KEY,
            procesado_en TEXT NOT NULL
        )
    """)
    con.commit()
    con.close()


def guardar_mensaje(numero: str, rol: str, contenido: str):
    with _db_lock:
        con = conectar()
        con.execute(
            "INSERT INTO historial (numero, rol, contenido, creado_en) VALUES (?, ?, ?, ?)",
            (numero, rol, contenido, datetime.now(timezone.utc).isoformat()),
        )
        con.commit()
        con.close()


def obtener_historial(numero: str, limite: int = None):
    """Devuelve los últimos `limite` mensajes de ese número, en orden cronológico,
    listos para pasarse como `input` a la Responses API."""
    if limite is None:
        limite = VENTANA_HISTORIAL
    con = conectar()
    filas = con.execute(
        "SELECT rol, contenido FROM historial WHERE numero = ? ORDER BY id DESC LIMIT ?",
        (numero, limite),
    ).fetchall()
    con.close()
    filas.reverse()  # de más viejo a más nuevo
    return [{"role": rol, "content": contenido} for rol, contenido in filas]


def limpiar_historial_antiguo(numero: str):
    """Conserva solo los últimos MAX_GUARDADOS_POR_NUMERO registros de ese número
    y además borra cualquier registro (de cualquier número) más viejo que
    DIAS_RETENCION_HISTORIAL días."""
    with _db_lock:
        con = conectar()
        con.execute(
            """
            DELETE FROM historial
            WHERE numero = ? AND id NOT IN (
                SELECT id FROM historial WHERE numero = ? ORDER BY id DESC LIMIT ?
            )
            """,
            (numero, numero, MAX_GUARDADOS_POR_NUMERO),
        )
        con.execute(
            "DELETE FROM historial WHERE creado_en < datetime('now', ?)",
            (f"-{DIAS_RETENCION_HISTORIAL} days",),
        )
        con.commit()
        con.close()


def ya_procesado(wamid: str) -> bool:
    if not wamid:
        return False
    con = conectar()
    fila = con.execute(
        "SELECT 1 FROM mensajes_procesados WHERE wamid = ?", (wamid,)
    ).fetchone()
    con.close()
    return fila is not None


def marcar_procesado(wamid: str):
    if not wamid:
        return
    with _db_lock:
        con = conectar()
        try:
            con.execute(
                "INSERT INTO mensajes_procesados (wamid, procesado_en) VALUES (?, ?)",
                (wamid, datetime.now(timezone.utc).isoformat()),
            )
            con.commit()
        except sqlite3.IntegrityError:
            pass  # ya estaba marcado (carrera entre hilos), no pasa nada
        con.close()


def limpiar_mensajes_procesados_antiguos():
    """Evita que la tabla de deduplicación crezca sin límite."""
    with _db_lock:
        con = conectar()
        con.execute(
            "DELETE FROM mensajes_procesados WHERE procesado_en < datetime('now', '-7 days')"
        )
        con.commit()
        con.close()


inicializar_db()

# ==========================================
# 📄 CONTEXTO PRIVADO (hoja de contexto del chatbot)
# ==========================================
RUTA_CONTEXTO = os.environ.get("RUTA_CONTEXTO", "CESS_hoja_contexto_chatbot.md")

_contexto_cache = {"texto": "", "mtime": None}


def cargar_contexto():
    """Carga la hoja de contexto y la vuelve a leer del disco si el archivo
    cambió, sin necesidad de reiniciar el servidor."""
    try:
        mtime_actual = os.path.getmtime(RUTA_CONTEXTO)
        if _contexto_cache["mtime"] != mtime_actual:
            with open(RUTA_CONTEXTO, "r", encoding="utf-8") as f:
                _contexto_cache["texto"] = f.read()
            _contexto_cache["mtime"] = mtime_actual
            log.info(f"📄 Hoja de contexto (re)cargada desde {RUTA_CONTEXTO}")
    except FileNotFoundError:
        log.error(f"⚠️ Archivo de contexto no encontrado: {RUTA_CONTEXTO}")
        _contexto_cache["texto"] = _contexto_cache["texto"] or ""
    return _contexto_cache["texto"]


# ==========================================
# 🔧 DEFINICIÓN DE LA FUNCIÓN DE TRASPASO (Responses API)
# ==========================================
tools_traspaso = [
    {
        "type": "function",
        "name": "notificar_traspaso",
        "description": (
            "Notifica a un asesor humano que un prospecto está siendo transferido. "
            "Llámala junto con tu respuesta normal al cliente, nunca en lugar de ella."
        ),
        "parameters": {
            "type": "object",
            "properties": {
                "tipo": {
                    "type": "string",
                    "enum": ["listo_para_inscribir", "duda_sin_resolver", "tramite_administrativo"],
                    "description": "Motivo del traspaso.",
                },
                "programa": {
                    "type": "string",
                    "description": "Programa de interés del prospecto, si se conoce.",
                },
                "resumen": {
                    "type": "string",
                    "description": "1 línea de contexto para el asesor.",
                },
            },
            "required": ["tipo", "resumen"],
        },
    }
]

MENSAJES_RESPALDO = {
    "listo_para_inscribir": "¡Perfecto! Para continuar con tu inscripción, comunícate al número 6144150015 con el mensaje \"estoy listo para la inscripcion\" o haz clic en el siguiente enlace: https://wa.me/526144150015?text=estoy%20listo%20para%20la%20inscripcion 🙂",
    "duda_sin_resolver": "En un momento te atiende un asesor para resolver tu duda 🙂",
    "tramite_administrativo": "En un momento te atiende un asesor para ayudarte con ese trámite 🙂",
}
MENSAJE_RESPALDO_GENERICO = "En un momento te atiende un asesor 🙂"
MENSAJE_ERROR_TECNICO = "Disculpa, tuve un problema técnico. En un momento te contacta un asesor 🙂"


@app.route('/', methods=['GET'])
def inicio():
    return "¡Servidor de WhatsApp e IA activo correctamente!", 200


@app.route('/webhook', methods=['GET'])
def verificar_webhook():
    mode = request.args.get('hub.mode')
    token = request.args.get('hub.verify_token')
    challenge = request.args.get('hub.challenge')

    if mode and token:
        if mode == 'subscribe' and token == VERIFY_TOKEN:
            return challenge, 200
        return 'Validación fallida', 403
    return 'Mal formato', 400


def firma_valida(payload_bytes: bytes, signature_header: str) -> bool:
    """Verifica que el POST realmente venga de Meta, usando el header
    X-Hub-Signature-256 y el App Secret de la app de Meta."""
    if not signature_header or not signature_header.startswith("sha256="):
        return False
    firma_recibida = signature_header.split("sha256=", 1)[1]
    firma_esperada = hmac.new(
        META_APP_SECRET.encode("utf-8"), payload_bytes, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(firma_esperada, firma_recibida)


@app.route('/webhook', methods=['POST'])
def recibir_webhook():
    """Valida la firma, responde 200 de inmediato y procesa el payload en
    un hilo aparte para no bloquear a Meta (que reintenta si tardamos)."""
    firma_header = request.headers.get("X-Hub-Signature-256", "")
    if not firma_valida(request.get_data(), firma_header):
        log.warning("⚠️ Webhook recibido con firma inválida — descartado.")
        return jsonify({"status": "invalid signature"}), 403

    data = request.get_json(silent=True) or {}
    threading.Thread(target=procesar_payload, args=(data,), daemon=True).start()
    return jsonify({"status": "success"}), 200


def marcar_como_leido(message_id: str, phone_number_id: str):
    if not message_id:
        return
    url = f"https://graph.facebook.com/{API_VERSION}/{phone_number_id}/messages"
    headers = {"Authorization": f"Bearer {META_TOKEN}", "Content-Type": "application/json"}
    payload = {"messaging_product": "whatsapp", "status": "read", "message_id": message_id}
    try:
        requests.post(url, json=payload, headers=headers, timeout=10)
    except requests.RequestException as e:
        log.warning(f"⚠️ No se pudo marcar como leído: {e}")


def procesar_payload(data: dict):
    """Lógica completa de negocio: recorre el payload de Meta, arma el
    contexto, llama a OpenAI y responde al usuario. Corre en un hilo aparte."""
    try:
        entries = data.get('entry', [])
        for entry in entries:
            changes = entry.get('changes', [])
            for change in changes:
                value = change.get('value', {})

                if change.get('field') != 'messages' or value.get('messaging_product') != 'whatsapp':
                    continue

                contacts = value.get('contacts', [])
                nombre_usuario = "Usuario"
                if contacts:
                    nombre_usuario = contacts[0].get('profile', {}).get('name', 'Usuario')

                metadata = value.get('metadata', {})
                phone_number_id = metadata.get('phone_number_id') or PHONE_NUMBER_ID

                messages = value.get('messages', [])
                for message in messages:
                    procesar_mensaje_individual(message, nombre_usuario, phone_number_id)

    except Exception as e:
        log.exception(f"❌ Error interno procesando el flujo de Meta: {e}")


def procesar_mensaje_individual(message: dict, nombre_usuario: str, phone_number_id: str):
    numero_usuario = message.get('from')
    wamid = message.get('id')

    # --- Deduplicación: Meta reintenta webhooks si no respondemos rápido ---
    if ya_procesado(wamid):
        log.info(f"🔁 Mensaje {wamid} ya procesado antes, se ignora (reintento de Meta).")
        return
    marcar_procesado(wamid)

    marcar_como_leido(wamid, phone_number_id)

    texto_usuario = None
    if message.get('type') == 'text':
        texto_usuario = message.get('text', {}).get('body')

    referral = message.get('referral')
    ad_context = ""
    if referral:
        headline = referral.get('headline', '')
        body = referral.get('body', '')
        source_id = referral.get('source_id', '')
        ad_context = f"\n[El usuario hizo clic en el anuncio de Facebook: '{headline}' - '{body}' (ID: {source_id})]"
        if not texto_usuario:
            texto_usuario = f"Hola, me interesa el anuncio: {headline}"

    if not texto_usuario:
        return

    log.info(f"📩 {nombre_usuario} ({numero_usuario}) dijo: {texto_usuario}")

    contexto_privado = cargar_contexto()

    instrucciones_sistema = (
        "Eres un asistente de servicio al cliente automatizado y amable.\n"
        "Usa ÚNICAMENTE el siguiente contexto para responder la pregunta del usuario.\n"
        "REGLA CRÍTICA: Si la respuesta no se encuentra explícitamente en el contexto, "
        "sigue las reglas de escalamiento definidas en el contexto (mensaje de escalación + "
        "llamada a la función notificar_traspaso). No inventes ni asumas información.\n\n"
        f"Contexto:\n{contexto_privado}"
    )

    if ad_context:
        instrucciones_sistema += (
            f"\n\nContexto de origen del anuncio:\n{ad_context}\n"
            "IMPORTANTE: Saluda amigablemente haciendo alusión al anuncio de forma natural "
            "y prioriza la información del contexto privado relacionada con el tema del anuncio."
        )

    historial_previo = obtener_historial(numero_usuario)
    entrada_modelo = historial_previo + [{"role": "user", "content": texto_usuario}]

    respuesta_final = ""
    tipo_traspaso_detectado = None

    try:
        response = client.responses.create(
            model="gpt-4o-mini",
            instructions=instrucciones_sistema,
            input=entrada_modelo,
            temperature=0.5,
            max_output_tokens=2048,
            store=True,
            tools=tools_traspaso,
        )

        respuesta_final = response.output_text or ""

        for item in response.output:
            if getattr(item, "type", None) == "function_call" and item.name == "notificar_traspaso":
                try:
                    datos = json.loads(item.arguments)
                except json.JSONDecodeError:
                    datos = {}

                tipo_traspaso_detectado = datos.get("tipo")
                etiqueta = {
                    "listo_para_inscribir": "🔥 LISTO PARA INSCRIBIR",
                    "duda_sin_resolver": "❓ DUDA SIN RESOLVER",
                    "tramite_administrativo": "🗂 TRÁMITE ADMINISTRATIVO",
                }.get(tipo_traspaso_detectado, tipo_traspaso_detectado or "TRASPASO")

                aviso = (
                    f"{etiqueta}\n"
                    f"Programa: {datos.get('programa', 'N/A')}\n"
                    f"Cliente: {nombre_usuario} ({numero_usuario})\n"
                    f"Nota: {datos.get('resumen', 'Sin detalle')}"
                )
                enviar_whatsapp(NUMERO_ASESOR, aviso, phone_number_id)

    except Exception as e:
        log.exception(f"❌ Error llamando a OpenAI para {numero_usuario}: {e}")
        enviar_whatsapp(numero_usuario, MENSAJE_ERROR_TECNICO, phone_number_id)
        # Aun con error de IA, guardamos el mensaje del usuario para no perder contexto
        guardar_mensaje(numero_usuario, "user", texto_usuario)
        return

    if not respuesta_final:
        respuesta_final = MENSAJES_RESPALDO.get(tipo_traspaso_detectado, MENSAJE_RESPALDO_GENERICO)
    elif tipo_traspaso_detectado == "listo_para_inscribir":
        instruccion_inscripcion = (
            "Para continuar con tu inscripción, comunícate al número 6144150015 con el mensaje "
            "\"estoy listo para la inscripcion\" o haz clic en este enlace: "
            "https://wa.me/526144150015?text=estoy%20listo%20para%20la%20inscripcion"
        )
        if "6144150015" not in respuesta_final and "614 415 0015" not in respuesta_final:
            respuesta_final = respuesta_final.strip() + "\n\n" + instruccion_inscripcion

    enviar_whatsapp(numero_usuario, respuesta_final, phone_number_id)
    log.info(f"🤖 Chatbot respondió a {numero_usuario}: {respuesta_final}")

    guardar_mensaje(numero_usuario, "user", texto_usuario)
    guardar_mensaje(numero_usuario, "assistant", respuesta_final)
    limpiar_historial_antiguo(numero_usuario)


def enviar_whatsapp(number: str, text: str, phone_number_id: str = None, reintentos: int = 2):
    if not phone_number_id:
        phone_number_id = PHONE_NUMBER_ID
    url = f"https://graph.facebook.com/{API_VERSION}/{phone_number_id}/messages"
    headers = {
        "Authorization": f"Bearer {META_TOKEN}",
        "Content-Type": "application/json",
    }
    payload = {
        "messaging_product": "whatsapp",
        "to": number,
        "type": "text",
        "text": {"body": text},
    }

    for intento in range(1, reintentos + 2):
        try:
            res = requests.post(url, json=payload, headers=headers, timeout=15)
            if res.status_code >= 400:
                log.error(
                    f"❌ Error enviando WhatsApp a {number} (status {res.status_code}): {res.text}"
                )
            return res
        except requests.RequestException as e:
            log.warning(f"⚠️ Intento {intento} fallido enviando WhatsApp a {number}: {e}")
            if intento <= reintentos:
                time.sleep(2 * intento)
    log.error(f"❌ No se pudo enviar el mensaje a {number} tras {reintentos + 1} intentos.")
    return None


if __name__ == '__main__':
    from waitress import serve
    log.info("¡Servidor de producción Waitress encendido en el puerto 5000!")
    serve(app, host='0.0.0.0', port=5000, threads=8)
