## Hi there 👋

"""
Genera un GIF animado: una persona sentada en un escritorio pasando
las hojas de un libro.

Requisitos:  pip install pillow
Uso:         python persona_leyendo.py
Resultado:   persona_leyendo.gif (en la misma carpeta)
"""

import math
import random
from PIL import Image, ImageDraw

# ---------- Configuración (cámbiala a tu gusto) ----------
W, H = 640, 480          # tamaño final del GIF
S = 2                    # supersampling: dibuja al doble y reduce (bordes suaves)
FRAMES_LECTURA = 16      # cuadros mientras "lee"
FRAMES_VOLTEO = 14       # cuadros que dura el volteo de cada hoja
HOJAS = 4                # cuántas hojas pasa en total
MS_POR_CUADRO = 60       # velocidad (milisegundos por cuadro)
ARCHIVO = "persona_leyendo.gif"

# ---------- Colores ----------
PARED = (222, 233, 243)
VENTANA = (176, 214, 240)
MARCO = (255, 255, 255)
ESCRITORIO = (150, 98, 58)
ESCRITORIO_LINEA = (133, 85, 49)
PIEL = (240, 195, 160)
PELO = (60, 40, 30)
CAMISA = (70, 110, 180)
PAGINA = (252, 250, 243)
PAGINA_SOMBRA = (225, 221, 208)
TAPA = (120, 50, 50)
TEXTO = (170, 168, 160)

# ---------- Geometría del libro (vista en perspectiva) ----------
CX = W // 2
LIBRO_ARRIBA, LIBRO_ABAJO = 335, 425
MEDIO_ARRIBA, MEDIO_ABAJO = 140, 170   # semiancho arriba / abajo del libro


def ease(t):
    """Suaviza el movimiento (arranca y frena despacio)."""
    return 0.5 - 0.5 * math.cos(math.pi * t)


def lerp(a, b, t):
    return a + (b - a) * t


def P(pts):
    """Escala puntos por el factor de supersampling."""
    return [(x * S, y * S) for x, y in pts]


def poligono(d, pts, fill, outline=None, width=1):
    d.polygon(P(pts), fill=fill, outline=outline, width=width * S if outline else 0)


def elipse(d, cx, cy, rx, ry, fill, outline=None, width=1):
    d.ellipse([(cx - rx) * S, (cy - ry) * S, (cx + rx) * S, (cy + ry) * S],
              fill=fill, outline=outline, width=width * S)


def linea(d, pts, fill, width=1):
    d.line(P(pts), fill=fill, width=int(width * S), joint="curve")


def punto_en_cuadrilatero(q, u, v):
    """Interpolación bilineal dentro de un cuadrilátero.
    q = (lomo_arriba, lomo_abajo, borde_abajo, borde_arriba)
    u: 0 (lomo) -> 1 (borde exterior), v: 0 (arriba) -> 1 (abajo)"""
    la, lb, ba, bt = q
    x_top = lerp(la[0], bt[0], u)
    y_top = lerp(la[1], bt[1], u)
    x_bot = lerp(lb[0], ba[0], u)
    y_bot = lerp(lb[1], ba[1], u)
    return lerp(x_top, x_bot, v), lerp(y_top, y_bot, v)


def lineas_de_texto(d, q, semilla):
    """Dibuja renglones de 'texto' dentro de una página."""
    rnd = random.Random(semilla)
    for i in range(9):
        v = 0.14 + i * 0.085
        largo = rnd.uniform(0.62, 0.88)
        p1 = punto_en_cuadrilatero(q, 0.14, v)
        p2 = punto_en_cuadrilatero(q, 0.14 + largo * 0.72, v)
        linea(d, [p1, p2], TEXTO, 1.6)


def cuadrilatero_pagina(lado, factor, levanta):
    """Página apoyada (lado = +1 derecha, -1 izquierda) o en movimiento.
    factor: cos del ángulo (1 = plana derecha, -1 = plana izquierda)."""
    ax = MEDIO_ARRIBA * factor
    bx = MEDIO_ABAJO * factor
    return ((CX, LIBRO_ARRIBA), (CX, LIBRO_ABAJO),
            (CX + bx, LIBRO_ABAJO - levanta), (CX + ax, LIBRO_ARRIBA - levanta))


