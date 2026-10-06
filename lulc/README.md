# IntegraCAR LULC — Land Use and Land Cover

Research line of the IntegraCAR project that builds open datasets and baseline
models for **land use and land cover (LULC) semantic segmentation** of rural
properties registered in the CAR (*Cadastro Ambiental Rural*) in Espírito
Santo, Brazil.

It is independent from the CAR process workflow (gestão, extração, dashboard):
nothing here touches process documents or personal data from the E-Docs.

| Item | Where |
| --- | --- |
| Extraction pipeline | [`integracar-lulc-builder`](https://github.com/integraCAR/integracar-lulc-builder) (public) |
| Dataset (10,000 pairs) | [`laicsiifes/IntegraCAR-LULC-10K`](https://huggingface.co/datasets/laicsiifes/IntegraCAR-LULC-10K) on Hugging Face |
| Reference subset (500 pairs) | [`laicsiifes/IntegraCAR-LULC-500`](https://huggingface.co/datasets/laicsiifes/IntegraCAR-LULC-500) on Hugging Face |
| Dataset landing page | [`IntegraCAR-LULC-10K`](https://github.com/integraCAR/IntegraCAR-LULC-10K) (public) |
| Training code and weights | [`Thrakrien/IntegraCAR-LULC-10K-code`](https://github.com/Thrakrien/IntegraCAR-LULC-10K-code) (outside the organization) |
| Paper | SIBGRAPI 2026 ([record](http://sibgrapi.sid.inpe.br/col/sid.inpe.br/sibgrapi/2026/09.04.16.13/doc/thisInformationItemHomePage.html)) |

---

## Contents

1. [integracar-lulc-builder](#integracar-lulc-builder)
2. [IntegraCAR-LULC-10K dataset](#integracar-lulc-10k-dataset)
3. [IntegraCAR-LULC-10K landing page](#integracar-lulc-10k-landing-page)
4. [Citation](#citation)
5. [Open points](#open-points)

---

## integracar-lulc-builder

Asynchronous Python pipeline that, for each rural property centroid, downloads
georeferenced high-resolution imagery and the matching LULC thematic map from
the **GeoBases** spatial data infrastructure of Espírito Santo, and writes
paired GeoTIFF files with identical extent, CRS, pixel grid and GSD.

### Epochs

| Epoch | SATELLITE | SEGMENTED |
| --- | --- | --- |
| 2019–2020 | KOMPSAT-3/3A orthophotomosaic, WMS layer `geonode:ijsn-ortofotomosaico-es-kompsat-3-3a-2019-2020` | IJSN LULC map (2019), WMS layer `geonode:ijsn_map_uso_solo_es_2019_20200` |
| 2012–2015 | IEMA aerial orthophotomosaic (0.25 m GSD), WMS layer `geonode:iema_ortofotomosaico_es_025m_2012-2015` | GeoBases vector shapefile `Mapeamento_Uso_Cobertura_Vegetal_2012`, rasterized locally with the 2019–2020 RGB palette |
| `both` | All four products per coordinate, with 100% spatial parity | |

WMS endpoint: `https://ide.geobases.es.gov.br/geoserver/ows` (version 1.3.0).
The 2012–2015 shapefile is downloaded from the public GeoBases S3 bucket when
not cached locally.

### How it works

```
CSV (property_id; x; y in EPSG:31984)
  -> pyproj: UTM 24S -> EPSG:4326 bounding box (buffer around the centroid)
  -> aiohttp: WMS GetMap (PNG)                  -> Pillow decode -> NumPy
  -> geopandas/shapely: clip + rasterize (2012) -> palette harmonization
  -> rasterio: GeoTIFF, LZW, EPSG:4326
  -> artifacts/dataset_manifest.csv + logs/execution.log
```

### Repository layout

```
extractor.py           CLI and async orchestration
config.py              WMS endpoints, layers, CRS, defaults
utils/
  wms.py               Async WMS GetMap requests
  rasterize.py         2012-2015 shapefile download, clip and rasterization
  palette.py           RGB palette harmonization between epochs
  manifest.py          Audit manifest writer
coordenadas_10k.csv    Coordinates used for the 10K dataset
sample_train_coordinates.csv  Small sample for quick runs
Technologies.md        Role of each dependency
```

### Install and run

Python 3.10 or newer (`numpy>=2.2` requires it, although the builder README still says 3.8+).

```bash
git clone https://github.com/integraCAR/integracar-lulc-builder.git
cd integracar-lulc-builder
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# 2019-2020 (default)
python extractor.py --csv sample_train_coordinates.csv --output ./output

# both epochs
python extractor.py --csv sample_train_coordinates.csv --output ./output --period both

# quick smoke test
python extractor.py --csv sample_train_coordinates.csv --output ./output --limit 3
```

### CLI parameters

| Parameter | Default | Description |
| --- | --- | --- |
| `--csv` | required | Input CSV, `;` delimiter, columns `property_id` (or `cod_imovel`), `x`, `y` in EPSG:31984 |
| `--output` (`-o`, `--path`, `--caminho`) | required | Output directory |
| `--period` (`--year`, `--ano`, `--periodo`) | `2019-2020` | `2019-2020`, `2012-2015` or `both` |
| `--shapefile` (`--shp`) | auto | Local 2012–2015 shapefile or folder; otherwise cache or S3 download |
| `--flat` | off | Single period: write directly into `SATELLITE/` and `SEGMENTED/` |
| `--buffer` | `1024` | Half side, in meters, of the square window around the centroid |
| `--width` (`--largura`), `--height` (`--altura`) | `2048` | Output size in pixels |
| `--limit` (`--count`, `--qtd`) | all | Process only the first N rows |
| `--workers` | `4` | Concurrent async tasks |

GSD = `2 × buffer / width`. The defaults give **1.0 m/pixel** over a 2,048 m
window.

### Output

```
<output>/
  2012_2015/SATELLITE/sample_N.tif
  2012_2015/SEGMENTED/sample_N.tif
  2019_2020/SATELLITE/sample_N.tif
  2019_2020/SEGMENTED/sample_N.tif
artifacts/dataset_manifest.csv
logs/execution.log
```

Manifest columns: `sample_id`, `property_id`, `period`, `x`, `y`,
`bbox_xmin`, `bbox_ymin`, `bbox_xmax`, `bbox_ymax`, `satellite_status`,
`land_cover_status` (`ok`, `error`, `empty`), `download_timestamp`.

---

## IntegraCAR-LULC-10K dataset

High-resolution optical satellite dataset for LULC semantic segmentation in the
context of the CAR in Espírito Santo, built with `integracar-lulc-builder` from
KOMPSAT-3/3A imagery and GeoBases thematic annotations (2019–2020).

| Property | Value |
| --- | --- |
| Hugging Face | `laicsiifes/IntegraCAR-LULC-10K`, DOI `10.57967/hf/10542` |
| Creator | LAICSI — Laboratório de Inteligência Computacional e Sistemas de Informação (IFES) |
| License | MIT |
| Size | 10,000 image/mask pairs (20,000 rows), Parquet |
| Image size | 2,048 × 2,048 px, RGB |
| Splits | `satellite_train` / `mask_train` 6,000 · `satellite_val` / `mask_val` 2,000 · `satellite_test` / `mask_test` 2,000 |
| Fields | `image`, `filename`, `latitude`, `longitude`, `municipio`, `microestad` |
| Sampling | Geographic stratification: 1,000 coordinates in each of the 10 official microregions of Espírito Santo |
| Annotations | IJSN experts, visual photointerpretation of 2019–2020 imagery |

```python
from datasets import load_dataset
ds = load_dataset("laicsiifes/IntegraCAR-LULC-10K")
```

### Classes

The original GeoBases nomenclature (about two dozen classes, such as native
forest, regenerating forest, pasture, coffee, banana, eucalyptus, mangrove,
restinga, water, built-up area, mining) is grouped into five classes, defined
with IDAF for the CAR context:

| Class | Grouped original classes |
| --- | --- |
| Vegetation Areas | Native forest, restinga, mangroves, marshes, altitude grasslands |
| Agropastoral Areas | Pastures, crops, silviculture, exposed soil, initial regeneration for productive use |
| Infrastructure | Buildings, paved and dirt roads, mining, other built surfaces |
| Water Bodies | Rivers, lakes, reservoirs, canals |
| Macega | Kept as its own class: pioneer or transitional vegetation of tall grasses and shrubs, usually fallow land or abandoned pasture |

### IntegraCAR-LULC-500

Reference subset with 500 samples (50 per microregion), split 300 / 100 / 100.
Meant for quick prototyping with low compute.

### Baselines (on LULC-500)

Best configurations reported on the landing page (Table II of the paper):

| Model | Patch stride | OA | Macro F1 | mIoU |
| --- | --- | --- | --- | --- |
| DeepLabv3 (ResNet-50) | 64 px | 82.5% | 66.8% | 53.7% |
| U-Net (EfficientNet-B5) | 64 px | 82.5% | 62.7% | 53.7% |

Dominant classes are well segmented (Agropastoral IoU 80.1%, Vegetation
75.4%). Macega (IoU 18.2%) and Infrastructure (44.1%) remain the open
challenges, due to spectral similarity in single-date RGB imagery.

### Limitations

The labels come from photointerpretation and may be uncertain at class
boundaries and in shadowed areas. The benchmark reflects land cover in
2019–2020, not the current state on the ground.

---

## IntegraCAR-LULC-10K landing page

Repository [`IntegraCAR-LULC-10K`](https://github.com/integraCAR/IntegraCAR-LULC-10K):
static page presenting the dataset, in English (`index.html`) and Portuguese
(`index-pt.html`). Bootstrap 5, Bootstrap Icons, Inter font and AOS
animations, all from CDN; no build step.

```
index.html        English page (main)
index-pt.html     Portuguese page
script.js         Smooth scroll, current year, animations
style.css
imagensSobre/     Partner logos and sample image/mask
favicon.ico, Logo-IntegraCAR-ES.png
```

Sections: about, technical specifications, acquisition API, nomenclature,
benchmarks, models, download (10K and 500), paper, limitations and citation.

To preview locally, open `index.html` in a browser or run
`python -m http.server` in the repository folder.

The Portuguese page is behind the English one: it still calls the dataset
"IntegraCAR-LC10K", lists the 23 original classes instead of the 5 grouped
ones, and has no benchmarks or models sections. The private repository
`LANDING-PAGE-INTEGRACAR-ES` holds an earlier version of this same page (see
[web/README.md](../web/README.md#landing-page-integracar-es)).

---

## Citation

```bibtex
@inproceedings{albertino2026integracar,
  title     = {IntegraCAR-LULC-10K: A High-Resolution Optical Satellite Dataset for LULC Segmentation in the Brazilian Rural Environmental Registry},
  author    = {Albertino, Calebe and Lima, Gabriel M. B. and Oliveira, Vinicius R. and Ribeiro, Arthur R. V. and Oliveira, Eliza K. S. and Souza, Eduardo H. P. and Dalvi, Otavio G. and Komati, Karin S. and Andrade, Jefferson O. and Boldt, Francisco A. and Paixao, Thiago M.},
  booktitle = {2026 39th SIBGRAPI Conference on Graphics, Patterns and Images (SIBGRAPI)},
  year      = {2026},
  organization = {IEEE},
  note      = {to appear}
}
```

Affiliation: Instituto Federal do Espírito Santo (IFES) and Instituto de Defesa
Agropecuária e Florestal do Espírito Santo (IDAF). Funding: Inova SEGER 2025,
with FAPES and SEGER.

---

## Open points

Inconsistencies found while writing this page, to be checked against the paper:

- **Resolution.** The landing page says 0.5 m/pixel; the builder defaults
  (`--buffer 1024`, `--width 2048`) produce 1.0 m/pixel.
- **CRS.** The landing page says EPSG:31984; the builder writes GeoTIFFs in
  EPSG:4326 (input coordinates are EPSG:31984).
- **Python version.** The builder README says 3.8+; `numpy>=2.2` in
  `requirements.txt` needs 3.10+.
- **Name.** "IntegraCAR-LULC-10K" (Hugging Face, paper) vs. "IntegraCAR-LC10K"
  (parts of both pages).
