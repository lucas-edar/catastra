# Catastra

Parcelas oficiales de CABA + AMBA (25 partidos de la Provincia de Buenos Aires) por API y MCP: buscás por punto, ficha o área, por SMP en CABA o por CCA (nomenclatura ARBA) o partida + partido en PBA y recibís superficie y partida. Zonificación del Código Urbanístico solo en CABA; PBA no tiene zonificación.
Pensado para devs y agentes de IA. **El resto de PBA próximamente; más provincias si hay interés.**

🌐 https://catastra.ldest.dev · MCP: `https://catastra.ldest.dev/mcp` (Streamable HTTP) · Registry: `dev.ldest.catastra/catastra`

## Qué le podés preguntar

| Herramienta MCP | Qué hace | Endpoint REST |
|---|---|---|
| `parcela_por_coordenada` (lat, lon, incluir_geometria=false) | Parcelas en un punto de CABA o AMBA | `GET /v1/parcelas/punto?lat=&lon=` |
| `parcela_por_nomenclatura` (smp o nomenclatura, partida, partido, jurisdiccion="CABA" o "PBA", incluir_geometria=false) | CABA (por defecto): SMP. PBA: CCA de 42 caracteres o partida + partido (código ARBA de tres dígitos, conservando ceros) | `GET /v1/parcelas/buscar?smp=`; PBA: `?jurisdiccion=PBA&nomenclatura=<CCA>` o `?jurisdiccion=PBA&partida=<partida>&partido=<codigo>` |
| `ficha_parcela` (id, incluir_geometria=false) | Ficha completa de CABA o AMBA; usá el id devuelto por una búsqueda | `GET /v1/parcelas/{id}` |
| `parcelas_en_area` (min_lon, min_lat, max_lon, max_lat, limite=50 [1–200], cursor opcional) | Parcelas de CABA y AMBA en hasta 1 km² (lados ≤ 2 km); cursor por último id, mismo rectángulo al paginar | `GET /v1/parcelas/area?bbox=min_lon,min_lat,max_lon,max_lat` |
| `cobertura_y_fuentes` | Cobertura y atribuciones (no descuenta cupo) | `GET /v1/cobertura` (público, sin clave) |

## Arranque rápido

1. Entrá a https://catastra.ldest.dev/#clave, pedí el código por mail y confirmalo.
2. Guardá la clave: se muestra **una sola vez**.
3. Mandala en el header `Authorization: Bearer <clave>` (REST y MCP).

```sh
curl 'https://catastra.ldest.dev/v1/parcelas/AR-C-055-200-005' \
  -H 'Authorization: Bearer <clave>'
```

## Configuración MCP

**Cursor** (`.cursor/mcp.json`; formato tomado de `llms-full.txt` del proyecto):
```json
{
  "mcpServers": {
    "catastra": {
      "type": "http",
      "url": "https://catastra.ldest.dev/mcp",
      "headers": { "Authorization": "Bearer <clave>" }
    }
  }
}
```

**Claude Code** ⚠️ sintaxis estándar de Claude Code, no probada contra Catastra:
```sh
claude mcp add --transport http catastra https://catastra.ldest.dev/mcp \
  --header "Authorization: Bearer <clave>"
```

**Claude Desktop** ⚠️ no probado. Claude Desktop conecta servidores remotos desde *Settings → Connectors*, pero ese flujo no permite header fijo; la opción con clave es vía `mcp-remote` en `claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "catastra": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://catastra.ldest.dev/mcp",
               "--header", "Authorization:${CATASTRA_AUTH}"],
      "env": { "CATASTRA_AUTH": "Bearer <clave>" }
    }
  }
}
```

**Cliente MCP genérico:** transporte Streamable HTTP, URL `https://catastra.ldest.dev/mcp`, header `Authorization: Bearer <clave>`. `initialize` y `tools/list` responden sin clave; `tools/call` la exige (verificado 05/11 15:48).

## Prompts de ejemplo

