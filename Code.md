import json
import os
import re
import math
from typing import List, Dict, Optional
from datetime import datetime

try:
    from googlesearch import search as google_search, SearchResult
except ImportError:
    google_search = None
    SearchResult = None

try:
    import requests
    from bs4 import BeautifulSoup
except ImportError:
    requests = None
    BeautifulSoup = None
    print("Advertencia: no están instaladas las dependencias de búsqueda web.")
    print("Instálalas con: python -m pip install requests beautifulsoup4")

SEARCH_URL = "https://www.google.com/search"


class MiniIA:
    """Mini IA simple que responde preguntas y usa Google si es posible."""

    PALABRAS_BUSQUEDA = (
        "qué", "quién", "quien", "cuándo", "cuando", "cómo", "como",
        "buscar", "información", "internet", "opinión",
        "noticia", "explica", "explicar", "definición", "significa",
        "última", "reciente", "dónde", "donde", "por qué", "porque"
    )

    def __init__(self, max_resultados: int = 3, historial_archivo: str = "historial.json"):
        self.max_resultados = max_resultados
        self.historial_archivo = historial_archivo
        self.historial: List[Dict[str, str]] = self.cargar_historial()

    def cargar_historial(self) -> List[Dict[str, str]]:
        if os.path.exists(self.historial_archivo):
            try:
                with open(self.historial_archivo, "r", encoding="utf-8") as f:
                    return json.load(f)
            except json.JSONDecodeError:
                print("Error al cargar el historial. Se creará uno nuevo.")
        return []

    def guardar_historial(self) -> None:
        try:
            with open(self.historial_archivo, "w", encoding="utf-8") as f:
                json.dump(self.historial, f, ensure_ascii=False, indent=4)
        except Exception as e:
            print(f"Error al guardar el historial: {e}")

    def necesita_internet(self, texto: str) -> bool:
        texto = texto.lower().strip()
        return any(palabra in texto for palabra in self.PALABRAS_BUSQUEDA)

    def buscar_web(self, pregunta: str) -> List[Dict[str, str]]:
        resultados: List[Dict[str, str]] = []

        if google_search is not None:
            try:
                for item in google_search(
                    pregunta,
                    num_results=self.max_resultados,
                    lang="es",
                    advanced=True,
                    sleep_interval=1,
                    safe="off",
                ):
                    if isinstance(item, SearchResult):
                        titulo = item.title or item.url
                        texto = item.description or item.url
                    else:
                        titulo = str(item)
                        texto = str(item)
                    resultados.append({"titulo": titulo, "texto": texto})
                if resultados:
                    return resultados
            except Exception as error:
                print(f"Error en la búsqueda de Google: {error}")

        if requests is None or BeautifulSoup is None:
            print("Advertencia: no están disponibles las dependencias de búsqueda web.")
            return resultados

        try:
            response = requests.post(
                SEARCH_URL,
                data={"q": pregunta},
                headers={"User-Agent": "Mozilla/5.0"},
                timeout=10,
            )
            response.raise_for_status()
            soup = BeautifulSoup(response.text, "html.parser")
            items = soup.find_all("div", class_="result")

            for item in items[: self.max_resultados]:
                title_node = item.find("a", class_="result__a")
                snippet_node = item.find("a", class_="result__snippet") or item.find("div", class_="result__snippet")
                titulo = title_node.get_text(strip=True) if title_node else "Sin título"
                texto = snippet_node.get_text(" ", strip=True) if snippet_node else "Sin resumen disponible."
                resultados.append({"titulo": titulo, "texto": texto})
        except Exception as error:
            print(f"Error en la búsqueda alternativa: {error}")
        return resultados

    def resolver_expresion_matematica(self, texto: str) -> Optional[str]:
        """Intenta resolver expresiones matemáticas simples."""
        try:
            texto_limpio = texto.lower().replace("x", "*").replace("por", "*").replace("dividido entre", "/").replace("div", "/").replace("^", "**")
            if "=" in texto_limpio:
                texto_limpio = texto_limpio.split("=")[0]
            
            caracteres_validos = set("0123456789+-*/.()% ")
            if not all(c in caracteres_validos for c in texto_limpio):
                return None
            
            if any(op in texto_limpio for op in ["+", "-", "*", "/", "%", "("]):
                resultado = eval(texto_limpio, {"__builtins__": {}}, {})
                if isinstance(resultado, (int, float)):
                    if isinstance(resultado, float) and resultado == int(resultado):
                        return f"El resultado es: {int(resultado)}"
                    else:
                        return f"El resultado es: {resultado:.4f}".rstrip("0").rstrip(".")
        except:
            pass
        return None

    def resolver_conversion(self, texto: str) -> Optional[str]:
        """Resuelve conversiones de temperatura y unidades simples."""
        texto_lower = texto.lower()
        
        celsius_a_fahrenheit = re.search(r"(\d+(?:\.\d+)?)\s*°?c\s+a\s+°?f", texto_lower)
        if celsius_a_fahrenheit:
            celsius = float(celsius_a_fahrenheit.group(1))
            fahrenheit = (celsius * 9/5) + 32
            return f"{celsius}°C = {fahrenheit:.1f}°F"
        
        fahrenheit_a_celsius = re.search(r"(\d+(?:\.\d+)?)\s*°?f\s+a\s+°?c", texto_lower)
        if fahrenheit_a_celsius:
            fahrenheit = float(fahrenheit_a_celsius.group(1))
            celsius = (fahrenheit - 32) * 5/9
            return f"{fahrenheit}°F = {celsius:.1f}°C"
        
        km_a_m = re.search(r"(\d+(?:\.\d+)?)\s*km\s+a\s+m(?:etros)?", texto_lower)
        if km_a_m:
            km = float(km_a_m.group(1))
            metros = km * 1000
            return f"{km} km = {metros:,.0f} metros"
        
        m_a_km = re.search(r"(\d+(?:\.\d+)?)\s*m(?:etros)?\s+a\s+km", texto_lower)
        if m_a_km:
            metros = float(m_a_km.group(1))
            km = metros / 1000
            return f"{metros} m = {km:.3f} km"
        
        return None

    def responder_matematica(self, texto: str) -> Optional[str]:
        """Responde preguntas sobre temas matemáticos."""
        texto_lower = texto.lower()
        
        raiz = re.search(r"ra[ií]z\s*(?:cuadrada\s+)?de\s+(\d+(?:\.\d+)?)", texto_lower)
        if raiz:
            numero = float(raiz.group(1))
            resultado = math.sqrt(numero)
            return f"La raíz cuadrada de {numero} es {resultado:.4f}".rstrip("0").rstrip(".")
        
        factorial = re.search(r"factorial\s+de\s+(\d+)|(\d+)!", texto_lower)
        if factorial:
            numero = int(factorial.group(1) or factorial.group(2))
            if numero > 20:
                return "El número es demasiado grande para calcular el factorial."
            resultado = math.factorial(numero)
            return f"El factorial de {numero} es {resultado}"
        
        porcentaje = re.search(r"(\d+(?:\.\d+)?)\s*%\s+de\s+(\d+(?:\.\d+)?)", texto_lower)
        if porcentaje:
            pct = float(porcentaje.group(1))
            numero = float(porcentaje.group(2))
            resultado = (pct / 100) * numero
            return f"El {pct}% de {numero} es {resultado:.2f}".rstrip("0").rstrip(".")
        
        return None

    def responder_general(self, texto: str) -> Optional[str]:
        """Responde preguntas generales sin necesidad de internet."""
        texto_lower = texto.lower()
        
        respuestas = {
            "hola": "¡Hola! ¿En qué puedo ayudarte?",
            "saludos": "¡Hola! ¿En qué puedo ayudarte?",
            "buenos días": "¡Buenos días! ¿Qué necesitas?",
            "buenas tardes": "¡Buenas tardes! ¿Cómo estás?",
            "buenas noches": "¡Buenas noches! ¿En qué puedo ayudarte?",
            "adiós": "¡Hasta luego! Fue un placer conversar.",
            "chau": "¡Hasta luego! Fue un placer conversar.",
            "tu nombre": "Soy Mini IA, un asistente inteligente y ágil.",
            "cómo te llamas": "Me llaman Mini IA. ¿Y tú?",
            "quién eres": "Soy Mini IA, una asistente de inteligencia artificial versátil.",
        }
        
        for clave, respuesta in respuestas.items():
            if clave in texto_lower:
                return respuesta
        
        if "hora" in texto_lower:
            return f"Son las {datetime.now().strftime('%H:%M:%S')}"
        
        if "fecha" in texto_lower or "día de hoy" in texto_lower or "qué día" in texto_lower:
            hoy = datetime.now()
            dias = ["lunes", "martes", "miércoles", "jueves", "viernes", "sábado", "domingo"]
            meses = ["enero", "febrero", "marzo", "abril", "mayo", "junio", "julio", "agosto", "septiembre", "octubre", "noviembre", "diciembre"]
            dia_semana = dias[hoy.weekday()]
            mes = meses[hoy.month - 1]
            return f"Hoy es {dia_semana}, {hoy.day} de {mes} de {hoy.year}"
        
        return None

    def responder_local(self, pregunta: str) -> str:
        texto = pregunta.strip()
        
        resultado = self.responder_general(texto)
        if resultado:
            return resultado
        
        resultado = self.resolver_expresion_matematica(texto)
        if resultado:
            return resultado
        
        resultado = self.resolver_conversion(texto)
        if resultado:
            return resultado
        
        resultado = self.responder_matematica(texto)
        if resultado:
            return resultado
        
        return "Esa pregunta es más compleja. Déjame buscar en internet... 🌐"

    def generar_resumen(self, resultados: List[Dict[str, str]], pregunta: str) -> str:
        if not resultados:
            return "Lo siento, no encontré información relevante en la web."

        resumen = f"Según la web sobre '{pregunta}':\n\n"
        textos = [r["texto"] for r in resultados if r["texto"]]
        texto_combinado = " ".join(textos[:2])
        resumen += texto_combinado[:500] + ("..." if len(texto_combinado) > 500 else "")
        return resumen

    def responder(self, pregunta: str) -> str:
        if self.necesita_internet(pregunta):
            resultados = self.buscar_web(pregunta)
            respuesta = self.generar_resumen(resultados, pregunta)
        else:
            respuesta = self.responder_local(pregunta)

        self.historial.append({"pregunta": pregunta, "respuesta": respuesta})
        self.guardar_historial()
        return respuesta

    def ejecutar(self) -> None:
        print("Mini IA iniciada. Escribe 'historial' para ver el historial o 'salir' para terminar.")
        while True:
            pregunta = input("Tú: ").strip()
            if not pregunta:
                continue
            if pregunta.lower() == "salir":
                print("Cerrando Mini IA. Hasta luego.")
                break
            if pregunta.lower() == "historial":
                self.mostrar_historial()
                continue
            respuesta = self.responder(pregunta)
            print(f"IA: {respuesta}\n")

    def mostrar_historial(self) -> None:
        if not self.historial:
            print("No hay historial aún.")
            return
        for i, item in enumerate(self.historial[-5:], 1):
            print(f"{i}. {item['pregunta']} -> {item['respuesta']}")


if __name__ == "__main__":
    ia = MiniIA()
    ia.ejecutar()