def posicion_mano(fase, t):
    """Devuelve (x, y) de la mano según la fase de la animación."""
    if fase == "lectura":
        r = ease(min(t / 0.6, 1.0))              # vuelve de izquierda a derecha
        return CX + lerp(-MEDIO_ABAJO, MEDIO_ABAJO, r), 415 - 60 * math.sin(math.pi * r)
    a = math.pi * ease(t)                        # volteo
    return CX + MEDIO_ABAJO * math.cos(a), 415 - 40 * math.sin(a)


# ---------- Dibujo de escena ----------
def dibujar_fondo(d, n):
    d.rectangle([0, 0, W * S, H * S], fill=PARED)
    # ventana
    d.rectangle([40 * S, 40 * S, 180 * S, 170 * S], fill=VENTANA, outline=MARCO, width=6 * S)
    linea(d, [(110, 40), (110, 170)], MARCO, 5)
    linea(d, [(40, 105), (180, 105)], MARCO, 5)
    # cuadro en la pared
    d.rectangle([470 * S, 60 * S, 580 * S, 140 * S], fill=(236, 205, 150), outline=(110, 80, 50), width=5 * S)
    d.polygon(P([(480, 130), (520, 90), (545, 115), (560, 100), (572, 130)]), fill=(120, 160, 110))


def dibujar_escritorio(d, n):
    d.rectangle([0, 330 * S, W * S, H * S], fill=ESCRITORIO)
    for y in (360, 395, 435):
        linea(d, [(0, y), (W, y)], ESCRITORIO_LINEA, 2)
    linea(d, [(0, 330), (W, 330)], (120, 75, 42), 4)

    # lámpara
    elipse(d, 85, 395, 38, 9, (60, 60, 70))
    linea(d, [(85, 392), (110, 330), (85, 285)], (80, 80, 90), 6)
    poligono(d, [(60, 285), (125, 270), (135, 300), (75, 312)], (240, 190, 70))

    # taza con vapor
    d.rectangle([555 * S, 372 * S, 595 * S, 412 * S], fill=(245, 245, 245), outline=(200, 200, 200), width=2 * S)
    elipse(d, 603, 392, 9, 12, None, (245, 245, 245), 4)
    for k, x0 in enumerate((565, 575, 585)):
        pts = [(x0 + 4 * math.sin(n * 0.35 + k + j * 0.9), 366 - j * 7) for j in range(5)]
        linea(d, pts, (235, 235, 240), 2)


def dibujar_persona(d, n, mirando):
    bob = 2 * math.sin(n * 0.18)
    # torso
    d.rounded_rectangle([(CX - 120) * S, 225 * S, (CX + 120) * S, 360 * S],
                        radius=40 * S, fill=CAMISA)
    # cuello
    d.rectangle([(CX - 14) * S, (190 + bob) * S, (CX + 14) * S, 232 * S], fill=PIEL)
    # cabeza
    cy = 150 + bob
    elipse(d, CX, cy - 6, 60, 60, PELO)
    elipse(d, CX, cy, 52, 56, PIEL)
    d.pieslice([(CX - 57) * S, (cy - 57) * S, (CX + 57) * S, (cy + 57) * S],
               start=195, end=345, fill=PELO)
    # ojos
    parpadeo = (n % 44) in (0, 1)
    dx = mirando[0]
    dy = mirando[1]
    for ex in (-20, 20):
        if parpadeo:
            linea(d, [(CX + ex - 9, cy + 8), (CX + ex + 9, cy + 8)], (60, 40, 30), 2)
        else:
            elipse(d, CX + ex, cy + 8, 10, 8, (255, 255, 255), (150, 120, 100), 1)
            elipse(d, CX + ex + dx, cy + 8 + dy, 4.2, 4.2, (40, 30, 30))
    # cejas y boca
    linea(d, [(CX - 30, cy - 8), (CX - 10, cy - 11)], PELO, 3)
    linea(d, [(CX + 10, cy - 11), (CX + 30, cy - 8)], PELO, 3)
    linea(d, [(CX - 9, cy + 34), (CX, cy + 37), (CX + 9, cy + 34)], (170, 90, 80), 2.5)


