# TrafficIA

## Descripción

TrafficIA es un generador de tráfico urbano en CARLA que permite estudiar cómo la elección de destinos influye en los desplazamientos dentro de un entorno simulado. Combina un modelo estocástico de selección de destinos, planificación de rutas y registro de viajes e incidentes para apoyar el análisis de movilidad.

El proyecto utiliza Town01, un catálogo de 42 destinos y escenarios de 20 y 100 vehículos. Compara dos selectores: elección uniforme y elección basada en historiales sintéticos, con una probabilidad de 80 % para rutina y 20 % para exploración.

**Título del trabajo de graduación:** Desarrollo de un generador de tráfico urbano en CARLA mediante un modelo estocástico de elección de destinos y planificación de rutas para análisis de movilidad en escenarios simulados.

## Tecnologías Utilizadas

- **CARLA 0.9.15 y su Python API:** simulación y comunicación con el entorno virtual.
- **Python 3.7.9:** implementación del generador y herramientas de análisis.
- **NumPy 1.21.6:** cálculo numérico.
- **pandas 1.3.5:** procesamiento de registros.
- **Matplotlib 3.5.3:** visualización de resultados.
- **NetworkX 2.6.3:** trabajo con grafos.
- **Shapely 2.0.7:** operaciones geométricas.

## Requisitos Previos

- Windows de 64 bits.
- CARLA 0.9.15 descomprimido por separado, con `CarlaUE4.exe`, el mapa Town01 y la carpeta `PythonAPI/carla`.
- Python 3.7.9 de 64 bits y las dependencias del proyecto.
- PowerShell para los comandos de demostración disponibles.
- Git para obtener y gestionar el repositorio.

## Instalación

TrafficIA se encuentra en [src/](src/), junto con sus configuraciones, datos de entrada, scripts, pruebas y documentación técnica. Dentro de esa carpeta, `src/` contiene el paquete Python del proyecto.

1. Abre PowerShell en la raíz de esta entrega, entra en la carpeta del software y comprueba el intérprete. El resultado debe ser `3.7.9 64`.

   ```powershell
   Set-Location -LiteralPath '.\src'
   python -c "import struct, sys; print(sys.version.split()[0], struct.calcsize('P') * 8)"
   ```

2. Crea un entorno virtual nuevo e instala las [dependencias](src/requirements.txt).

   ```powershell
   python -m venv carla-env
   $py = Join-Path (Get-Location).Path 'carla-env\Scripts\python.exe'
   & $py -m pip install --upgrade pip==24.0
   & $py -m pip install -r .\requirements.txt
   ```

