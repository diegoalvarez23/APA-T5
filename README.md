# Sonido estéreo y ficheros WAVE

## Nom i cognoms

> [!Important]
> Introduzca a continuación su nombre y apellidos:
>
> Diego Alvarez Tome
>

##### Estereo.py
```
"""
Nombre y apellidos: Diego Alvarez Tome

Funciones para manejar ficheros WAVE PCM usando únicamente struct.
Incluye conversión estéreo-mono, mono-estéreo, codificación estéreo
en muestras de 32 bits y decodificación de nuevo a estéreo.
"""


import struct


def leer_cabecera(fic):
    """Lee un fichero WAVE PCM y devuelve su información básica y datos."""

    with open(fic, "rb") as f:
        riff = f.read(4)
        if riff != b"RIFF":
            raise ValueError("El fichero no es RIFF")

        tam_riff = struct.unpack("<I", f.read(4))[0]

        wave = f.read(4)
        if wave != b"WAVE":
            raise ValueError("El fichero no es WAVE")

        fmt_encontrado = False
        data_encontrado = False

        formato = None
        canales = None
        frecuencia = None
        byte_rate = None
        block_align = None
        bits = None
        data = None
        tam_data = None

        while True:
            chunk_id = f.read(4)

            if len(chunk_id) == 0:
                break

            if len(chunk_id) != 4:
                raise ValueError("Cacho WAVE incompleto")

            tam_chunk = struct.unpack("<I", f.read(4))[0]
            contenido = f.read(tam_chunk)

            if chunk_id == b"fmt ":
                fmt_encontrado = True

                if tam_chunk < 16:
                    raise ValueError("Subcacho fmt incorrecto")

                (
                    formato,
                    canales,
                    frecuencia,
                    byte_rate,
                    block_align,
                    bits,
                ) = struct.unpack("<HHIIHH", contenido[:16])

                if formato != 1:
                    raise ValueError("El fichero no usa PCM lineal")

            elif chunk_id == b"data":
                data_encontrado = True
                tam_data = tam_chunk
                data = contenido

            if tam_chunk % 2 == 1:
                f.read(1)

            if fmt_encontrado and data_encontrado:
                break

    if not fmt_encontrado:
        raise ValueError("No se ha encontrado el subcacho fmt")

    if not data_encontrado:
        raise ValueError("No se ha encontrado el subcacho data")

    return {
        "tam_riff": tam_riff,
        "formato": formato,
        "canales": canales,
        "frecuencia": frecuencia,
        "byte_rate": byte_rate,
        "block_align": block_align,
        "bits": bits,
        "tam_data": tam_data,
        "data": data,
    }


def escribir_wav(fic, canales, frecuencia, bits, data):
    """Escribe un fichero WAVE PCM con una cabecera estándar."""

    bytes_por_muestra = bits // 8
    block_align = canales * bytes_por_muestra
    byte_rate = frecuencia * block_align
    tam_data = len(data)
    tam_riff = 36 + tam_data

    with open(fic, "wb") as f:
        f.write(b"RIFF")
        f.write(struct.pack("<I", tam_riff))
        f.write(b"WAVE")

        f.write(b"fmt ")
        f.write(struct.pack("<I", 16))
        f.write(struct.pack("<H", 1))
        f.write(struct.pack("<H", canales))
        f.write(struct.pack("<I", frecuencia))
        f.write(struct.pack("<I", byte_rate))
        f.write(struct.pack("<H", block_align))
        f.write(struct.pack("<H", bits))

        f.write(b"data")
        f.write(struct.pack("<I", tam_data))
        f.write(data)


def limitar_16_bits(valor):
    """Limita un valor entero al rango permitido por muestras PCM de 16 bits."""

    return max(-32768, min(32767, valor))


def estereo2mono(ficEste, ficMono, canal=2):
    """
    Convierte un fichero WAVE estéreo de 16 bits en un fichero mono.

    canal=0: canal izquierdo.
    canal=1: canal derecho.
    canal=2: semisuma (L + R) / 2.
    canal=3: semidiferencia (L - R) / 2.
    """

    cab = leer_cabecera(ficEste)

    if cab["canales"] != 2:
        raise ValueError("El fichero de entrada no es estéreo")

    if cab["bits"] != 16:
        raise ValueError("El fichero de entrada no tiene muestras de 16 bits")

    if canal not in (0, 1, 2, 3):
        raise ValueError("El argumento canal debe ser 0, 1, 2 o 3")

    num_muestras = cab["tam_data"] // 2
    muestras = struct.unpack("<" + "h" * num_muestras, cab["data"])

    pares = list(zip(muestras[0::2], muestras[1::2]))

    if canal == 0:
        mono = [l for l, r in pares]
    elif canal == 1:
        mono = [r for l, r in pares]
    elif canal == 2:
        mono = [(l + r) // 2 for l, r in pares]
    else:
        mono = [(l - r) // 2 for l, r in pares]

    data = struct.pack("<" + "h" * len(mono), *mono)

    escribir_wav(
        ficMono,
        canales=1,
        frecuencia=cab["frecuencia"],
        bits=16,
        data=data,
    )


def mono2estereo(ficIzq, ficDer, ficEste):
    """
    Construye un fichero WAVE estéreo de 16 bits a partir de dos ficheros mono.
    """

    izq = leer_cabecera(ficIzq)
    der = leer_cabecera(ficDer)

    if izq["canales"] != 1 or der["canales"] != 1:
        raise ValueError("Los dos ficheros de entrada deben ser monofónicos")

    if izq["bits"] != 16 or der["bits"] != 16:
        raise ValueError("Los dos ficheros deben tener muestras de 16 bits")

    if izq["frecuencia"] != der["frecuencia"]:
        raise ValueError("Los dos ficheros deben tener la misma frecuencia")

    if izq["tam_data"] != der["tam_data"]:
        raise ValueError("Los dos ficheros deben tener el mismo tamaño de datos")

    num_muestras = izq["tam_data"] // 2

    muestras_izq = struct.unpack("<" + "h" * num_muestras, izq["data"])
    muestras_der = struct.unpack("<" + "h" * num_muestras, der["data"])

    estereo = [
        muestra
        for par in zip(muestras_izq, muestras_der)
        for muestra in par
    ]

    data = struct.pack("<" + "h" * len(estereo), *estereo)

    escribir_wav(
        ficEste,
        canales=2,
        frecuencia=izq["frecuencia"],
        bits=16,
        data=data,
    )


def codEstereo(ficEste, ficCod):
    """
    Codifica un WAVE estéreo de 16 bits como WAVE mono de 32 bits.

    Los 16 bits más significativos contienen la semisuma.
    Los 16 bits menos significativos contienen la semidiferencia.
    """

    cab = leer_cabecera(ficEste)

    if cab["canales"] != 2:
        raise ValueError("El fichero de entrada no es estéreo")

    if cab["bits"] != 16:
        raise ValueError("El fichero de entrada no tiene muestras de 16 bits")

    num_muestras = cab["tam_data"] // 2
    muestras = struct.unpack("<" + "h" * num_muestras, cab["data"])

    codificadas = []

    for l, r in zip(muestras[0::2], muestras[1::2]):
        semisuma = (l + r) // 2
        semidiferencia = (l - r) // 2

        parte_alta = semisuma << 16
        parte_baja = semidiferencia & 0xFFFF

        codificadas.append(parte_alta | parte_baja)

    data = struct.pack("<" + "i" * len(codificadas), *codificadas)

    escribir_wav(
        ficCod,
        canales=1,
        frecuencia=cab["frecuencia"],
        bits=32,
        data=data,
    )


def decEstereo(ficCod, ficEste):
    """
    Decodifica un WAVE mono de 32 bits y reconstruye un WAVE estéreo de 16 bits.
    """

    cab = leer_cabecera(ficCod)

    if cab["canales"] != 1:
        raise ValueError("El fichero de entrada debe ser monofónico")

    if cab["bits"] != 32:
        raise ValueError("El fichero de entrada debe tener muestras de 32 bits")

    num_muestras = cab["tam_data"] // 4
    muestras = struct.unpack("<" + "i" * num_muestras, cab["data"])

    estereo = []

    for muestra in muestras:
        semisuma = muestra >> 16
        semidiferencia = muestra & 0xFFFF

        if semidiferencia >= 0x8000:
            semidiferencia -= 0x10000

        l = limitar_16_bits(semisuma + semidiferencia)
        r = limitar_16_bits(semisuma - semidiferencia)

        estereo.extend([l, r])

    data = struct.pack("<" + "h" * len(estereo), *estereo)

    escribir_wav(
        ficEste,
        canales=2,
        frecuencia=cab["frecuencia"],
        bits=16,
        data=data,
    )

```