- «¿Qué parcela hay en -34.551262, -58.490720? Dame superficie, partida y zonificación.»
- «Traeme la ficha de la parcela SMP 055-200-005.»
- «Listame las parcelas de esta manzana de Saavedra y cuál es la más grande.»
- «¿Qué fuentes usás y hasta qué fecha están actualizadas?»
- «Comparame la zonificación de estas dos direcciones en CABA.» (pasando coordenadas; hoy no busca por dirección)

## Ejemplo real

**Verificado en vivo (05/11/2026 15:48, hora de Buenos Aires):**
- `initialize` → OK: `serverInfo.name = "catastra"`, `protocolVersion 2025-06-18`, capability `tools`.
- `tools/list` → OK: las 5 herramientas de la tabla, todas `readOnlyHint: true`.
- `tools/call ficha_parcela {"id":"AR-C-055-200-005"}` sin clave → `{"error":{"codigo":"no_autorizado","mensaje":"Se requiere una clave válida"}}`.
- `GET /v1/cobertura` → OK: CABA, 318.046 parcelas (317.894 con Código Urbanístico).

**Salida real** de producción (`structuredContent` de MCP), capturada por Parcela el 05/11/2026 entre las 15:50 y las 15:51 (hora de Buenos Aires; horario de modificación de los archivos en `ejemplos-reales-2026-11-05/`). Las respuestas no traen geometría porque `incluir_geometria` queda en false por defecto.

```
Usuario: Traeme la ficha de la parcela 055-200-005 en CABA.
Agente → ficha_parcela {"id": "AR-C-055-200-005"}
```
```json
{
  "id": "AR-C-055-200-005",
  "jurisdiccion": "CABA",
  "barrio": "Saavedra",
  "comuna": "Comuna 12",
  "nomenclatura": {
    "original": "055 - 200 - 005",
    "normalizada": "055-200-005",
    "componentes": {
      "seccion": "055",
      "manzana": "200",
      "parcela": "005"
    }
  },
  "partida": "369869",
  "superficie": {
    "geometrica_m2": 374.690796448272
  },
  "centroide": {
    "lat": -34.55126173363963,
    "lon": -58.49071996652731
  },
  "fuentes": [
    {
      "nombre": "Código Urbanístico — BA Data",
      "url": "https://data.buenosaires.gob.ar/dataset/codigo-urbanistico",
      "fecha_corte": "2024-12-31",
      "atribucion": "Fuente: Gobierno de la Ciudad de Buenos Aires — BA Data",
      "licencia": "CC-BY-2.5-AR"
    },
    {
      "nombre": "Parcelas CABA — BA Data",
      "url": "https://data.buenosaires.gob.ar/dataset/parcelas",
      "fecha_corte": "2026-07-07",
      "atribucion": "Fuente: Gobierno de la Ciudad de Buenos Aires — BA Data",
      "licencia": "CC-BY-2.5-AR"
    }
  ],
  "aviso_uso": "Información orientativa: no es una certificación urbanística oficial; consultá la normativa vigente en el municipio",
  "zonificacion": {
    "metodo": "join_smp",
    "distrito": "U34",
    "distrito_2": null,
    "fot": {
      "entre_medianeras": "0",
      "perimetro_libre": "0",
      "semi_libre": "0"
    },
    "plano_limite": "0",
    "unidad_edificabilidad": "0",
    "afectaciones": {
      "catalogado": "0",
      "apertura": "0",
      "ensanche": "0",
      "rivolta": "0"
    },
    "fos": null,
    "nota": "FOS no aplica en el Código Urbanístico de CABA"
  }
}
```
Resumen: Saavedra, Comuna 12, partida 369869, 374,7 m² de superficie geométrica, distrito U34.

