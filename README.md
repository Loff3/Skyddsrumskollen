# Skyddsrumskollen
En gratis, open-source och optimerad alternativ version av MSB's skyddsrumskarta

  # Skyddsrumskollen (SDI)

  End-to-end geodataplattform för att analysera och visualisera skyddsrumskapacitet i relation till befolkning per tätort i Sverige.

  Projektet visar hur en **Spatial Data Infrastructure (SDI)** kan byggas med öppna standarder och publiceras som interoperabla geotjänster (WMS/WFS) för
  användning i QGIS, Python och webbläsare.

  ## Innehåll
  - [Mål](#mål)
  - [Arkitektur](#arkitektur)
  - [Data och källor](#data-och-källor)
  - [Datapipeline](#datapipeline)
  - [Metadata och standarder](#metadata-och-standarder)
  - [Tjänster (WMS/WFS)](#tjänster-wmswfs)
  - [Prestandatester](#prestandatester)
  - [Resultat](#resultat)
  - [Teknikstack](#teknikstack)
  - [Reproducerbarhet](#reproducerbarhet)
  - [Begränsningar](#begränsningar)

  ---

  ## Mål
  - Bygga en komplett SDI från myndighetsdata till publicerade karttjänster.
  - Integrera skyddsrum, tätorter och administrativa gränser i ett gemensamt geodataflöde.
  - Identifiera tätorter med möjlig kapacitetsbrist.
  - Utvärdera systemets beteende under normal och hög belastning.

  ## Arkitektur

  **Flöde:**
  `INSPIRE Geoportal -> QGIS (rensning + analys) -> PostgreSQL/PostGIS -> GeoServer (WMS/WFS) -> QGIS/Python/Webb -> JMeter`

  ## Data och källor
  - **MSB**: Skyddsrum (punktdata, kapacitet)
    - Fält: `geom`, `numberOfOc`
  - **SCB**: Tätorter + befolkning (polygondata)
    - Fält: `geom`, `tatort`, `bef`
  - **Lantmäteriet**: Länsgränser (polygondata)
    - Fält: `geom`, `VARNAME_1`

  ### Datakvalitet
  - Ursprunglig shapefile gav teckenkodningsproblem (US-ASCII).
  - Löst genom nedladdning som **GeoPackage (UTF-8)**.
  - Onödiga/tomma attributfält rensades.
  - Gemensamt koordinatsystem: **EPSG:3006 (SWEREF 99 TM)**.

  ## Datapipeline
  1. Import och kvalitetsgranskning i QGIS.
  2. Städning av attribut och struktur.
  3. Point-in-polygon för att räkna skyddsrum per tätort.
  4. Import till PostgreSQL/PostGIS via DB Manager.
  5. Publicering i GeoServer (workspace, store, layers).
  6. Styling och klassning för tydlig visualisering.
  7. Grundläggande CSP i GeoServer för begränsad domänåtkomst.

  ## Metadata och standarder
  Projektet följer:
  - **INSPIRE** (öppen geodata, interoperabilitet)
  - **ISO 19139** (metadata)
  - **OGC WMS/WFS** (tjänstestandarder)

  Använda metadatafält (urval):
  - Title
  - Abstract
  - Category
  - Responsible Organisation
  - SRS
  - License

  ## Tjänster (WMS/WFS)

  ### Exempel: WFS 1.1.0 med filter
  ```http
  http://localhost:8086/geoserver/wfs?service=wfs&version=1.1.0&request=GetFeature&typeName=project:skyddsrum_tatort_count_simplified&Filter=<Filter><PropertyIsLessThan><PropertyName>NUMPOINTS</PropertyName><Literal>5</Literal></PropertyIsLessThan></Filter>&outputFormat=GML2

  ### Exempel: WFS 2.0.0 med FES 2.0

  http://localhost:8086/geoserver/wfs?service=wfs&version=2.0.0&request=GetFeature&typeNames=project:skyddsrum_tatort_count_simplified&count=50&Filter=<fes:Filter
  xmlns:fes="http://www.opengis.net/fes/2.0"><fes:PropertyIsGreaterThan><fes:ValueReference>bef</fes:ValueReference><fes:Literal>10000</fes:Literal></fes:PropertyIsGreaterThan></fes:Filter>&outputFormat=text/xml;
  subtype=gml/3.1.1

  ## Prestandatester

  Verktyg: Apache JMeter

  ### Load test

  - 200 threads, ramp-up 60
  - Gradvis ökning av svarstid och latens när samtidigheten stiger.
  - Indikerar ökande belastning på WFS-anrop.

  ### Stress test

  - 300 threads, ramp-up 50
      - 0% error rate, men svarstider upp mot ~21s vid slutet.
  - 500 threads, ramp-up 40
      - Brytpunkt nås: fel börjar uppstå runt tråd ~215.
      - Vid högsta belastning upp till ~50s latens i sena anrop.

  ### Tolkning

  - Systemet fungerar väl under normal användning.
  - Tunga samtidiga WFS-läsningar kräver optimering för produktionsmiljö med hög trafik.

  ## Resultat

  - Fungerande SDI-lösning med publicerade WMS/WFS-tjänster.
  - Visualisering av skyddsrumskapacitet per tätort:
      - Punktlager för skyddsrum (kapacitet)
      - Polygonlager för tätorter (riskklassning)
      - Länsgränser för geografisk kontext
  - Interoperabilitet verifierad i:
      - QGIS
      - Python-klient
      - Webbläsare (HTTP-anrop)

  ## Teknikstack

  - QGIS
  - PostgreSQL/PostGIS
  - GeoServer
  - Python (PyQGIS/WFS-anrop)
  - Apache JMeter

  ## Reproducerbarhet

  > Exakta scripts/testplaner kan läggas i docs/ och tests/jmeter/.

  1. Ladda ner MSB/SCB/Lantmäteriet-data via INSPIRE/Geodataportalen.
  2. Rensa och förbered i QGIS.
  3. Kör point-in-polygon och skapa slutlager.
  4. Importera till PostGIS.
  5. Publicera lager i GeoServer.
  6. Verifiera via WFS-anrop i QGIS/Python.
  7. Kör JMeter-load/stress-scenarier.

  ## Begränsningar

  - Ingen cache-/tile-optimering i denna version.