##### Código de `estereo2mono()`

```python
def estereo2mono(ficEste, ficMono, canal=2):
    """
    Convierte un fichero WAVE estéreo de 16 bits en un fichero mono.

    canal=0: canal izquierdo.
    canal=1: canal derecho.
    canal=2: semisuma (L + R) / 2.
    canal=3: semidiferencia (L - R) / 2.
    """

    cab = leer_cabecera(ficEste)

    if cab["canales"] != 2:
        raise ValueError("El fichero de entrada no es estéreo")

    if cab["bits"] != 16:
        raise ValueError("El fichero de entrada no tiene muestras de 16 bits")

    if canal not in (0, 1, 2, 3):
        raise ValueError("El argumento canal debe ser 0, 1, 2 o 3")

    num_muestras = cab["tam_data"] // 2
    muestras = struct.unpack("<" + "h" * num_muestras, cab["data"])

    pares = list(zip(muestras[0::2], muestras[1::2]))

    if canal == 0:
        mono = [l for l, r in pares]
    elif canal == 1:
        mono = [r for l, r in pares]
    elif canal == 2:
        mono = [(l + r) // 2 for l, r in pares]
    else:
        mono = [(l - r) // 2 for l, r in pares]

    data = struct.pack("<" + "h" * len(mono), *mono)

    escribir_wav(
        ficMono,
        canales=1,
        frecuencia=cab["frecuencia"],
        bits=16,
        data=data,
    )
```