```
Usuario: ¿Qué parcela hay en -34.551262, -58.490720?
Agente → parcela_por_coordenada {"lat": -34.551262, "lon": -58.490720}
```
```json
{
  "parcelas": [
    {
      "id": "AR-C-055-200-005",
      "jurisdiccion": "CABA",
      "barrio": "Saavedra",
      "comuna": "Comuna 12",
      "nomenclatura": {
        "original": "055 - 200 - 005",
        "normalizada": "055-200-005",
        "componentes": {
          "seccion": "055",
          "manzana": "200",
          "parcela": "005"
        }
      },
      "partida": "369869",
      "superficie": {
        "geometrica_m2": 374.690796448272
      },
      "centroide": {
        "lat": -34.55126173363963,
        "lon": -58.49071996652731
      },
      "zonificacion": {
        "distrito": "U34",
        "…": "igual que la ficha"
      },
      "fuentes": "… igual que la ficha"
    }
  ]
}
```
(Recortado: `fuentes`, `aviso_uso` y el resto de `zonificacion` son idénticos a la ficha de arriba.)

> Sobre los "0": en afectaciones (catalogado, apertura, ensanche, rivolta) "0" significa "no". FOT, plano límite y unidad de edificabilidad en 0 son el valor que publica BA Data para esa parcela; Catastra no los modifica.

## Qué no somos

- **No es un certificado oficial.** Es información orientativa; no reemplaza una certificación urbanística.
- **No reemplaza el Informe Urbanístico** ni la consulta de la normativa vigente en el municipio.
- **Los datos tienen fecha de corte** (ver abajo) y no se actualizan en tiempo real.
- Ningún dato de titulares ni de valuación.
- No busca por dirección postal (todavía): usá coordenadas, SMP en CABA, o CCA o partida + partido en PBA.

## Cobertura

**CABA + AMBA (25 partidos de la Provincia de Buenos Aires).** El resto de PBA próximamente; más provincias si hay interés.

Partidos cubiertos (código ARBA): 003 Almirante Brown; 004 Avellaneda; 025 Lanús; 030 Esteban Echeverría; 032 Florencio Varela; 047 General San Martín; 055 La Plata; 057 Tigre; 063 Lomas de Zamora; 070 La Matanza; 072 Merlo; 074 Moreno; 086 Quilmes; 096 San Fernando; 097 San Isidro; 101 Morón; 110 Vicente López; 117 Tres de Febrero; 120 Berazategui; 130 Ezeiza; 131 San Miguel; 132 José C. Paz; 133 Malvinas Argentinas; 135 Hurlingham; 136 Ituzaingó.

PBA no tiene zonificación. Solo CABA incluye Código Urbanístico cuando está en la fuente. Las cantidades de parcelas por partido se consultan en `/v1/cobertura`.

## Fuentes y fechas de corte

Fuente: Gobierno de la Ciudad de Buenos Aires — BA Data, licencia CC-BY-2.5-AR.
- Parcelas CABA — corte **2026-07-07** (se publica semestral) — https://data.buenosaires.gob.ar/dataset/parcelas
- Código Urbanístico — corte **2024-12-31** — https://data.buenosaires.gob.ar/dataset/codigo-urbanistico

Fuente: ARBA (Agencia de Recaudación de la Provincia de Buenos Aires) — parcelas PBA, licencia CC BY 2.5 AR, corte **noviembre de 2026**.

## Límites (plan gratis)

- 100 consultas por mes calendario (hora de Buenos Aires) y 10 por minuto, por clave, compartidas entre REST y MCP.
- `parcelas_en_area`: hasta 1 km², lados de hasta 2 km, hasta 200 resultados por página.
- `cobertura_y_fuentes` no descuenta cupo.

---

## English

Coverage: CABA and AMBA (25 Buenos Aires Province districts). The rest of Buenos Aires Province is coming soon; more provinces if there is interest. Catastra serves official parcel data via REST API and a remote MCP server (`https://catastra.ldest.dev/mcp`, Streamable HTTP). Look up parcels by point, record ID or small area, by SMP in CABA, or by CCA (ARBA cadastral identifier) or partida + district code in PBA. Zoning is available only for CABA; PBA has no zoning. No ownership or valuation data. Get a free key (100 queries/month, 10/min) at https://catastra.ldest.dev/#clave and send it as `Authorization: Bearer <key>`. Informational only — not an official zoning certificate. Sources and cutoffs: BA Data, CABA parcels 2026-07-07 and zoning code 2024-12-31; ARBA, PBA parcels November 2026, CC BY 2.5 AR (attribution: Fuente: ARBA).