3. Configura y carga `CARLA_PYTHONAPI` como se explica en [Configuración del entorno](#configuración-del-entorno). En la misma sesión, instala el cliente de CARLA compatible con Python 3.7 de 64 bits.

   ```powershell
   & $py -m pip install "$env:CARLA_PYTHONAPI\dist\carla-0.9.15-cp37-cp37m-win_amd64.whl"
   & $py -m pip check
   ```

4. Comprueba la importación del SDK y ejecuta las pruebas. Estas comprobaciones no necesitan el servidor de CARLA abierto.

   ```powershell
   & $py -c "import carla; print(carla.__file__)"
   & $py -B -m unittest discover -s tests -v
   ```

El simulador, el entorno virtual y las salidas generadas no se distribuyen con el código. La [guía técnica](src/docs/guias/manual.md#preparar-el-entorno) amplía las instrucciones y la resolución de incidencias.

## Configuración del entorno

`CARLA_PYTHONAPI` indica la ruta absoluta a `PythonAPI/carla` de CARLA 0.9.15, que debe contener la subcarpeta `agents`. Permite importar los agentes de navegación; instalar el paquete `carla` por sí solo no garantiza que estos módulos estén disponibles. También puede indicarse mediante `--carla-pythonapi`; cuando los agentes aún no están importables, esa ruta se comprueba antes que la variable de entorno.

El archivo [src/.env.example](src/.env.example) contiene un valor de ejemplo. Desde la carpeta `src/` de esta entrega, crea la copia local si todavía no existe:

```powershell
if (-not (Test-Path -LiteralPath '.\.env')) {
    Copy-Item -LiteralPath '.\.env.example' -Destination '.\.env'
}
```

Edita `.env` y sustituye la ruta de ejemplo por la de tu instalación, sin comillas alrededor del valor. Este archivo permanece excluido de Git. TrafficIA lee el entorno del proceso y no carga `.env` automáticamente; en cada sesión de PowerShell, carga la variable y comprueba la ruta:

```powershell
$lineaCarla = Get-Content -LiteralPath '.\.env' | Where-Object { $_ -match '^CARLA_PYTHONAPI=' } | Select-Object -First 1
if (-not $lineaCarla) { throw 'Define CARLA_PYTHONAPI en .env.' }
$env:CARLA_PYTHONAPI = ($lineaCarla -split '=', 2)[1].Trim()
if (-not (Test-Path -LiteralPath (Join-Path $env:CARLA_PYTHONAPI 'agents') -PathType Container)) {
    throw 'CARLA_PYTHONAPI debe apuntar a la carpeta que contiene agents.'
}
$carlaDir = Split-Path -Parent (Split-Path -Parent $env:CARLA_PYTHONAPI)
$py = Join-Path (Get-Location).Path 'carla-env\Scripts\python.exe'
```

`$carlaDir` corresponde a la carpeta que contiene `CarlaUE4.exe` y `$py` al intérprete del entorno virtual. Al abrir otra consola, entra de nuevo en `src/` y repite el bloque de carga; no es necesario reinstalar las dependencias. El host, el puerto, el mapa y la duración se configuran mediante JSON y argumentos de ejecución.

## Ejecución

Desde la misma sesión de PowerShell, dentro de `src/` y con las variables del apartado anterior definidas, inicia el simulador:

```powershell
Start-Process -FilePath (Join-Path $carlaDir 'CarlaUE4.exe') -WorkingDirectory $carlaDir `
  -ArgumentList '-quality-level=Low', '-dx11', '-windowed', '-ResX=1280', '-ResY=720' `
  -WindowStyle Normal
```

Espera a que aparezca el mundo de CARLA. Carga Town01 y ejecuta una demostración de 20 vehículos con el selector uniforme:

```powershell
& $py .\scripts\maps\load_map.py --town Town01
if ($LASTEXITCODE -ne 0) { throw 'No se pudo cargar Town01.' }
& $py .\scripts\simulation\observe_trips.py `
  --config .\config\experiments\demand_20_100\town01_020.json `
  --scenario random_baseline --seed 210 --seconds 30 --simulation-speed 1 `
  --navigation-diagnostics --no-cameras --carla-pythonapi $env:CARLA_PYTHONAPI
```

La consola indica la carpeta de resultados dentro de `src/data/output/`, contada desde la raíz de la entrega. Consulta `run_config.json` para comprobar los parámetros y `trips_log.csv` para ver las elecciones y terminaciones de los viajes.

Para probar el selector basado en historiales, sustituye `--scenario random_baseline` por `--scenario history_80_20`. Para utilizar 100 vehículos, cambia `town01_020.json` por `town01_100.json` en la misma ruta de configuración. Estas opciones permiten reproducir los cuatro casos de la demostración. Cada caso utiliza **30 segundos simulados**; el tiempo real de procesamiento puede ser mayor. Para ejecutar campañas, analizar registros o generar gráficas, consulta el [manual de TrafficIA](src/docs/guias/manual.md).

## Capturas

Las capturas del sistema se encuentran en [capturas/](capturas/). Se incluyen **15 imágenes**, numeradas consecutivamente.

Las primeras 12 imágenes muestran el inicio de CARLA, la carga de Town01, circulación, viajes con ruta y los cuatro casos de 20 y 100 vehículos con ambos selectores. Las tres figuras siguientes presentan resultados experimentales de TrafficIA.

| Figura | Contenido y relación con los objetivos |
|---|---|
| [Captura 13](capturas/captura-13.png) | Distribución de destinos asignados y alcanzados por categoría para comparar ambos selectores (objetivos 1 y 3). |
| [Captura 14](capturas/captura-14.png) | Evolución del historial y proporciones de decisiones de rutina y exploración del modelo (objetivo 2). |
| [Captura 15](capturas/captura-15.png) | Tiempo medio de viajes completados, velocidad media de presencia e indicadores de colisión e invasión de carril para comparar los selectores (objetivo 3). |

## Demo

El [video demostrativo de TrafficIA](demo/demo.mp4) presenta el proyecto, muestra las cuatro corridas de 20 y 100 vehículos con ambos selectores, explica sus registros y cierra con el aporte del sistema.

## Documentación

- [Informe final](docs/informe_final.pdf).
- [Consentimiento de uso de fotografías y video firmado](docs/consentimiento_firmado.pdf).
- [Documentación técnica de TrafficIA](src/docs/README.md).

La organización sigue el [manual oficial de entrega](https://csuvg.github.io/Guia-entrega-trabajo-graduacion/docs/manual.html).

## Autores

- **Sergio Alejandro Orellana Colindres — Carné 221122.**

Universidad del Valle de Guatemala. Ingeniería en Ciencia de la Computación y Tecnologías de la Información. Trabajo de graduación, 2026.

**Asesor:** PhD. Gabriel Antonio Barrientos Rodriguez.

## Licencia

El código de TrafficIA se distribuye bajo **Mozilla Public License 2.0**; el texto se incluye en [src/LICENSE](src/LICENSE). Esta referencia corresponde al código y no modifica las condiciones de uso del informe, las capturas ni el consentimiento institucional.
