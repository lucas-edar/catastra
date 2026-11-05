# Catastra

Parcelas oficiales de la Ciudad de Buenos Aires por API y MCP: buscás por punto, SMP o área y recibís superficie, partida y zonificación del Código Urbanístico.
Pensado para devs y agentes de IA. **CABA hoy.**

🌐 https://catastra.ldest.dev · MCP: `https://catastra.ldest.dev/mcp` (Streamable HTTP) · Registry: `dev.ldest.catastra/catastra`

## Qué le podés preguntar

| Herramienta MCP | Qué hace | Endpoint REST |
|---|---|---|
| `parcela_por_coordenada` (lat, lon, incluir_geometria?) | Parcela en un punto de CABA | `GET /v1/parcelas/punto?lat=&lon=` |
| `parcela_por_nomenclatura` (smp o nomenclatura, jurisdiccion=CABA) | Busca por SMP, con o sin espacios | `GET /v1/parcelas/buscar?smp=` |
| `ficha_parcela` (id `AR-C-<smp>`) | Ficha completa de una parcela | `GET /v1/parcelas/{id}` |
| `parcelas_en_area` (bbox, límite 1–200, cursor) | Parcelas en hasta 1 km² (lados ≤ 2 km) | `GET /v1/parcelas/area?bbox=min_lon,min_lat,max_lon,max_lat` |
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

**Cursor** (`.cursor/mcp.json`):
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

**Claude Code** (todavía no probado con Catastra):
```sh
claude mcp add --transport http catastra https://catastra.ldest.dev/mcp \
  --header "Authorization: Bearer <clave>"
```

**Claude Desktop** (todavía no probado con Catastra). Claude Desktop conecta servidores remotos desde *Settings → Connectors*, pero ese flujo no permite header fijo; la opción con clave es vía `mcp-remote` en `claude_desktop_config.json`:
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

**Cliente MCP genérico:** transporte Streamable HTTP, URL `https://catastra.ldest.dev/mcp`, header `Authorization: Bearer <clave>`. `initialize` y `tools/list` responden sin clave; `tools/call` la exige.

## Prompts de ejemplo

- «¿Qué parcela hay en -34.551262, -58.490720? Dame superficie, partida y zonificación.»
- «Traeme la ficha de la parcela SMP 055-200-005.»
- «Listame las parcelas de esta manzana de Saavedra y cuál es la más grande.»
- «¿Qué fuentes usás y hasta qué fecha están actualizadas?»
- «Comparame la zonificación de estas dos direcciones en CABA.» (pasando coordenadas; hoy no busca por dirección)

## Ejemplo real

Respuestas reales del servidor (`structuredContent` de MCP, noviembre de 2026). Sin geometría, porque `incluir_geometria` es false por defecto.

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
- **CABA hoy.** Otras jurisdicciones todavía no están disponibles.
- No busca por dirección postal (todavía): usá coordenadas o SMP.

## Fuentes y fechas de corte

Fuente: Gobierno de la Ciudad de Buenos Aires — BA Data, licencia CC-BY-2.5-AR.
- Parcelas CABA — corte **2026-07-07** (se publica semestral) — https://data.buenosaires.gob.ar/dataset/parcelas
- Código Urbanístico — corte **2024-12-31** — https://data.buenosaires.gob.ar/dataset/codigo-urbanistico

## Límites (plan gratis)

- 100 consultas por mes calendario (hora de Buenos Aires) y 10 por minuto, por clave, compartidas entre REST y MCP.
- `parcelas_en_area`: hasta 1 km², lados de hasta 2 km, hasta 200 resultados por página.
- `cobertura_y_fuentes` no descuenta cupo.

---

## English

Catastra serves official Buenos Aires City (CABA) parcel data via REST API and a remote MCP server (`https://catastra.ldest.dev/mcp`, Streamable HTTP). Look up parcels by point, SMP or small area and get area, tax ID (partida) and zoning. Get a free key (100 queries/month, 10/min) at https://catastra.ldest.dev/#clave and send it as `Authorization: Bearer <key>`. Informational only — not an official zoning certificate. Data cutoffs: parcels 2026-07-07, zoning code 2024-12-31. Coverage: CABA today.