##### Código de `mono2estereo()`

```python
def mono2estereo(ficIzq, ficDer, ficEste):
    """
    Construye un fichero WAVE estéreo de 16 bits a partir de dos ficheros mono.
    """

    izq = leer_cabecera(ficIzq)
    der = leer_cabecera(ficDer)

    if izq["canales"] != 1 or der["canales"] != 1:
        raise ValueError("Los dos ficheros de entrada deben ser monofónicos")

    if izq["bits"] != 16 or der["bits"] != 16:
        raise ValueError("Los dos ficheros deben tener muestras de 16 bits")

    if izq["frecuencia"] != der["frecuencia"]:
        raise ValueError("Los dos ficheros deben tener la misma frecuencia")

    if izq["tam_data"] != der["tam_data"]:
        raise ValueError("Los dos ficheros deben tener el mismo tamaño de datos")

    num_muestras = izq["tam_data"] // 2

    muestras_izq = struct.unpack("<" + "h" * num_muestras, izq["data"])
    muestras_der = struct.unpack("<" + "h" * num_muestras, der["data"])

    estereo = [
        muestra
        for par in zip(muestras_izq, muestras_der)
        for muestra in par
    ]

    data = struct.pack("<" + "h" * len(estereo), *estereo)

    escribir_wav(
        ficEste,
        canales=2,
        frecuencia=izq["frecuencia"],
        bits=16,
        data=data,
    )
```

##### Código de `codEstereo()`

```python
def codEstereo(ficEste, ficCod):
    """
    Codifica un WAVE estéreo de 16 bits como WAVE mono de 32 bits.

    Los 16 bits más significativos contienen la semisuma.
    Los 16 bits menos significativos contienen la semidiferencia.
    """

    cab = leer_cabecera(ficEste)

    if cab["canales"] != 2:
        raise ValueError("El fichero de entrada no es estéreo")

    if cab["bits"] != 16:
        raise ValueError("El fichero de entrada no tiene muestras de 16 bits")

    num_muestras = cab["tam_data"] // 2
    muestras = struct.unpack("<" + "h" * num_muestras, cab["data"])

    codificadas = []

    for l, r in zip(muestras[0::2], muestras[1::2]):
        semisuma = (l + r) // 2
        semidiferencia = (l - r) // 2

        parte_alta = semisuma << 16
        parte_baja = semidiferencia & 0xFFFF

        codificadas.append(parte_alta | parte_baja)

    data = struct.pack("<" + "i" * len(codificadas), *codificadas)

    escribir_wav(
        ficCod,
        canales=1,
        frecuencia=cab["frecuencia"],
        bits=32,
        data=data,
    )
```

##### Código de `decEstereo()`

```python
def decEstereo(ficCod, ficEste):
    """
    Decodifica un WAVE mono de 32 bits y reconstruye un WAVE estéreo de 16 bits.
    """

    cab = leer_cabecera(ficCod)

    if cab["canales"] != 1:
        raise ValueError("El fichero de entrada debe ser monofónico")

    if cab["bits"] != 32:
        raise ValueError("El fichero de entrada debe tener muestras de 32 bits")

    num_muestras = cab["tam_data"] // 4
    muestras = struct.unpack("<" + "i" * num_muestras, cab["data"])

    estereo = []

    for muestra in muestras:
        semisuma = muestra >> 16
        semidiferencia = muestra & 0xFFFF

        if semidiferencia >= 0x8000:
            semidiferencia -= 0x10000

        l = limitar_16_bits(semisuma + semidiferencia)
        r = limitar_16_bits(semisuma - semidiferencia)

        estereo.extend([l, r])

    data = struct.pack("<" + "h" * len(estereo), *estereo)

    escribir_wav(
        ficEste,
        canales=2,
        frecuencia=cab["frecuencia"],
        bits=16,
        data=data,
    )
```