def dibujar_libro(d, fase, t):
    # tapa y grosor
    poligono(d, [(CX - 182, LIBRO_ARRIBA - 4), (CX + 182, LIBRO_ARRIBA - 4),
                 (CX + 190, LIBRO_ABAJO + 12), (CX - 190, LIBRO_ABAJO + 12)], TAPA)
    der = cuadrilatero_pagina(+1, 1, 0)
    izq = cuadrilatero_pagina(-1, -1, 0)
    for q in (der, izq):
        poligono(d, list(q), PAGINA, PAGINA_SOMBRA, 1)
    # canto de las hojas (efecto de grosor)
    for k in range(3):
        linea(d, [(CX - 168 + k, LIBRO_ABAJO + 2 + k), (CX + 168 - k, LIBRO_ABAJO + 2 + k)], PAGINA_SOMBRA, 1.5)

    lineas_de_texto(d, der, d_id["pag"] + 1)
    lineas_de_texto(d, izq, d_id["pag"])
    linea(d, [(CX, LIBRO_ARRIBA), (CX, LIBRO_ABAJO)], (190, 185, 170), 2)

    # hoja en movimiento
    if fase == "volteo":
        a = math.pi * ease(t)
        f = math.cos(a)
        q = cuadrilatero_pagina(+1, f, 42 * math.sin(a))
        sombra = int(255 - 40 * math.sin(a))
        poligono(d, list(q), (sombra, sombra - 2, sombra - 10), (200, 196, 182), 1)
        lineas_de_texto(d, q, d_id["pag"] + 2 if f > 0 else d_id["pag"] + 3)


def dibujar_brazos(d, mano):
    hx, hy = mano
    # brazo izquierdo (quieto, apoyado)
    linea(d, [(CX - 105, 255), (CX - 190, 330), (CX - 225, 392)], CAMISA, 34)
    elipse(d, CX - 228, 396, 19, 15, PIEL)
    # brazo derecho (pasa las hojas)
    linea(d, [(CX + 105, 255), (CX + 175, 320), (hx + 12, hy - 22)], CAMISA, 34)
    elipse(d, hx, hy, 19, 15, PIEL)
    # dedos
    for k in (-8, 0, 8):
        elipse(d, hx - 4 + k * 0.4, hy + 9, 5, 7, PIEL)


# estado compartido simple (número de página actual)
d_id = {"pag": 0}


def crear_cuadro(n, fase, t, pagina):
    d_id["pag"] = pagina * 2
    img = Image.new("RGB", (W * S, H * S), PARED)
    d = ImageDraw.Draw(img)

    dibujar_fondo(d, n)
    # hacia dónde mira: sigue la mano al voltear, recorre renglones al leer
    if fase == "volteo":
        mirando = (3.5 * math.cos(math.pi * ease(t)), 3)
    else:
        mirando = (3.5 * math.sin(n * 0.55), 3.2)
    dibujar_persona(d, n, mirando)
    dibujar_escritorio(d, n)
    dibujar_libro(d, fase, t)
    dibujar_brazos(d, posicion_mano(fase, t))

    return img.resize((W, H), Image.LANCZOS)


def main():
    cuadros = []
    n = 0
    for pagina in range(HOJAS):
        for i in range(FRAMES_LECTURA):
            cuadros.append(crear_cuadro(n, "lectura", i / (FRAMES_LECTURA - 1), pagina))
            n += 1
        for i in range(FRAMES_VOLTEO):
            cuadros.append(crear_cuadro(n, "volteo", i / (FRAMES_VOLTEO - 1), pagina))
            n += 1

    cuadros[0].save(
        ARCHIVO,
        save_all=True,
        append_images=cuadros[1:],
        duration=MS_POR_CUADRO,
        loop=0,           # 0 = se repite para siempre
        optimize=True,
    )
    print(f"Listo: {ARCHIVO} ({len(cuadros)} cuadros)")


if __name__ == "__main__":
    main()
