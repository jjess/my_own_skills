---
name: python_custom_log
description: Convención de logs para scripts Python del usuario (estilo del proyecto ndwt). Usar al crear un script Python nuevo, al añadir o modificar logging en uno existente, o cuando se pidan logs "como en ndwt" - fichero diario en log/, cabecera START y cierre END con tiempos, y entradas jerárquicas "   +--->" / "   +------>".
---

# Logs en scripts Python (estilo ndwt)

Todo script Python que sea un punto de entrada (se ejecuta, no se importa) registra su
actividad con `logging`, a la vez en consola y en un fichero diario, con el formato de abajo.

## 1. Configuración (una vez, en el punto de entrada)

```python
import logging
import os
import sys
from datetime import datetime
from pathlib import Path

#######################
# Logging
#######################

BASE_DIR = Path(__file__).resolve().parent          # ajustar si el script vive en src/
LOG_DIR = BASE_DIR / "log"                           # <proyecto>/log, NO relativo al cwd
THIS_SCRIPT_NAME = os.path.basename(sys.argv[0])

logger = logging.getLogger(__name__)


def setup_logging(log_tag: str) -> Path:
    """Abre el log diario <LOG_DIR>/<log_tag>-AAAA-MM-DD.log (y consola)."""
    LOG_DIR.mkdir(parents=True, exist_ok=True)
    log_date = datetime.now().strftime("%Y-%m-%d")
    log_filename = LOG_DIR / f"{log_tag}-{log_date}.log"
    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s - %(levelname)s - %(message)s",
        handlers=[
            logging.FileHandler(log_filename, encoding="utf-8"),
            logging.StreamHandler(),
        ],
    )
    return log_filename
```

- `log_tag` sale de la configuración del proyecto con un valor por defecto, por ejemplo
  `cfg.get("log_tag", "<nombre_script>")`. El fichero se reutiliza durante todo el día (append).
- La ruta del log se calcula desde la ubicación del script, nunca con `../log/` relativo
  al directorio actual: el script puede lanzarse desde cualquier sitio (cron, wrappers .sh).
- `logging.basicConfig` solo se llama en el punto de entrada. Los módulos importables usan
  `logging.getLogger(__name__)` y no configuran handlers.
- La ayuda (`-h`), la llamada sin argumentos y los errores de argparse no abren el log.

## 2. Primera y última entrada

La primera entrada del proceso es START y la última es END con inicio, fin y duración. END se
emite en un `finally` para que aparezca también si hay error o Ctrl+C:

```python
start_time = datetime.now()
logger.info("*" * 48)
logger.info(f"START: {THIS_SCRIPT_NAME}")
logger.info("*" * 48)
try:
    rc = run()                        # trabajo real del script
finally:
    end_time = datetime.now()
    elapsed = int((end_time - start_time).total_seconds())
    elapsed_minutes, elapsed_seconds = divmod(elapsed, 60)
    logger.info("*" * 48)
    logger.info(f"END: {THIS_SCRIPT_NAME}")
    logger.info(f"   +---> start:    {start_time.strftime('%Y-%m-%d %H:%M:%S')}")
    logger.info(f"   +---> end:      {end_time.strftime('%Y-%m-%d %H:%M:%S')}")
    logger.info(f"   +---> elapsed:  {elapsed_minutes}m {elapsed_seconds}s")
    logger.info("*" * 48)
```

Conviene registrar también el código de salida (`   +---> exit code: N`) antes de END.

## 3. Entradas del log

| Tipo                         | Formato                                                  |
|------------------------------|----------------------------------------------------------|
| Entrada normal (nivel 1)     | `logger.info(f"   +---> texto {variable}")`              |
| Subtarea (nivel 2)           | `logger.info(f"   +------> texto {variable}")`           |
| Nivel 3 y siguientes         | añadir `---` por nivel: `   +--------->`                 |
| Error                        | `logger.error(f"   +---> X ERROR texto")` (con su nivel) |
| Aviso                        | `logger.warning(f"   +---> AVISO texto")`                |
| Separador / título de bloque | `"-" * 23`, título y `"-" * 23`                          |

- Una subtarea (`+------>`) solo aparece dentro de un bloque abierto por una entrada del
  nivel anterior (`+--->`). La sangría refleja la jerarquía del proceso.
- La salida de subprocesos se registra línea a línea como subtarea del paso que los lanzó,
  por ejemplo `logger.info(f"   +------> | {linea}")`.
- Conviene que un helper genere el prefijo, para que el nivel no se escriba a mano:

```python
def log(msg: str, level: int = 1, lvl: int = logging.INFO) -> None:
    """Entrada de log jerárquica: nivel 1 '   +--->', nivel 2 '   +------>', ..."""
    logger.log(lvl, f"   +{'---' * level}> {msg}")
```

## 4. Reglas

- Solo ASCII en los mensajes (acentos y ñ permitidos). Nada de emojis ni flechas Unicode:
  usar `->`, `[OK]`, `X ERROR`. Si se registra texto externo (salida de otros programas),
  sanearlo antes.
- Nunca registrar secretos: API keys, contraseñas, tokens ni cabeceras de autorización. Basta
  con indicar que se han cargado ("API key: cargada de .env (no se muestra)").
- No usar `print()` para la información del proceso: todo pasa por `logger`, que ya escribe
  en consola y en el fichero.
- El directorio `log/` existe en el repositorio pero vacío. En `.gitignore`:
  ```
  log/*
  !log/.gitkeep
  ```
  y se crea `log/.gitkeep`.
